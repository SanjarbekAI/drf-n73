# Lesson 11 — In-Class Tasks

## Task 1: Basic APITestCase (20 min)
Write tests for your Product API covering:
- `GET /api/products/` returns 200 and correct count
- `POST /api/products/` with valid data returns 201
- `POST /api/products/` with missing `name` returns 400 with error in `name` field
- `GET /api/products/999/` returns 404
- `DELETE /api/products/<id>/` requires authentication (returns 401 if no token)

Run: `python manage.py test` and make all tests pass.

---

## Task 2: Permission Tests (20 min)
Write tests that verify permission logic:
```python
def test_owner_can_delete_own_product(self): ...
def test_non_owner_cannot_delete_product(self): ...
def test_admin_can_delete_any_product(self): ...
def test_unauthenticated_cannot_create_product(self): ...
```

Use `force_authenticate()` to test with different users.

---

## Task 3: factory_boy Setup (15 min)
1. Install `factory-boy`
2. Create `UserFactory` and `ProductFactory`
3. Rewrite your setUp using factories:
```python
def setUp(self):
    self.user = UserFactory()
    self.products = ProductFactory.create_batch(20)
```

---

## Task 4: Switch to pytest (15 min)
1. Install `pytest-django`
2. Create `pytest.ini`
3. Rewrite 3 of your tests using `@pytest.mark.django_db` and fixtures
4. Run: `pytest -v`

---

## Task 5: Deep-Dive Swagger Configuration (30 min)

You already have basic Swagger from Lesson 1. Now configure it properly.

### Step 1 — Full SPECTACULAR_SETTINGS (10 min)
Update `settings.py` with the full config:
- Add title, description, version
- Add `COMPONENTS` with a `BearerAuth` security scheme so the 🔒 **Authorize** button works with JWT
- Add `TAGS` to group endpoints: `Auth`, `Posts`, `Comments`, `Users`

Test: open `/api/docs/` → click the **Authorize** button → it should show a Bearer token input field.

### Step 2 — Document your ViewSets (10 min)
Use `@extend_schema_view` to document all standard actions on `PostViewSet`:
```python
@extend_schema_view(
    list=extend_schema(summary='List posts', tags=['Posts']),
    create=extend_schema(summary='Create a post', tags=['Posts']),
    retrieve=extend_schema(summary='Get post detail', tags=['Posts']),
    update=extend_schema(summary='Update post', tags=['Posts']),
    destroy=extend_schema(summary='Delete post', tags=['Posts']),
)
class PostViewSet(viewsets.ModelViewSet):
    ...
```

Verify: endpoints are grouped under the **Posts** tag in Swagger UI.

### Step 3 — Document a Custom Action (5 min)
Add `@extend_schema` to your `publish` action with:
- A clear `summary` and `description`
- `request=None` (no body needed)
- A `200` response example: `{"status": "published"}`
- A `403` response example: `{"detail": "Permission denied"}`

### Step 4 — Export to Postman (5 min)
```bash
python manage.py spectacular --file schema.yml --validate
```
Open Postman → Import → select `schema.yml` → your entire API collection is auto-created.

How many requests does Postman create from your schema?

---

## Task 6: Check Coverage
```bash
pip install pytest-cov
pytest --cov=myapp --cov-report=html
# Open htmlcov/index.html
```

What percentage coverage do you have? What's missing?

