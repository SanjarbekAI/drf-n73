# Lesson 7: Pagination & Versioning

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain why pagination is essential for large datasets
- Implement PageNumberPagination, LimitOffsetPagination, and CursorPagination
- Create custom pagination classes
- Use URL and Header-based API versioning
- Apply versioning strategies in a real project

---

## 1. Why Pagination?

Without pagination:
- `GET /api/products/` → returns ALL 50,000 products
- Huge response size, slow query, crashes mobile apps

With pagination:
- `GET /api/products/?page=1` → returns 20 products + metadata on how to get the next page

---

## 2. PageNumberPagination

The most common and user-friendly format.

```python
# pagination.py
from rest_framework.pagination import PageNumberPagination

class StandardResultsPagination(PageNumberPagination):
    page_size = 20              # items per page
    page_size_query_param = 'page_size'  # client can change page size
    max_page_size = 100         # client can't request more than 100
    page_query_param = 'page'   # ?page=2
```

```python
# settings.py — apply globally
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'myapp.pagination.StandardResultsPagination',
    'PAGE_SIZE': 20,
}
```

```python
# views.py — override for specific ViewSet
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    pagination_class = StandardResultsPagination
```

### Response format:
```json
{
    "count": 150,
    "next": "http://api.example.com/products/?page=3",
    "previous": "http://api.example.com/products/?page=1",
    "results": [...]
}
```

---

## 3. LimitOffsetPagination

More flexible — allows clients to control offset directly.

```python
from rest_framework.pagination import LimitOffsetPagination

class LargeResultsPagination(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100
    # GET /api/products/?limit=10&offset=40
    # → items 41-50
```

### Use case: Infinite scroll, data exports.

---

## 4. CursorPagination

The most scalable — uses a cursor instead of page numbers. Great for real-time feeds.

```python
from rest_framework.pagination import CursorPagination

class PostCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must specify ordering field
    cursor_query_param = 'cursor'
    # GET /api/posts/?cursor=cD0yMDIzLTAxLTAxKzAw...
```

### When to use CursorPagination:
- Real-time data (social feeds, chat)
- Very large datasets (millions of rows)
- When page numbers don't make sense (data changes frequently)

---

## 5. Custom Pagination with Extra Metadata

```python
class CustomPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response({
            'pagination': {
                'total': self.page.paginator.count,
                'total_pages': self.page.paginator.num_pages,
                'current_page': self.page.number,
                'page_size': self.get_page_size(self.request),
                'has_next': self.page.has_next(),
                'has_previous': self.page.has_previous(),
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
            },
            'results': data
        })
```

---

## 6. Disable Pagination for Specific Views

```python
class AllCategoriesView(generics.ListAPIView):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    pagination_class = None  # categories are small, return all
```

---

## 7. API Versioning

Versioning allows you to evolve the API without breaking existing clients.

### Why version?
- v1: returns `{"full_name": "John Doe"}`
- v2: returns `{"first_name": "John", "last_name": "Doe"}`
- Old clients still use v1, new clients use v2

### Versioning Strategies in DRF:

#### URL Path Versioning (Most common, most explicit):
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

```python
# urls.py
urlpatterns = [
    path('api/<version>/posts/', PostListView.as_view()),
    # GET /api/v1/posts/
    # GET /api/v2/posts/
]
```

#### Query Parameter Versioning:
```python
# GET /api/posts/?version=v2
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
}
```

#### Accept Header Versioning:
```
Accept: application/json; version=v2
```

```python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
}
```

---

## 8. Using Version in Views

```python
class PostListView(generics.ListAPIView):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return PostSerializerV2
        return PostSerializerV1

    def get_queryset(self):
        qs = Post.objects.all()
        if self.request.version == 'v2':
            return qs.filter(is_published=True)  # v2 only shows published
        return qs
```

---

## 9. Best Practice: Organize by Version

```
myproject/
    api/
        v1/
            views.py
            serializers.py
            urls.py
        v2/
            views.py
            serializers.py
            urls.py
```

```python
# project/urls.py
urlpatterns = [
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]
```

---

## 📝 Key Takeaways
1. Always paginate list endpoints — never return unbounded querysets
2. Use `PageNumberPagination` for simple APIs, `CursorPagination` for feeds/real-time data
3. Let clients control page size within a maximum limit
4. Versioning is critical for long-lived APIs — plan for it from the start
5. URL versioning is the most explicit and cacheable strategy
6. Access `request.version` in views to branch behavior per version

