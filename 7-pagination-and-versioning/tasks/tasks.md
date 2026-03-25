# Lesson 7 — In-Class Tasks

## Task 1: PageNumberPagination (15 min)
Add pagination to your Product API:
- 10 products per page by default
- Clients can request up to 50 per page with `?page_size=50`
- Default query param is `?page=`

Seed 50 products in the database. Test:
- `GET /api/products/` → page 1
- `GET /api/products/?page=3` → page 3
- `GET /api/products/?page_size=5&page=2` → custom page size

---

## Task 2: Custom Pagination Response (15 min)
Override `get_paginated_response()` to return this structure:
```json
{
    "meta": {
        "total": 50,
        "total_pages": 5,
        "current_page": 2,
        "page_size": 10
    },
    "links": {
        "next": "...",
        "previous": "..."
    },
    "data": [...]
}
```

---

## Task 3: CursorPagination for a Feed (20 min)
Create a `GET /api/posts/feed/` endpoint that uses CursorPagination.

- Order by `-created_at`
- 15 posts per page
- After calling the endpoint, copy the `next` cursor and use it to get the next page

Why doesn't CursorPagination allow you to jump to page 5 directly?

---

## Task 4: API Versioning (20 min)
Add URL versioning to your API:
- `GET /api/v1/posts/` → returns `{"author": "username"}`
- `GET /api/v2/posts/` → returns `{"author": {"id": 1, "username": "...", "avatar": "..."}}`

Implement two serializers (V1 and V2) and use `request.version` to select the right one.

---

## Task 5: Deprecation Warning
For v1 endpoints, add a custom response header that warns clients:
```
X-API-Warning: This API version is deprecated. Please migrate to v2.
```

Hint: Override the `finalize_response()` method or use middleware.

