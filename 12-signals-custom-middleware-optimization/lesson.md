# Lesson 12: Signals, Custom Middleware & Query Optimization

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Use Django signals to decouple side effects from views
- Write custom middleware for cross-cutting concerns
- Identify and fix N+1 query problems with `select_related` and `prefetch_related`
- Use `only()`, `defer()`, and `values()` for field-level optimization
- Use `annotate()` to add computed fields at the database level
- Profile queries with `django-debug-toolbar`

---

## 1. Django Signals — Decoupled Side Effects

Signals allow code to react to events in other parts of the application, without tight coupling.

```
User.objects.create_user(...)
        ↓  post_save signal fires automatically
create_token()
send_welcome_email()
create_user_profile()
```

### Built-in signals:
| Signal | When |
|--------|------|
| `pre_save` | Before `model.save()` |
| `post_save` | After `model.save()` |
| `pre_delete` | Before `model.delete()` |
| `post_delete` | After `model.delete()` |
| `m2m_changed` | M2M relationship changed |
| `request_started` | HTTP request received |
| `request_finished` | HTTP response sent |

---

## 2. Using Signals

```python
# myapp/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
from .models import UserProfile
from .tasks import send_welcome_email

@receiver(post_save, sender=User)
def on_user_created(sender, instance, created, **kwargs):
    """Fires every time a User is saved. created=True only on first save."""
    if created:
        # Create auth token
        Token.objects.create(user=instance)
        # Create profile
        UserProfile.objects.create(user=instance)
        # Queue welcome email (async)
        send_welcome_email.delay(instance.id)

@receiver(post_save, sender='myapp.Post')
def on_post_published(sender, instance, created, **kwargs):
    """When a post is published, notify followers."""
    if not created and instance.published:
        from .tasks import notify_followers
        notify_followers.delay(instance.id)
```

```python
# myapp/apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    name = 'myapp'

    def ready(self):
        import myapp.signals  # IMPORTANT: connect signals when app loads
```

```python
# myapp/__init__.py
default_app_config = 'myapp.apps.MyAppConfig'
```

---

## 3. Custom Signals

```python
# signals.py
from django.dispatch import Signal

post_published = Signal()  # custom signal
post_liked = Signal()

# In view or model
from .signals import post_published
post_published.send(sender=Post, instance=post, publisher=request.user)

# In another part of the app
@receiver(post_published)
def on_post_published(sender, instance, publisher, **kwargs):
    # send emails, update cache, etc.
    ...
```

---

## 4. Custom Middleware

Middleware runs on every request/response. Use for:
- Logging all requests
- Adding custom response headers
- Blocking certain IPs
- Attaching correlation IDs for tracing

```python
# middleware.py
import time
import uuid
import logging

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    """Log all API requests with timing."""
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start_time = time.time()
        request_id = str(uuid.uuid4())[:8]

        # Attach to request for use in views
        request.request_id = request_id

        response = self.get_response(request)

        duration = (time.time() - start_time) * 1000  # milliseconds
        logger.info(
            f'[{request_id}] {request.method} {request.path} '
            f'→ {response.status_code} ({duration:.1f}ms)'
        )
        return response


class SecurityHeadersMiddleware:
    """Add security headers to all responses."""
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        response['X-Content-Type-Options'] = 'nosniff'
        response['X-Frame-Options'] = 'DENY'
        response['X-XSS-Protection'] = '1; mode=block'
        response['Referrer-Policy'] = 'strict-origin-when-cross-origin'
        return response


class MaintenanceModeMiddleware:
    """Return 503 if maintenance mode is enabled."""
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.conf import settings
        if getattr(settings, 'MAINTENANCE_MODE', False):
            from django.http import JsonResponse
            return JsonResponse(
                {'error': 'Service temporarily unavailable. Try again later.'},
                status=503
            )
        return self.get_response(request)
```

```python
# settings.py
MIDDLEWARE = [
    'myapp.middleware.RequestLoggingMiddleware',
    'myapp.middleware.SecurityHeadersMiddleware',
    'django.middleware.security.SecurityMiddleware',
    ...
]
```

---

## 5. The N+1 Query Problem

The most common performance killer in Django APIs.

```python
# BAD — N+1 queries
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()  # 1 query

# When serializer accesses post.author for each post:
# + 1 query per post = N+1 total queries!
```

### Fix with `select_related` (ForeignKey, OneToOne):
```python
# GOOD — 2 queries total (posts + authors JOIN)
queryset = Post.objects.select_related('author', 'category').all()
```

### Fix with `prefetch_related` (ManyToMany, reverse FK):
```python
# GOOD — 3 queries (posts + tags + comments)
queryset = Post.objects.prefetch_related('tags', 'comments').all()

# Combined
queryset = Post.objects.select_related('author').prefetch_related('tags', 'comments')
```

---

## 6. `only()` and `defer()` — Field-Level Optimization

```python
# Fetch ONLY what you need (list view — no content body needed)
queryset = Post.objects.only('id', 'title', 'author_id', 'created_at')

# Fetch everything EXCEPT large fields
queryset = Post.objects.defer('content', 'raw_html')
```

---

## 7. `values()` and `values_list()` — No Model Instances

```python
# Returns dicts instead of model instances — faster for read-only data
posts = Post.objects.values('id', 'title', 'author__username')
# [{'id': 1, 'title': 'Hello', 'author__username': 'john'}, ...]

# Returns tuples
ids = Post.objects.values_list('id', flat=True)
# [1, 2, 3, ...]
```

---

## 8. `annotate()` — Database-Level Computed Fields

Move aggregation to the database instead of Python:

```python
from django.db.models import Count, Avg, Sum, Q

# Add counts to the queryset — ONE query instead of N
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    like_count=Count('likes'),
    average_rating=Avg('reviews__rating'),
)

# In the serializer — just reference the annotated field
class PostSerializer(serializers.ModelSerializer):
    comment_count = serializers.IntegerField(read_only=True)
    like_count = serializers.IntegerField(read_only=True)
    ...
```

---

## 9. `django-debug-toolbar` — Profile Your Queries

```bash
pip install django-debug-toolbar
```

```python
# settings.py
INSTALLED_APPS = ['debug_toolbar', ...]
MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware', ...]
INTERNAL_IPS = ['127.0.0.1']

# urls.py
import debug_toolbar
urlpatterns = [path('__debug__/', include(debug_toolbar.urls))] + urlpatterns
```

This shows a panel with all SQL queries, their count, and execution time. Use it to find N+1 problems instantly.

---

## 📝 Key Takeaways
1. Signals decouple side effects (emails, cache invalidation) from core logic
2. Always call `import myapp.signals` in `AppConfig.ready()` or signals won't fire
3. Custom middleware is for cross-cutting concerns (logging, headers, auth)
4. **N+1 is the #1 DRF performance mistake** — always `select_related`/`prefetch_related`
5. `annotate()` pushes aggregation to the database — far faster than Python loops
6. Use `django-debug-toolbar` to catch query problems before they reach production

