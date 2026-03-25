# Lesson 12 — Homework

## 📌 Assignment: Production-Ready Blog API — Final Polish

This is the capstone homework for the DRF section. You will optimize and harden the Blog API built across all previous lessons.

### Part 1: Signal-Driven Architecture
Move ALL side effects out of views and into signals:

| Event (Signal) | Side Effect |
|----------------|-------------|
| User created | Create profile, create token, queue welcome email |
| Post published | Notify followers (async task), invalidate feed cache |
| Comment created | Notify post author (async task), update comment count cache |
| Like added/removed | Update like count cache in Redis |
| User followed | Send follow notification (async task) |

### Part 2: Middleware Stack
Implement all three middleware:
1. `RequestLoggingMiddleware` — log every request with method, path, status, duration, user
2. `SecurityHeadersMiddleware` — add all standard security headers
3. `CorrelationIDMiddleware` — generate and attach `X-Correlation-ID` to every request/response

### Part 3: Query Optimization Audit
Go through every ViewSet in your project and ensure:

**All list endpoints must:**
- Use `select_related()` for all ForeignKey/OneToOne traversals in the serializer
- Use `prefetch_related()` for all ManyToMany/reverse FK traversals
- Use `annotate()` for any count/aggregate fields instead of `SerializerMethodField` with `queryset.count()`
- Use `only()` or `defer()` where large fields aren't needed in list views

**Benchmark:**
Use Django shell to count queries before and after:
```python
from django.db import connection, reset_queries
from django.conf import settings

settings.DEBUG = True
reset_queries()
# run your viewset's get_queryset() here
print(len(connection.queries))
```

### Part 4: Caching + Signals Integration
When a signal fires that changes data, also invalidate the related cache:
- Post published → delete `post_list_*` cache keys
- Comment added → delete `post_detail_{id}` cache key
- Like added → update the like count in Redis sorted set

### Part 5: Full Integration Test
Write a test that simulates a complete user journey:
```python
@pytest.mark.django_db
def test_full_blog_flow(api_client):
    # 1. Register
    # 2. Login, get tokens
    # 3. Create a post
    # 4. Publish the post (check signal fired)
    # 5. Another user comments
    # 6. Original user likes the comment
    # 7. Verify all counts are correct
    # 8. Delete the post
    # 9. Verify 404 on the deleted post
```

### Submission Checklist:
- [ ] All side effects are in signals, not views
- [ ] Three middleware classes are implemented and registered
- [ ] Zero N+1 queries in any list endpoint (verified with debug toolbar)
- [ ] All aggregations use `annotate()` not Python loops
- [ ] Cache invalidation fires from signals
- [ ] Full integration test passes
- [ ] Query count before/after optimization is documented in a comment

### 🏆 Bonus — Performance Comparison Report
Create `PERFORMANCE.md` with:
- Number of queries per endpoint before and after optimization
- Response time measurements (use `ab` or `wrk` for load testing)
- Redis cache hit rate after implementing caching

