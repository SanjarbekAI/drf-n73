# Lesson 3: Views & ViewSets — From FBV to ViewSets

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain the evolution from FBV → CBV → GenericViews → ViewSets
- Use `APIView` for custom logic
- Use `GenericAPIView` mixins to reduce boilerplate
- Use `ModelViewSet` for full CRUD with minimal code
- Override ViewSet methods for custom behavior

---

## 1. The Evolution of Views

```
Function-Based Views (FBV)
        ↓  more organized
Class-Based Views (APIView)
        ↓  less repetition
Generic Views (ListCreateAPIView, etc.)
        ↓  least code for standard CRUD
ViewSets (ModelViewSet)
```

Each level trades flexibility for less code. Choose based on how much custom logic you need.

---

## 2. `APIView` — Class-Based, Full Control

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Post
from .serializers import PostSerializer

class PostListView(APIView):
    def get(self, request):
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)  # set author automatically
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class PostDetailView(APIView):
    def get_object(self, pk):
        try:
            return Post.objects.get(pk=pk)
        except Post.DoesNotExist:
            raise Http404

    def get(self, request, pk):
        post = self.get_object(pk)
        serializer = PostSerializer(post)
        return Response(serializer.data)

    def put(self, request, pk):
        post = self.get_object(pk)
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk):
        post = self.get_object(pk)
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## 3. Generic Views — Less Boilerplate

DRF provides ready-made combinations of mixins:

| Class | HTTP Methods | Purpose |
|-------|-------------|---------|
| `ListAPIView` | GET | List objects |
| `CreateAPIView` | POST | Create object |
| `RetrieveAPIView` | GET | Get single object |
| `UpdateAPIView` | PUT/PATCH | Update object |
| `DestroyAPIView` | DELETE | Delete object |
| `ListCreateAPIView` | GET, POST | List + Create |
| `RetrieveUpdateDestroyAPIView` | GET, PUT, PATCH, DELETE | Detail CRUD |

```python
from rest_framework import generics
from .models import Post
from .serializers import PostSerializer

class PostListCreateView(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def perform_create(self, serializer):
        # automatically set the author to the logged-in user
        serializer.save(author=self.request.user)

class PostRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

```python
# urls.py
urlpatterns = [
    path('posts/', PostListCreateView.as_view()),
    path('posts/<int:pk>/', PostRetrieveUpdateDestroyView.as_view()),
]
```

---

## 4. `ModelViewSet` — Full CRUD in ~5 Lines

```python
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

This single class handles:
- `GET /posts/` → `list()`
- `POST /posts/` → `create()`
- `GET /posts/1/` → `retrieve()`
- `PUT /posts/1/` → `update()`
- `PATCH /posts/1/` → `partial_update()`
- `DELETE /posts/1/` → `destroy()`

---

## 5. `ReadOnlyModelViewSet` — Safe Public APIs

```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    # Only GET (list and retrieve) — no POST/PUT/DELETE
```

---

## 6. Overriding ViewSet Methods

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_queryset(self):
        # Only return posts by the current user
        return Post.objects.filter(author=self.request.user)

    def get_serializer_class(self):
        # Use different serializer for reads vs writes
        if self.action in ['list', 'retrieve']:
            return PostReadSerializer
        return PostWriteSerializer

    def destroy(self, request, *args, **kwargs):
        # Soft delete instead of hard delete
        instance = self.get_object()
        instance.is_deleted = True
        instance.save()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## 7. Custom Actions with `@action`

Add extra endpoints beyond standard CRUD:

```python
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """POST /api/posts/1/publish/ — Publish a post"""
        post = self.get_object()
        post.published = True
        post.save()
        return Response({'status': 'post published'})

    @action(detail=True, methods=['post'])
    def like(self, request, pk=None):
        """POST /api/posts/1/like/ — Like a post"""
        post = self.get_object()
        post.likes += 1
        post.save()
        return Response({'likes': post.likes})

    @action(detail=False, methods=['get'])
    def trending(self, request):
        """GET /api/posts/trending/ — Get trending posts"""
        trending = Post.objects.order_by('-views_count')[:10]
        serializer = self.get_serializer(trending, many=True)
        return Response(serializer.data)
```

---

## 8. When to Use What?

| Scenario | Use |
|----------|-----|
| Standard CRUD, no custom logic | `ModelViewSet` |
| Standard CRUD + custom queryset/serializer | `ModelViewSet` with overrides |
| Only some actions (e.g., no delete) | `GenericAPIView` + mixins |
| Complex custom logic per action | `APIView` |
| Quick one-off endpoints | `@api_view` FBV |

---

## 📝 Key Takeaways
1. `ModelViewSet` gives you full CRUD for free — use it as the default
2. Override `get_queryset()` to filter what users can access
3. Override `get_serializer_class()` to use different serializers per action
4. `@action` decorator adds custom endpoints beyond standard CRUD
5. `perform_create()` / `perform_update()` are the right hooks to inject extra data (e.g., author)

