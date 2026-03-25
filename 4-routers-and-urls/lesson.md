# Lesson 4: Routers & URL Patterns

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Use DRF Routers to auto-generate URL patterns
- Understand the difference between `SimpleRouter` and `DefaultRouter`
- Design clean, consistent RESTful URL structures
- Build nested routes for related resources
- Use `drf-nested-routers` for complex hierarchies

---

## 1. Why Routers?

Without a router, you have to manually register every URL:
```python
# Without router — repetitive and error-prone
urlpatterns = [
    path('posts/', PostListView.as_view()),
    path('posts/<int:pk>/', PostDetailView.as_view()),
    path('products/', ProductListView.as_view()),
    path('products/<int:pk>/', ProductDetailView.as_view()),
    # ... repeat for every resource
]
```

With a router:
```python
router.register('posts', PostViewSet)
router.register('products', ProductViewSet)
# Done. All 6 URLs are auto-generated.
```

---

## 2. DefaultRouter — Setup & Usage

```python
# myapp/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSet, ProductViewSet, CategoryViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')
router.register('products', ProductViewSet, basename='product')
router.register('categories', CategoryViewSet, basename='category')

urlpatterns = router.urls
```

```python
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),
]
```

### Auto-generated URLs from `router.register('posts', PostViewSet)`:
```
GET    /api/posts/          → list
POST   /api/posts/          → create
GET    /api/posts/{id}/     → retrieve
PUT    /api/posts/{id}/     → update
PATCH  /api/posts/{id}/     → partial_update
DELETE /api/posts/{id}/     → destroy
GET    /api/posts/{id}/publish/   → custom @action
```

---

## 3. `SimpleRouter` vs `DefaultRouter`

| Feature | SimpleRouter | DefaultRouter |
|---------|-------------|---------------|
| Auto URL generation | ✅ | ✅ |
| API root page (`/api/`) | ❌ | ✅ |
| Format suffixes (`.json`) | Optional | Optional |
| Use in production | Preferred | Development/Testing |

```python
# Production preference — no root browser page
from rest_framework.routers import SimpleRouter
router = SimpleRouter()
```

---

## 4. `basename` — Why It Matters

The `basename` is used to generate URL names (for `reverse()` and DRF's hyperlinked serializers):

```python
router.register('posts', PostViewSet, basename='post')
# Generates:
#   post-list    → /api/posts/
#   post-detail  → /api/posts/{pk}/
```

**When is `basename` required?**
- When your ViewSet doesn't have a `queryset` attribute
- When you override `get_queryset()` dynamically

```python
class UserPostViewSet(viewsets.ModelViewSet):
    # No queryset here — it's dynamic
    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)

# Must provide basename:
router.register('my-posts', UserPostViewSet, basename='user-post')
```

---

## 5. Custom Action URLs

`@action` decorators on a ViewSet automatically get their URL registered:

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @action(detail=True, methods=['post'], url_path='publish', url_name='publish')
    def publish(self, request, pk=None):
        # POST /api/posts/{pk}/publish/
        ...

    @action(detail=False, methods=['get'], url_path='featured', url_name='featured')
    def featured(self, request):
        # GET /api/posts/featured/
        ...
```

- `detail=True` → URL includes `{pk}` (e.g., `/posts/5/publish/`)
- `detail=False` → No `{pk}` (e.g., `/posts/featured/`)
- `url_path` → Custom URL segment (default: method name)
- `url_name` → Custom URL name suffix (default: method name with `-` replacing `_`)

---

## 6. Nested Routes with `drf-nested-routers`

```bash
pip install drf-nested-routers
```

```python
# Use case: /api/posts/{post_pk}/comments/
from rest_framework_nested import routers

router = routers.DefaultRouter()
router.register('posts', PostViewSet, basename='post')

# Nested: /api/posts/{post_pk}/comments/
posts_router = routers.NestedDefaultRouter(router, 'posts', lookup='post')
posts_router.register('comments', CommentViewSet, basename='post-comment')

urlpatterns = router.urls + posts_router.urls
```

```python
# ViewSet uses parent lookup kwarg
class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer

    def get_queryset(self):
        # post_pk comes from the URL
        return Comment.objects.filter(post_id=self.kwargs['post_pk'])

    def perform_create(self, serializer):
        post = Post.objects.get(pk=self.kwargs['post_pk'])
        serializer.save(post=post, author=self.request.user)
```

---

## 7. RESTful URL Design Best Practices

```
✅ GOOD                           ❌ BAD
/api/posts/                       /api/getPosts/
/api/posts/5/                     /api/post?id=5
/api/posts/5/comments/            /api/getPostComments/5/
POST /api/posts/5/publish/        /api/publishPost/5/
GET  /api/users/me/               /api/getCurrentUser/

Rules:
- Use nouns, not verbs
- Use plural names (/posts/, not /post/)
- Lowercase and hyphens (not underscores or camelCase)
- Nested resources for genuine ownership relationships
- Keep nesting max 2 levels deep
```

---

## 8. Versioning via URLs

```python
# v1 and v2 of the API
urlpatterns = [
    path('api/v1/', include('myapp.v1.urls')),
    path('api/v2/', include('myapp.v2.urls')),
]
```

---

## 📝 Key Takeaways
1. Routers auto-generate all standard RESTful URLs for a ViewSet
2. Use `DefaultRouter` during development (has a browsable API root), `SimpleRouter` in production
3. Always set `basename` when your ViewSet has a dynamic queryset
4. `@action(detail=True)` for resource-specific actions, `detail=False` for collection actions
5. Nested routes should reflect ownership, not just relationships
6. Keep URL design noun-based, plural, lowercase

