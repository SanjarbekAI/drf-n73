# Lesson 7 — Homework

## 📌 Assignment: Paginated Social Feed + Versioned User API

### Part 1: Social Feed with CursorPagination
Build a `GET /api/v1/feed/` endpoint that:
- Returns posts from users the current user follows
- Uses CursorPagination (ordered by `-created_at`)
- Page size: 20
- Each post includes: id, title, excerpt (first 150 chars of content), author username, like count, comment count, created_at

### Part 2: Versioned User Profile Endpoint

#### V1 Response (`/api/v1/users/<id>/`):
```json
{
    "id": 1,
    "username": "john",
    "bio": "Developer"
}
```

#### V2 Response (`/api/v2/users/<id>/`):
```json
{
    "id": 1,
    "username": "john",
    "bio": "Developer",
    "stats": {
        "posts_count": 42,
        "followers_count": 100,
        "following_count": 50
    },
    "recent_posts": [
        {"id": 5, "title": "My Latest Post", "created_at": "..."}
    ]
}
```

### Part 3: Pagination on Admin Exports
Create a special `GET /api/admin/export/posts/` endpoint that:
- Requires admin authentication
- Returns up to 1000 posts per page (for data export purposes)
- Uses `LimitOffsetPagination`
- Allows `?limit=1000&offset=0`

### Bonus — Pagination Metadata Endpoint:
```
GET /api/posts/?page=2&page_size=10
```
Response should include in the `meta`:
- `total` — total record count
- `filtered_total` — count after applying any search/filter
- `total_pages` — based on page_size
- `page_size_options` — `[10, 25, 50, 100]` — available page sizes

### Submission Checklist:
- [ ] Feed uses CursorPagination and works correctly
- [ ] V1 and V2 responses are different as specified
- [ ] Admin export endpoint uses LimitOffset
- [ ] All list endpoints are paginated (nothing returns unbounded results)
- [ ] Versioning is done via URL path (`/api/v1/` and `/api/v2/`)

