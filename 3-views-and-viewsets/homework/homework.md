# Lesson 3 — Homework

## 📌 Assignment: E-Commerce Product & Order API

### Models
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

### Requirements

#### CategoryViewSet:
- Read-only (no one should create categories via API — use admin)
- List and retrieve only

#### ProductViewSet:
- Full CRUD (but only return `is_active=True` products by default)
- Override `destroy()` to set `is_active=False` (soft delete)
- Add a custom action `POST /products/<id>/restock/` that accepts `{"quantity": 50}` and adds to stock
- Add a custom action `GET /products/out-of-stock/` that returns all products where `stock=0`

#### OrderViewSet:
- List returns only the logged-in user's orders
- Create automatically sets `customer = request.user`
- Add a custom action `POST /orders/<id>/cancel/` that sets status to `'cancelled'` (only if current status is `'pending'`)

### Bonus:
Return a proper error response from the cancel action if the order is not cancellable:
```json
{"error": "Cannot cancel an order that is already shipped or delivered."}
```

### Submission Checklist:
- [ ] All three ViewSets are implemented
- [ ] Custom actions work correctly
- [ ] Soft delete hides products from the list
- [ ] Users can only see their own orders
- [ ] Error handling is done properly with correct status codes

