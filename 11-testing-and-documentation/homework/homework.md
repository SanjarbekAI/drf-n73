# Lesson 11 — Homework

## 📌 Assignment: Full Test Suite + Documentation

### Part 1: Complete Test Suite
Write a comprehensive test suite for your Blog API. Minimum coverage: **80%**.

Required test classes:

#### `AuthTestCase`
- Register with valid data → 201, tokens returned
- Register with duplicate email → 400
- Register with weak password → 400
- Login with correct credentials → 200, tokens returned
- Login with wrong password → 401
- Access protected endpoint without token → 401
- Access protected endpoint with expired token → 401

#### `PostTestCase`
- List posts (anonymous) → 200, paginated
- List posts (authenticated) → 200
- Create post (authenticated) → 201, author set automatically
- Create post (missing fields) → 400 with field errors
- Update own post → 200
- Update other user's post → 403
- Delete own post → 204
- Delete other user's post → 403
- Publish post (custom action) → 200, post.published = True

#### `CommentTestCase`
- List comments for a post → 200
- Create comment (authenticated) → 201
- Create comment (anonymous) → 401
- Delete own comment → 204
- Delete other user's comment → 403

#### `FilterTestCase`
- Filter posts by category → correct results
- Search posts by title → correct results
- Order posts by created_at desc → correct order
- Combine search + filter + ordering → correct results

### Part 2: Factory Definitions
Create factories for all your models:
```
UserFactory, PostFactory, CategoryFactory,
CommentFactory, TagFactory, LikeFactory
```

Each factory should use `faker` for realistic test data.

### Part 3: API Documentation
Set up `drf-spectacular` and document all endpoints:

Required for each endpoint:
- `summary` — one line description
- `description` — what it does, auth required, etc.
- `responses` — what status codes it returns
- `parameters` — query params (filters, pagination)

Add a `tags` grouping so the docs are organized:
```python
@extend_schema(tags=['Posts'])
class PostViewSet(...): ...

@extend_schema(tags=['Auth'])
class LoginView(...): ...
```

### Part 4: Test Celery Tasks
Write tests for async tasks using `task.apply()` (runs synchronously in tests):
```python
@pytest.mark.django_db
def test_send_welcome_email(mailoutbox):
    user = UserFactory()
    send_welcome_email.apply(args=[user.id])
    assert len(mailoutbox) == 1
    assert mailoutbox[0].to == [user.email]
```

### Bonus — Parametrized Tests
Use `@pytest.mark.parametrize` to test multiple invalid inputs efficiently:
```python
@pytest.mark.parametrize('bad_password', [
    'short',
    'nouppercase1!',
    'NOLOWERCASE1!',
    'NoNumbers!',
    'NoSpecialChar1',
])
@pytest.mark.django_db
def test_register_rejects_weak_password(api_client, bad_password):
    response = api_client.post('/api/auth/register/', {
        'username': 'user1',
        'email': 'user@example.com',
        'password': bad_password,
        'confirm_password': bad_password,
    })
    assert response.status_code == 400
```

### Submission Checklist:
- [ ] All test classes are implemented
- [ ] Coverage is ≥ 80% (`pytest --cov`)
- [ ] All factories are defined and used in tests
- [ ] Swagger docs are live at `/api/docs/`
- [ ] All endpoints have `@extend_schema` with summary and description
- [ ] Celery task tests use `.apply()` (synchronous in test context)

