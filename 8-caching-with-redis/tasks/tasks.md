# Lesson 8 — In-Class Tasks

## Task 1: Redis Setup & Basic Cache (15 min)
1. Start Redis (`redis-server` or via Docker: `docker run -p 6379:6379 redis`)
2. Install and configure `django-redis` as the cache backend
3. Test the connection:
```python
from django.core.cache import cache
cache.set('test', 'hello redis', 60)
print(cache.get('test'))  # should print 'hello redis'
```

---

## Task 2: `cache_page` on a Public Endpoint (15 min)
Apply `cache_page` to `GET /api/products/` with a 5-minute TTL.

Verify caching works:
1. Add a print statement inside the view: `print("DB HIT")`
2. Make 5 requests — does "DB HIT" print each time or only once?
3. What happens if you change a product in the database while it's cached?

---

## Task 3: Manual Caching with Invalidation (25 min)
Implement manual caching for `GET /api/products/`:
- Cache key: `product_list`
- TTL: 10 minutes
- On `POST`, `PUT`, `DELETE` → delete the cache key

Walk through this sequence and confirm the cache is properly invalidated:
1. `GET /api/products/` → cache miss, returns 10 products
2. `GET /api/products/` → cache hit
3. `POST /api/products/` → creates product, invalidates cache
4. `GET /api/products/` → cache miss again, now returns 11 products

---

## Task 4: Per-User Caching (20 min)
Build a `GET /api/users/me/stats/` endpoint that returns:
```json
{
    "post_count": 15,
    "total_likes": 230,
    "total_views": 1200
}
```

Cache the result per user (`user_{id}_stats`) for 5 minutes.
Invalidate the cache whenever the user creates a new post or receives a new like.

---

## Task 5: Redis CLI Inspection (5 min)
Use the Redis CLI to inspect your cache keys:
```bash
redis-cli
> KEYS *
> TTL product_list
> GET myapp:product_list
> DEL myapp:product_list
```

What's the key format? Why is there a prefix?

