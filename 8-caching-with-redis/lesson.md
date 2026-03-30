# Lesson 8: Caching with Redis

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain how caching improves API performance
- Set up Redis as Django's cache backend
- Use `cache_page` decorator for view-level caching
- Implement manual cache get/set/delete patterns
- Use `cache_control` and `vary_on_headers`
- Implement cache invalidation strategies

---

## 1. Why Caching?

```
Without cache:
  Request → Django → Database Query (50ms) → Serialize (10ms) → Response (60ms)

With cache:
  Request → Django → Cache HIT (0.5ms) → Response (1ms)
```

Caching is most valuable for:
- Endpoints called frequently (product listings, blog posts)
- Data that doesn't change often (categories, public profiles)
- Expensive queries (aggregations, joins across many tables)

---

## 2. Redis Setup

```bash
# Install Redis (run as service)
# Windows: use WSL2 or Docker
# Linux/Mac: sudo apt install redis-server

pip install django-redis
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',   # prevents collisions between projects
        'TIMEOUT': 300,          # default cache timeout: 5 minutes
    }
}
```

---

## 3. `cache_page` — Simplest View-Level Caching

```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator
from rest_framework import generics

# Function-based view
@api_view(['GET'])
@cache_page(60 * 15)  # cache for 15 minutes
def product_list(request):
    products = Product.objects.all()
    serializer = ProductSerializer(products, many=True)
    return Response(serializer.data)

# Class-based view — using method_decorator
@method_decorator(cache_page(60 * 15), name='dispatch')
class ProductListView(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

⚠️ **Warning:** `cache_page` caches per URL. It does NOT differentiate between users.
Do NOT use `cache_page` on authenticated endpoints that return user-specific data.

---

## 4. Manual Caching — get/set/delete

More control — cache specific data, not the full response:

```python
from django.core.cache import cache
from rest_framework.views import APIView
from rest_framework.response import Response

class ProductListView(APIView):
    def get(self, request):
        cache_key = 'product_list_all'
        cached_data = cache.get(cache_key)

        if cached_data is not None:
            return Response(cached_data)

        # Cache miss — fetch from database
        products = Product.objects.select_related('category').all()
        serializer = ProductSerializer(products, many=True)
        data = serializer.data

        # Store in cache for 10 minutes
        cache.set(cache_key, data, timeout=60 * 10)
        return Response(data)
```

---

## 5. Cache Invalidation

The hardest problem in caching: keeping cache in sync with database.

### Pattern 1: Invalidate on write

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    CACHE_KEY_PREFIX = 'products'

    def _invalidate_product_cache(self):
        cache.delete('product_list_all')
        cache.delete_pattern('products_*')  # requires django-redis

    def perform_create(self, serializer):
        serializer.save()
        self._invalidate_product_cache()

    def perform_update(self, serializer):
        serializer.save()
        self._invalidate_product_cache()

    def perform_destroy(self, instance):
        instance.delete()
        self._invalidate_product_cache()
```

### Pattern 2: Cache versioning

```python
def get_cache_version():
    version = cache.get('products_version', 1)
    return version

def bump_cache_version():
    cache.incr('products_version', 1)

class ProductListView(APIView):
    def get(self, request):
        version = get_cache_version()
        cache_key = f'product_list_v{version}'
        cached = cache.get(cache_key)
        if cached:
            return Response(cached)
        # ... fetch, serialize, cache.set(cache_key, data, 3600)
```

---

## 6. Per-User Caching

```python
class UserDashboardView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        cache_key = f'dashboard_user_{request.user.id}'
        cached = cache.get(cache_key)
        if cached:
            return Response(cached)

        # Expensive aggregation
        data = {
            'post_count': Post.objects.filter(author=request.user).count(),
            'likes_received': Like.objects.filter(post__author=request.user).count(),
            'follower_count': Follow.objects.filter(following=request.user).count(),
        }
        cache.set(cache_key, data, timeout=60 * 5)  # 5 minutes
        return Response(data)
```

---

## 7. Caching with `vary_on_headers` and `vary_on_cookie`

```python
from django.views.decorators.vary import vary_on_headers

@vary_on_headers('Accept-Language')
@cache_page(60 * 60)
@api_view(['GET'])
def product_list(request):
    # Cached separately per language
    lang = request.META.get('HTTP_ACCEPT_LANGUAGE', 'en')
    ...
```

---

## 8. Cache Utility — Reusable Pattern

```python
# utils/cache.py
from django.core.cache import cache
import hashlib
import json

def make_cache_key(*args, **kwargs):
    """Create a deterministic cache key from arguments."""
    key_data = json.dumps({'args': args, 'kwargs': kwargs}, sort_keys=True)
    return hashlib.md5(key_data.encode()).hexdigest()

def cached_queryset(cache_key, queryset_func, timeout=300):
    """Generic cache-or-fetch pattern."""
    data = cache.get(cache_key)
    if data is None:
        data = queryset_func()
        cache.set(cache_key, data, timeout)
    return data
```

---

## 9. Redis Directly — Beyond Django Cache

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Counters (atomic increment)
r.incr('api:hits:today')

# Leaderboard (sorted sets)
r.zadd('post:likes', {'post_5': 142, 'post_3': 98})
top_posts = r.zrevrange('post:likes', 0, 9, withscores=True)

# Pub/Sub (for real-time features)
# Expiring keys (session-like behavior)
r.setex('session:abc123', 3600, 'user_id:42')
```

---

## 📝 Key Takeaways
1. Redis cache dramatically reduces database load for repeated, unchanged data
2. `cache_page` is quick but blunt — don't use it for user-specific data
3. Manual `cache.get/set/delete` gives fine-grained control
4. Cache invalidation on writes keeps data consistent
5. Use namespaced cache keys (`user_123_dashboard`) to avoid collisions
6. Set appropriate TTLs — too long = stale data, too short = no benefit

