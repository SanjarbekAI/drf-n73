# Lesson 1 — In-Class Tasks

## Task 1: Setup & First Endpoint (20 min)
Create a new Django project with DRF and build a working `GET /api/books/` endpoint.

**Requirements:**
- Book model: `title`, `author`, `published_year`, `isbn`
- Return all books as JSON
- Test it in the Browsable API

**Starter:**
```python
# models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_year = models.IntegerField()
    isbn = models.CharField(max_length=13, unique=True)
```

---

## Task 2: Full CRUD (25 min)
Extend your Book API to support all CRUD operations:
- `GET /api/books/` → list
- `POST /api/books/` → create
- `GET /api/books/<id>/` → retrieve
- `PUT /api/books/<id>/` → update
- `DELETE /api/books/<id>/` → delete

**Test each endpoint using curl or the Browsable API.**

---

## Task 3: Status Codes (10 min)
Review your views and answer:
1. What status code does your POST return on success? Is it correct?
2. What happens if you request `GET /api/books/9999/`? Fix it if needed.
3. What status code should a DELETE return? Fix it.

---

## Task 4: Discussion Questions
- What would happen if we returned `200` for every response regardless of outcome?
- Why is it important to separate frontend and backend?
- What are the disadvantages of function-based views? (Hint: think about code repetition)

---

## Task 5: Set Up Swagger (15 min)
Add `drf-spectacular` to your project so all your endpoints are visible in Swagger UI.

**Steps:**
1. Install: `pip install drf-spectacular`
2. Add to `INSTALLED_APPS` and set `DEFAULT_SCHEMA_CLASS` in `settings.py`
3. Add the 3 URL patterns to `myproject/urls.py`
4. Run the server and open `http://localhost:8000/api/docs/`

**Verify:**
- You can see your Book endpoints listed in Swagger UI
- Click **"Try it out"** on `GET /api/books/` → click **"Execute"** → you get a real response
- Open `http://localhost:8000/api/redoc/` — what's different about ReDoc vs Swagger UI?
- Open `http://localhost:8000/api/schema/` — this is the raw OpenAPI JSON (try pasting it into https://editor.swagger.io)

> 💡 Keep Swagger set up for all remaining lessons — every new endpoint you build will auto-appear here.

