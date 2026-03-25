# Lesson 6 — Homework

## 📌 Assignment: Advanced Post Filtering System

### Goal
Add a rich filtering and search system to the Blog Post API.

### FilterSet Requirements
Create a `PostFilter` class with:
- `author_username` — filter by author's username (case-insensitive)
- `title_contains` — filter posts where title contains the given text
- `published_before` and `published_after` — date range filter on `created_at`
- `min_likes` — filter posts with at least N likes
- `has_image` — boolean filter: posts that have an image attached vs. not
- `tags` — filter by one or more tag IDs: `?tags=1,2,3` (hint: use `ModelMultipleChoiceFilter`)

### Search Setup
- Search across: `title`, `content`, `author__username`, `tags__name`
- Allow full-text search on `content` (PostgreSQL `@` prefix)

### Ordering Setup
- Allow ordering by: `created_at`, `title`, `likes_count`, `views_count`
- Default: newest first

### Custom Action: Filter by Read Status
Add `GET /api/posts/unread/` that returns posts the current user hasn't read yet.
(This requires a `PostRead` model: `user + post` M2M relationship)

### Bonus — Save Filters as Collections
Allow users to save a set of filter parameters as a named "collection":
```
POST /api/collections/
Body: { "name": "My Tech Posts", "filters": "category=tech&min_likes=10&ordering=-created_at" }

GET /api/collections/
→ returns user's saved filter collections

GET /api/collections/5/posts/
→ applies the saved filters and returns matching posts
```

### Submission Checklist:
- [ ] All filter fields work correctly
- [ ] Multi-value tag filter works (`?tags=1,2`)
- [ ] Date range filter works
- [ ] Search across multiple fields including related fields
- [ ] Default ordering is applied when none is specified
- [ ] `select_related` / `prefetch_related` is used to prevent N+1 queries

