# Lesson 1 — Homework

## 📌 Assignment: Build a Product Catalog API

### Context
You are building the backend for an online store. Your task is to create a basic Product API.

### Requirements

#### Model: `Product`
```
- name          (CharField, max 200)
- description   (TextField)
- price         (DecimalField, max_digits=10, decimal_places=2)
- stock         (IntegerField, default=0)
- created_at    (DateTimeField, auto_now_add)
```

#### Endpoints to implement:
| Method | URL | Description |
|--------|-----|-------------|
| GET | `/api/products/` | List all products |
| POST | `/api/products/` | Create a product |
| GET | `/api/products/<id>/` | Get product detail |
| PUT | `/api/products/<id>/` | Full update |
| PATCH | `/api/products/<id>/` | Partial update (only update fields sent) |
| DELETE | `/api/products/<id>/` | Delete product |

#### Bonus Challenges:
1. Add a `category` field (CharField) to the Product model and include it in the API
2. Return `404` with a custom error message: `{"error": "Product not found"}` instead of an empty response
3. Add a computed field `in_stock` (boolean) to the serializer that returns `True` if `stock > 0`

### Submission Checklist:
- [ ] All 6 endpoints work correctly
- [ ] Correct HTTP status codes are used
- [ ] Admin is set up so products can be created from the admin panel
- [ ] At least 5 products are seeded in the database
- [ ] Code is clean and organized (no unused imports)

### 💡 Hint for computed field:
```python
class ProductSerializer(serializers.ModelSerializer):
    in_stock = serializers.SerializerMethodField()

    def get_in_stock(self, obj):
        return obj.stock > 0

    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'stock', 'in_stock', 'created_at']
```

