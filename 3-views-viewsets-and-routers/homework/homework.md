# Lesson 3 — Homework

## 📌 Assignment: E-Commerce API with Full URL Structure

### Part A: E-Commerce Product & Order API (ViewSets)

#### Models
```python
class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)

class Product(models.Model):
    name = models.CharField(max_length=200)
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='products')
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField(default=0)
    is_active = models.BooleanField(default=True)

class Order(models.Model):
    STATUS_CHOICES = [('pending', 'Pending'), ('shipped', 'Shipped'), ('delivered', 'Delivered')]
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    created_at = models.DateTimeField(auto_now_add=True)
```

#### Requirements

##### CategoryViewSet:
- Read-only (no one should create categories via API — use admin)
- List and retrieve only

##### ProductViewSet:
- Full CRUD (but only return `is_active=True` products by default)
- Override `destroy()` to set `is_active=False` (soft delete)
- Add a custom action `POST /products/<id>/restock/` that accepts `{"quantity": 50}` and adds to stock
- Add a custom action `GET /products/out-of-stock/` that returns all products where `stock=0`

##### OrderViewSet:
- List returns only the logged-in user's orders
- Create automatically sets `customer = request.user`
- Add a custom action `POST /orders/<id>/cancel/` that sets status to `'cancelled'` (only if current status is `'pending'`)

Return a proper error response from the cancel action if the order is not cancellable:
```json
{"error": "Cannot cancel an order that is already shipped or delivered."}
```

---

### Part B: Wire It All Up with Routers

#### URL Structure to Implement:
```
/api/categories/                              → list/retrieve categories
/api/categories/<id>/                         → retrieve category
/api/products/                                → list/create products
/api/products/<id>/                           → retrieve/update/delete product
/api/products/<product_pk>/reviews/           → list/create reviews for a product
/api/products/<product_pk>/reviews/<id>/      → retrieve/update/delete review
/api/products/<id>/restock/                   → POST: restock a product
/api/products/out-of-stock/                   → GET: all out-of-stock products
/api/orders/                                  → list/create orders
/api/orders/<id>/cancel/                      → POST: cancel an order
```

#### Requirements:
1. Use `DefaultRouter` for all top-level resources
2. Use `drf-nested-routers` for reviews under products
3. All ViewSets must use `get_queryset()` appropriately
4. No manual `path()` calls for ViewSet routes

---

### Bonus:
- Add `GET /api/products/<id>/similar/` that returns products in the same category
- Ensure reviews can only be edited or deleted by the review author

### Submission Checklist:
- [ ] All three ViewSets are implemented
- [ ] Custom actions work correctly
- [ ] Soft delete hides products from the list
- [ ] Users can only see their own orders
- [ ] Error handling is done properly with correct status codes
- [ ] All URLs are implemented using routers (no manual `path()` for ViewSet routes)
- [ ] Nested reviews route works correctly
- [ ] Code is organized in separate files (models, serializers, views, urls)

