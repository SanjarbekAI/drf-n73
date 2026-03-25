# Lesson 9 — Homework

## 📌 Assignment: Full Caching Layer for the Blog API

### Part 1: Cache Public Endpoints
Apply caching to these public, read-heavy endpoints:

| Endpoint | Cache Key | TTL | Invalidate When |
|----------|-----------|-----|-----------------|
| `GET /api/posts/` | `post_list_page_{page}` | 5 min | Any post is created/updated/deleted |
| `GET /api/posts/<id>/` | `post_detail_{id}` | 10 min | That specific post is updated/deleted |
| `GET /api/categories/` | `category_list` | 1 hour | Any category changes |
| `GET /api/posts/trending/` | `trending_posts` | 15 min | On a timer (do NOT invalidate on every like) |

### Part 2: Per-User Dashboard Cache
Cache `GET /api/users/me/dashboard/` per user:
```json
{
    "drafts_count": 3,
    "published_count": 12,
    "total_views": 5420,
    "total_likes": 233,
    "recent_comments": [...]
}
```
- TTL: 3 minutes
- Invalidate on: new post, new like, new comment on user's post

### Part 3: Cache Stampede Prevention
The "thundering herd" problem: when a cache key expires, many requests all hit the DB simultaneously.

Implement a simple lock-based prevention:
```python
import time
from django.core.cache import cache

def get_or_set_with_lock(cache_key, fetch_func, timeout=300):
    """
    Fetch data with a distributed lock to prevent cache stampede.
    """
    data = cache.get(cache_key)
    if data is not None:
        return data

    lock_key = f'lock:{cache_key}'
    lock_acquired = cache.add(lock_key, '1', timeout=30)  # add() only sets if key doesn't exist

    if lock_acquired:
        try:
            data = fetch_func()
            cache.set(cache_key, data, timeout)
        finally:
            cache.delete(lock_key)
    else:
        # Another process is fetching — wait briefly and retry
        time.sleep(0.1)
        data = cache.get(cache_key) or fetch_func()

    return data
```

Apply this utility to at least one endpoint.

### Part 4: Redis Sorted Set — Post View Counter
Use Redis directly (not Django cache) to track post view counts:
- On each `GET /api/posts/<id>/`, increment a Redis counter: `ZINCRBY post:views 1 post_{id}`
- `GET /api/posts/trending/` → use `ZREVRANGE post:views 0 9 WITHSCORES` to get top 10 most-viewed posts

### Bonus — Cache Warming
Write a Django management command `python manage.py warm_cache` that:
- Pre-populates the cache for all popular endpoints on server startup
- Fetches the top 100 products and all categories and caches them

```python
# management/commands/warm_cache.py
from django.core.management.base import BaseCommand
from django.core.cache import cache

class Command(BaseCommand):
    help = 'Pre-warm application caches'

    def handle(self, *args, **options):
        self.stdout.write('Warming cache...')
        # populate cache here
        self.stdout.write(self.style.SUCCESS('Cache warmed successfully!'))
```

### Submission Checklist:
- [ ] All four public endpoints are cached with correct TTLs
- [ ] Per-user dashboard is cached and properly invalidated
- [ ] Cache invalidation fires on writes (not just TTL expiry)
- [ ] Redis sorted set view counter is working
- [ ] No user-specific data is leaked in shared cache keys

