# Lesson 6: Filtering, Searching & Ordering

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Implement URL-based filtering with `django-filter`
- Add full-text search with `SearchFilter`
- Add sorting with `OrderingFilter`
- Write custom filter backends
- Combine all three in a single endpoint

---

## 1. Why Filtering?

Without filtering, a client would have to:
1. Download ALL records
2. Filter client-side

This is slow, wastes bandwidth, and exposes data the user shouldn't see.

```
GET /api/products/?category=electronics&min_price=100&max_price=500&ordering=-price&search=samsung
```

---

## 2. Setup: `django-filter`

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ]
}
```

---

## 3. Basic `filterset_fields` (Quick Setup)

```python
from rest_framework import viewsets
from django_filters.rest_framework import DjangoFilterBackend
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['category', 'is_active', 'brand']
    # Usage: GET /api/products/?category=electronics&is_active=true
```

---

## 4. Custom FilterSet — Range, Lookups, etc.

```python
# filters.py
import django_filters
from .models import Product

class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    name = django_filters.CharFilter(field_name='name', lookup_expr='icontains')
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    category_name = django_filters.CharFilter(field_name='category__name', lookup_expr='iexact')

    class Meta:
        model = Product
        fields = ['min_price', 'max_price', 'name', 'category', 'is_active']
```

```python
# views.py
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = ProductFilter
    # GET /api/products/?min_price=100&max_price=500&name=samsung
```

### Common `lookup_expr` values:
| Expression | SQL | Example |
|-----------|-----|---------|
| `exact` | `= 'value'` | `?status=active` |
| `iexact` | `ILIKE 'value'` | `?name=samsung` |
| `contains` | `LIKE '%value%'` | `?name__contains=sam` |
| `icontains` | `ILIKE '%value%'` | `?name=sam` |
| `gt` | `> value` | `?price__gt=100` |
| `gte` | `>= value` | `?min_price=100` |
| `lt` | `< value` | `?price__lt=500` |
| `lte` | `<= value` | `?max_price=500` |
| `in` | `IN (1,2,3)` | `?id__in=1,2,3` |

---

## 5. Search Filter

```python
from rest_framework.filters import SearchFilter

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [SearchFilter]
    search_fields = ['name', 'description', 'category__name']
    # GET /api/products/?search=samsung
```

### Search field prefixes:
| Prefix | Behavior |
|--------|---------|
| `name` (no prefix) | Case-insensitive contains |
| `^name` | Starts-with search |
| `=name` | Exact match |
| `@name` | Full-text search (PostgreSQL) |
| `$name` | Regex search |

```python
search_fields = ['^name', '=isbn', '@description']
```

---

## 6. Ordering Filter

```python
from rest_framework.filters import OrderingFilter

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [OrderingFilter]
    ordering_fields = ['price', 'name', 'created_at', 'stock']
    ordering = ['-created_at']  # default ordering
    # GET /api/products/?ordering=price         → ascending by price
    # GET /api/products/?ordering=-price        → descending by price
    # GET /api/products/?ordering=price,-name   → price ASC, name DESC
```

---

## 7. Combining All Three

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = ProductFilter
    search_fields = ['name', 'description', 'category__name']
    ordering_fields = ['price', 'name', 'created_at']
    ordering = ['-created_at']

# Full example:
# GET /api/products/?search=laptop&min_price=500&max_price=2000&ordering=-price&category=electronics
```

---

## 8. Custom Filter Backend

For advanced cases — filter based on the current user:

```python
# filters.py
from rest_framework import filters

class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """Filter queryset to only show objects owned by the requesting user."""
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(author=request.user)

class ActiveOnlyFilterBackend(filters.BaseFilterBackend):
    """Only show active/non-deleted records."""
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(is_active=True)
```

```python
class MyPostsViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [IsOwnerFilterBackend, SearchFilter, OrderingFilter]
    search_fields = ['title', 'content']
```

---

## 9. Filtering in the QuerySet vs FilterBackend

```python
# Option A: Filter in get_queryset() — for user-specific, security-critical filtering
def get_queryset(self):
    return Post.objects.filter(author=self.request.user)

# Option B: FilterBackend — for flexible, optional filtering by clients
filter_backends = [DjangoFilterBackend]
filterset_fields = ['status', 'category']

# Best practice: combine both
# get_queryset() for SECURITY (what user CAN see)
# filter_backends for UX (how user CHOOSES to filter within that scope)
```

---

## 📝 Key Takeaways
1. `django-filter` handles complex field-based filtering (ranges, lookups)
2. `SearchFilter` handles full-text search across multiple fields
3. `OrderingFilter` handles client-controlled sorting
4. Always add `select_related`/`prefetch_related` when filtering on related fields
5. Use `get_queryset()` for security-critical filters; filter backends for user preferences
6. Filtering, searching, and ordering can all be combined in one view

