# Lesson 6 — In-Class Tasks

## Task 1: Basic FilterSet (15 min)
Add filtering to your Product API:
- Filter by `category` (exact match)
- Filter by `is_active` (boolean)
- Filter by `min_price` and `max_price` (range)

Test: `GET /api/products/?category=1&min_price=50&max_price=200&is_active=true`

---

## Task 2: Search Filter (10 min)
Add search to the Product API:
- Search across `name`, `description`, and `category__name`
- Make the name field a starts-with search (`^name`)
- Make the description field a full-text PostgreSQL search (`@description`)

Test: `GET /api/products/?search=phone`

---

## Task 3: Ordering Filter (10 min)
Add ordering to the Product API:
- Allow ordering by: `price`, `name`, `stock`, `created_at`
- Set default ordering to `-created_at` (newest first)

Test both:
- `GET /api/products/?ordering=price`
- `GET /api/products/?ordering=-price,name`

---

## Task 4: Combined Filtering (15 min)
Make a single Product endpoint that supports all three simultaneously:
```
GET /api/products/?search=samsung&min_price=100&max_price=1000&category=electronics&ordering=-price
```

Make sure all filters work together without conflict.

---

## Task 5: Custom Filter Backend (20 min)
Build a `RecentlyAddedFilterBackend` that:
- Checks if the query param `?recent=true` is in the request
- If so, only returns items created in the last 7 days
- If not, returns everything (no filtering)

```python
class RecentlyAddedFilterBackend(filters.BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        if request.query_params.get('recent') == 'true':
            from django.utils import timezone
            from datetime import timedelta
            seven_days_ago = timezone.now() - timedelta(days=7)
            return queryset.filter(created_at__gte=seven_days_ago)
        return queryset
```

Apply it to your Product ViewSet and test it.

---

## Task 6: Performance Check
Add `django-debug-toolbar` or check the query count:
- Before adding `select_related('category')` to the queryset
- After adding `select_related('category')`

How many queries does filtering on `category__name` cause without `select_related`?

