# Lesson 1: Introduction to Django REST Framework

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain what REST is and why APIs are needed
- Install and configure DRF in a Django project
- Build their first working API endpoint
- Understand the DRF request/response cycle

---

## 1. What is REST? (The Why)

**REST** (Representational State Transfer) is an architectural style for building web APIs.

### REST Principles:
| Principle | Meaning |
|-----------|---------|
| **Stateless** | Each request is independent; no server-side session |
| **Client-Server** | Frontend and backend are separate |
| **Uniform Interface** | Resources identified by URLs |
| **Cacheable** | Responses can be cached |

### HTTP Methods → CRUD Mapping:
```
GET    /posts/       → List all posts
POST   /posts/       → Create a new post
GET    /posts/1/     → Retrieve post with id=1
PUT    /posts/1/     → Update post with id=1 (full update)
PATCH  /posts/1/     → Partial update post with id=1
DELETE /posts/1/     → Delete post with id=1
```

### HTTP Status Codes:
```
200 OK           → Success
201 Created      → Resource created
204 No Content   → Success, nothing to return
400 Bad Request  → Validation error
401 Unauthorized → Not authenticated
403 Forbidden    → Authenticated but no permission
404 Not Found    → Resource doesn't exist
500 Server Error → Something went wrong on the server
```

---

## 2. Why DRF? (Django REST Framework)

Django by itself can return JSON responses, but DRF adds:
- **Serializers** — Convert complex Python objects (QuerySets) to JSON
- **Class-based views** — Reusable, clean API views
- **Authentication** — Token, JWT, Session out of the box
- **Permissions** — Fine-grained access control
- **Browsable API** — Visual interface for testing in the browser
- **Throttling, Filtering, Pagination** — Built-in

---

## 3. Setup

### Install:
```bash
pip install djangorestframework
pip install django
pip install psycopg2-binary   # for PostgreSQL
```

### `settings.py`:
```python
INSTALLED_APPS = [
    # ... django apps
    'rest_framework',
    'myapp',
]

REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # remove in production
    ],
}
```

### Project structure:
```
myproject/
    myproject/
        settings.py
        urls.py
    myapp/
        models.py
        serializers.py
        views.py
        urls.py
    manage.py
    requirements.txt
```

---

## 4. Your First API — Function-Based View

### `myapp/models.py`:
```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

### `myapp/serializers.py`:
```python
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'created_at']
```

### `myapp/views.py` — Function-Based View (FBV):
```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Post
from .serializers import PostSerializer

@api_view(['GET', 'POST'])
def post_list(request):
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def post_detail(request, pk):
    try:
        post = Post.objects.get(pk=pk)
    except Post.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = PostSerializer(post)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### `myapp/urls.py`:
```python
from django.urls import path
from . import views

urlpatterns = [
    path('posts/', views.post_list, name='post-list'),
    path('posts/<int:pk>/', views.post_detail, name='post-detail'),
]
```

### `myproject/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),
]
```

---

## 5. The DRF Request/Response Cycle

```
HTTP Request
     ↓
Django URL Router
     ↓
DRF View (authentication, permission check)
     ↓
Serializer (deserialize input / validate)
     ↓
Database (ORM Query)
     ↓
Serializer (serialize output)
     ↓
DRF Response (JSON)
     ↓
HTTP Response
```

---

## 6. Testing with curl / Browsable API

```bash
# List all posts
curl http://127.0.0.1:8000/api/posts/

# Create a post
curl -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Hello World", "content": "My first post"}'

# Get single post
curl http://127.0.0.1:8000/api/posts/1/

# Delete a post
curl -X DELETE http://127.0.0.1:8000/api/posts/1/
```

Or open `http://127.0.0.1:8000/api/posts/` in your browser to use the **Browsable API**.

---

## 7. Swagger UI — Auto-Generated API Documentation

From Lesson 1 onwards, set up **Swagger** so every endpoint you build is instantly documented and testable in a browser. This is one of the biggest productivity wins in DRF development.

### Install:
```bash
pip install drf-spectacular
```

### `settings.py`:
```python
INSTALLED_APPS = [
    # ...existing apps...
    'rest_framework',
    'drf_spectacular',   # ← add this
]

REST_FRAMEWORK = {
    # Tell DRF to use drf-spectacular for schema generation
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'My Blog API',
    'DESCRIPTION': 'A REST API built with Django REST Framework',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,  # hide the raw schema endpoint from docs UI
}
```

### `myproject/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),

    # Schema & Docs endpoints
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),           # raw OpenAPI JSON
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),  # Swagger UI
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),        # ReDoc UI
]
```

### What you get:
| URL | What it shows |
|-----|--------------|
| `http://localhost:8000/api/docs/` | **Swagger UI** — interactive, click "Try it out" to test any endpoint |
| `http://localhost:8000/api/redoc/` | **ReDoc** — cleaner read-only reference docs |
| `http://localhost:8000/api/schema/` | Raw OpenAPI 3.0 JSON — importable into Postman, Insomnia, etc. |

### Result in the browser:
```
Swagger UI shows:
  GET  /api/posts/         → "List all posts"
  POST /api/posts/         → "Create a post"
  GET  /api/posts/{id}/    → "Retrieve a post"
  PUT  /api/posts/{id}/    → "Update a post"
  DELETE /api/posts/{id}/  → "Delete a post"

Each endpoint has:
  - Parameters documented
  - Request body schema
  - Response schema
  - "Try it out" button to send real requests
```

> 💡 **No extra code needed.** `drf-spectacular` reads your serializers and views automatically and generates the full schema. You just install it and add the URLs.

---

## 📝 Key Takeaways
1. REST is a convention for structuring APIs — DRF makes it easy to implement
2. Serializers are the bridge between Python objects and JSON
3. `@api_view` decorator turns a regular function into a DRF view
4. Always return proper HTTP status codes — they communicate meaning
5. DRF's Browsable API is great during development; Swagger UI is better to share with frontend teams
6. Install `drf-spectacular` from Lesson 1 — auto-generates API docs with zero extra effort

