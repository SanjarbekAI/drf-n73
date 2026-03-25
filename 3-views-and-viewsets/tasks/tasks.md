# Lesson 3 — In-Class Tasks

## Task 1: Rewrite FBV as APIView (15 min)
Take the Book API from Lesson 1 (function-based) and rewrite it as `APIView` classes.
- `BookListView(APIView)` — handles GET and POST
- `BookDetailView(APIView)` — handles GET, PUT, DELETE

**Verify both implementations produce identical results.**

---

## Task 2: Generic Views (15 min)
Now rewrite the same API using generic views:
- Use `ListCreateAPIView` for the list endpoint
- Use `RetrieveUpdateDestroyAPIView` for the detail endpoint

How many lines of code did you save compared to `APIView`?

---

## Task 3: ModelViewSet (15 min)
Rewrite again using `ModelViewSet`. Your entire views.py should be under 15 lines.

Then add:
- Override `get_queryset()` to only return books published after year 2000
- Override `perform_create()` to print `"New book created: {title}"` to the console

---

## Task 4: Custom Actions (20 min)
Add these custom endpoints to your `BookViewSet`:
1. `POST /api/books/<id>/mark-read/` — marks a book as read (adds a `is_read` BooleanField to the model)
2. `GET /api/books/classics/` — returns only books published before 1990
3. `GET /api/books/<id>/similar/` — returns books by the same author (excluding the current book)

---

## Task 5: Soft Delete (Bonus, 15 min)
Instead of actually deleting a book, implement "soft delete":
- Add `is_deleted = BooleanField(default=False)` to the model
- Override `destroy()` in the ViewSet to set `is_deleted=True` instead of deleting
- Override `get_queryset()` to exclude deleted books from all list/detail results

