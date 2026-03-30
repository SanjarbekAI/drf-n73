# Lesson 9: Celery & Asynchronous Tasks

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain what Celery is and when to use it
- Set up Celery with Redis as a broker
- Write and call basic Celery tasks
- Use task chaining, grouping, and retries
- Schedule periodic tasks with Celery Beat
- Monitor tasks with Flower

---

## 1. Why Async Tasks?

Some operations are too slow to do during an HTTP request:
- Sending emails → 1-3 seconds
- Generating PDF reports → 5-10 seconds
- Processing uploaded images (resize, compress) → 2-5 seconds
- Sending push notifications to 10,000 users → minutes
- Running machine learning inference → variable

```
BAD (synchronous):
  POST /api/orders/ → create order → send confirmation email (3s) → Response (3s total)

GOOD (async):
  POST /api/orders/ → create order → queue email task → Response (50ms)
  ...background... → Celery worker picks up task → sends email
```

---

## 2. Celery Architecture

```
Django App          Message Broker        Celery Worker(s)
    │                  (Redis)                  │
    │── task.delay() ──► [Queue] ──────────────► execute task()
    │                                            │
    │◄─────────────────────── result (stored in Redis) ─────┘
```

- **Producer** — Django sends tasks to the queue
- **Broker** — Redis holds the task queue
- **Worker** — Celery process that picks and executes tasks
- **Backend** — Redis stores task results

---

## 3. Installation & Setup

```bash
pip install celery redis django-celery-beat django-celery-results
```

### `myproject/celery.py`:
```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

### `myproject/__init__.py`:
```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

### `settings.py`:
```python
INSTALLED_APPS = [
    ...
    'django_celery_beat',
    'django_celery_results',
]

# Celery configuration
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'django-db'  # store results in PostgreSQL
CELERY_CACHE_BACKEND = 'default'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# django-celery-results
CELERY_RESULT_BACKEND = 'django-db'
```

```bash
python manage.py migrate  # create celery results tables
```

---

## 4. Writing Tasks

```python
# myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail
import time

@shared_task
def send_welcome_email(user_id):
    """Send a welcome email to a newly registered user."""
    from django.contrib.auth.models import User
    user = User.objects.get(id=user_id)
    send_mail(
        subject='Welcome!',
        message=f'Hi {user.username}, welcome to our platform!',
        from_email='noreply@example.com',
        recipient_list=[user.email],
    )
    return f'Email sent to {user.email}'

@shared_task
def process_image(image_path):
    """Resize and compress an uploaded image."""
    from PIL import Image
    img = Image.open(image_path)
    img.thumbnail((800, 800))
    img.save(image_path, optimize=True, quality=85)
    return f'Processed: {image_path}'

@shared_task
def generate_report(report_id):
    """Long-running report generation."""
    time.sleep(10)  # simulate heavy work
    return {'report_id': report_id, 'status': 'complete'}
```

---

## 5. Calling Tasks

```python
# views.py
from .tasks import send_welcome_email, generate_report
from rest_framework.response import Response

class RegisterView(APIView):
    def post(self, request):
        serializer = RegisterSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            # Queue the email — don't wait for it
            send_welcome_email.delay(user.id)
            return Response({'message': 'Registered!'}, status=201)

class ReportView(APIView):
    def post(self, request):
        # Start the task and return the task ID immediately
        task = generate_report.delay(request.data['report_id'])
        return Response({'task_id': task.id, 'status': 'processing'})

    def get(self, request, task_id):
        # Check task status
        from celery.result import AsyncResult
        result = AsyncResult(task_id)
        return Response({
            'task_id': task_id,
            'status': result.status,         # PENDING, STARTED, SUCCESS, FAILURE
            'result': result.result,          # None until complete
        })
```

---

## 6. Task Options: Retries

```python
@shared_task(bind=True, max_retries=3)
def send_notification(self, user_id, message):
    try:
        # ... send push notification
        pass
    except SomeTransientError as exc:
        # Retry after 60 seconds, up to 3 times
        raise self.retry(exc=exc, countdown=60)
```

---

## 7. Task Chains and Groups

```python
from celery import chain, group, chord

# Chain: run tasks sequentially (output of one feeds into next)
result = chain(
    validate_order.s(order_id),
    charge_payment.s(),
    send_confirmation_email.s(),
)()

# Group: run tasks in parallel
result = group(
    send_email.s(user_id) for user_id in user_ids
)()

# Chord: parallel tasks + callback when all done
result = chord(
    group(process_image.s(path) for path in image_paths)
)(generate_album_thumbnail.s())
```

---

## 8. Celery Beat — Periodic Tasks

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'clear-expired-tokens-daily': {
        'task': 'myapp.tasks.clear_expired_tokens',
        'schedule': crontab(hour=3, minute=0),  # every day at 3:00 AM
    },
    'update-trending-posts-every-15-min': {
        'task': 'myapp.tasks.update_trending_posts',
        'schedule': crontab(minute='*/15'),  # every 15 minutes
    },
    'send-daily-digest': {
        'task': 'myapp.tasks.send_daily_digest',
        'schedule': crontab(hour=8, minute=0, day_of_week='mon-fri'),
    },
}
```

```python
# tasks.py
@shared_task
def clear_expired_tokens():
    from rest_framework.authtoken.models import Token
    from django.utils import timezone
    from datetime import timedelta
    cutoff = timezone.now() - timedelta(days=30)
    deleted_count, _ = Token.objects.filter(created__lt=cutoff).delete()
    return f'Deleted {deleted_count} expired tokens'

@shared_task
def update_trending_posts():
    import redis
    r = redis.Redis(host='localhost', port=6379, db=0)
    # recalculate trending from Redis counters and cache the result
    ...
```

---

## 9. Running Celery

```bash
# Start worker (in a separate terminal)
celery -A myproject worker --loglevel=info

# Start beat scheduler (in another terminal)
celery -A myproject beat --loglevel=info

# Monitor with Flower (web UI)
pip install flower
celery -A myproject flower --port=5555
# Open http://localhost:5555
```

---

## 10. Task Progress Updates with WebSocket / Polling

```python
@shared_task(bind=True)
def long_import_task(self, file_path):
    rows = read_csv(file_path)
    total = len(rows)
    for i, row in enumerate(rows):
        process_row(row)
        # Update progress (stored in Redis)
        self.update_state(
            state='PROGRESS',
            meta={'current': i + 1, 'total': total, 'percent': int((i + 1) / total * 100)}
        )
    return {'status': 'complete', 'total': total}
```

```python
# Poll progress from the view
def get(self, request, task_id):
    result = AsyncResult(task_id)
    if result.state == 'PROGRESS':
        return Response({'status': 'progress', 'progress': result.info})
    elif result.state == 'SUCCESS':
        return Response({'status': 'done', 'result': result.result})
```

---

## 📝 Key Takeaways
1. Move anything that takes > 200ms out of the request cycle into a Celery task
2. Redis is both the broker (task queue) and can be the result backend
3. `task.delay()` is the most common way to call tasks asynchronously
4. Always return the task ID so clients can poll for status
5. Use Celery Beat for scheduled/periodic tasks
6. Monitor tasks with Flower in development

