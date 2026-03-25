# Lesson 11: Testing & API Documentation

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Write unit and integration tests for DRF APIs using `APITestCase`
- Use `pytest-django` for more expressive tests
- Test authentication, permissions, and serializer validation
- Use factories with `factory_boy` to generate test data
- Generate interactive API documentation with Swagger (drf-spectacular)

---

## 1. Why Test APIs?

Testing ensures:
- Endpoints return correct data and status codes
- Authentication and permissions work correctly
- Serializer validation catches bad input
- Business logic (create, update, delete) behaves correctly
- Regressions are caught before deployment

**Test Pyramid for APIs:**
```
         /\
        /E2E\       ← End-to-end (Postman, browser)
       /------\
      /Integration\ ← API endpoint tests (DRF TestCase)
     /------------\
    /   Unit Tests  \ ← serializers, models, utils
   /----------------\
```

---

## 2. DRF `APITestCase`

```python
# tests/test_posts.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth.models import User
from django.urls import reverse
from myapp.models import Post

class PostAPITestCase(APITestCase):

    def setUp(self):
        """Runs before each test method."""
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123',
            email='test@example.com'
        )
        self.other_user = User.objects.create_user(
            username='otheruser',
            password='testpass123',
        )
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content for this post.',
            author=self.user
        )
        self.client = APIClient()

    def test_list_posts_unauthenticated(self):
        """Anyone can list posts."""
        url = reverse('post-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_create_post_authenticated(self):
        """Authenticated users can create posts."""
        self.client.force_authenticate(user=self.user)
        url = reverse('post-list')
        data = {'title': 'New Post', 'content': 'Some content here.'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Post.objects.count(), 2)
        self.assertEqual(response.data['title'], 'New Post')

    def test_create_post_unauthenticated(self):
        """Anonymous users cannot create posts."""
        url = reverse('post-list')
        data = {'title': 'New Post', 'content': 'Content'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_update_post_owner(self):
        """Only the post owner can update their post."""
        self.client.force_authenticate(user=self.user)
        url = reverse('post-detail', kwargs={'pk': self.post.pk})
        response = self.client.patch(url, {'title': 'Updated Title'}, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated Title')

    def test_update_post_non_owner(self):
        """Non-owners cannot update someone else's post."""
        self.client.force_authenticate(user=self.other_user)
        url = reverse('post-detail', kwargs={'pk': self.post.pk})
        response = self.client.patch(url, {'title': 'Hacked!'}, format='json')
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_delete_post_owner(self):
        """Owner can delete their post."""
        self.client.force_authenticate(user=self.user)
        url = reverse('post-detail', kwargs={'pk': self.post.pk})
        response = self.client.delete(url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Post.objects.count(), 0)
```

---

## 3. Testing with JWT Tokens

```python
class JWTAuthTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='user', password='pass123')
        self.login_url = reverse('token_obtain_pair')

    def get_tokens(self):
        response = self.client.post(self.login_url, {'username': 'user', 'password': 'pass123'})
        return response.data['access'], response.data['refresh']

    def test_protected_endpoint_with_jwt(self):
        access_token, _ = self.get_tokens()
        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {access_token}')
        response = self.client.get(reverse('post-list'))
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_invalid_token_rejected(self):
        self.client.credentials(HTTP_AUTHORIZATION='Bearer invalid.token.here')
        response = self.client.get(reverse('post-list'))
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

---

## 4. Testing Serializer Validation

```python
# tests/test_serializers.py
from django.test import TestCase
from myapp.serializers import PostSerializer

class PostSerializerTestCase(TestCase):
    def test_valid_data(self):
        data = {'title': 'Valid Title', 'content': 'Valid content here.'}
        serializer = PostSerializer(data=data)
        self.assertTrue(serializer.is_valid())

    def test_title_too_short(self):
        data = {'title': 'Hi', 'content': 'Valid content here.'}
        serializer = PostSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn('title', serializer.errors)

    def test_missing_required_field(self):
        data = {'title': 'Only title, no content'}
        serializer = PostSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn('content', serializer.errors)
```

---

## 5. `factory_boy` — Test Data Factories

```bash
pip install factory-boy
```

```python
# tests/factories.py
import factory
from django.contrib.auth.models import User
from myapp.models import Post, Category

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')

class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category
    name = factory.Faker('word')

class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post
    title = factory.Faker('sentence', nb_words=5)
    content = factory.Faker('paragraphs', nb=3, ext_word_list=None)
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
```

```python
# In tests
class PostListTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.posts = PostFactory.create_batch(25)  # create 25 posts at once
```

---

## 6. `pytest-django` — More Expressive Tests

```bash
pip install pytest-django pytest
```

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
python_files = tests.py test_*.py *_tests.py
```

```python
# tests/test_posts.py (pytest style)
import pytest
from rest_framework.test import APIClient
from myapp.tests.factories import UserFactory, PostFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client):
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client, user

@pytest.mark.django_db
def test_list_posts(api_client):
    PostFactory.create_batch(5)
    response = api_client.get('/api/posts/')
    assert response.status_code == 200
    assert len(response.data['results']) == 5

@pytest.mark.django_db
def test_create_post_requires_auth(api_client):
    response = api_client.post('/api/posts/', {'title': 'T', 'content': 'C'})
    assert response.status_code == 401

@pytest.mark.django_db
def test_create_post_authenticated(authenticated_client):
    client, user = authenticated_client
    response = client.post('/api/posts/', {'title': 'My Post', 'content': 'Content here'})
    assert response.status_code == 201
    assert response.data['title'] == 'My Post'
```

---

## 7. API Documentation with drf-spectacular — Full Setup

> You already installed `drf-spectacular` in Lesson 1 for basic usage.
> In this lesson we go deep: full configuration, JWT auth in Swagger UI,
> grouping with tags, response examples, and exporting to Postman.

---

### 7.1 Full `SPECTACULAR_SETTINGS`

```python
# settings.py
INSTALLED_APPS = [
    ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    # ─── Basic Info ────────────────────────────────────────────────
    'TITLE': 'Blog API',
    'DESCRIPTION': '''
A full-featured Blog API built with Django REST Framework.

## Authentication
This API uses **JWT Bearer tokens**.
1. Call `POST /api/auth/token/` with your credentials to get tokens.
2. Click **Authorize** (🔒) at the top right and enter: `Bearer <your_access_token>`
3. All protected endpoints will now work.
    ''',
    'VERSION': '1.0.0',
    'CONTACT': {'name': 'API Support', 'email': 'support@example.com'},
    'LICENSE': {'name': 'MIT'},

    # ─── Schema Generation ─────────────────────────────────────────
    'SERVE_INCLUDE_SCHEMA': False,      # don't show raw schema in docs
    'COMPONENT_SPLIT_REQUEST': True,    # separate request/response schemas
    'SORT_OPERATIONS': True,            # alphabetical endpoint order

    # ─── Security — teach Swagger UI how to send JWT ───────────────
    'SECURITY': [{'BearerAuth': []}],
    'COMPONENTS': {
        'securitySchemes': {
            'BearerAuth': {
                'type': 'http',
                'scheme': 'bearer',
                'bearerFormat': 'JWT',
                'description': 'Enter your JWT access token. Get one from POST /api/auth/token/',
            }
        }
    },

    # ─── Tags ordering in the sidebar ──────────────────────────────
    'TAGS': [
        {'name': 'Auth',      'description': 'Registration, login, logout, token refresh'},
        {'name': 'Posts',     'description': 'Blog post CRUD and actions'},
        {'name': 'Comments',  'description': 'Comments on posts'},
        {'name': 'Users',     'description': 'User profiles'},
    ],
}
```

---

### 7.2 URLs

```python
# myproject/urls.py
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),

    # ─── Documentation ────────────────────────────────────────
    path('api/schema/',  SpectacularAPIView.as_view(),                          name='schema'),
    path('api/docs/',    SpectacularSwaggerView.as_view(url_name='schema'),     name='swagger-ui'),
    path('api/redoc/',   SpectacularRedocView.as_view(url_name='schema'),       name='redoc'),
]
```

| URL | Purpose |
|-----|---------|
| `/api/docs/` | Swagger UI — interactive, try endpoints directly |
| `/api/redoc/` | ReDoc — clean readable reference |
| `/api/schema/` | Raw OpenAPI 3.0 JSON — import into Postman/Insomnia |

---

### 7.3 `@extend_schema` — Documenting Every Endpoint

#### On a whole ViewSet (applies to all actions):
```python
from drf_spectacular.utils import extend_schema, extend_schema_view, OpenApiParameter, OpenApiExample
from drf_spectacular.types import OpenApiTypes

@extend_schema_view(
    list=extend_schema(
        summary='List all posts',
        description='Returns a paginated list of posts. Supports filtering, searching, and ordering.',
        tags=['Posts'],
    ),
    create=extend_schema(
        summary='Create a new post',
        description='Creates a post. The author is automatically set to the authenticated user.',
        tags=['Posts'],
    ),
    retrieve=extend_schema(summary='Get post details',    tags=['Posts']),
    update=extend_schema(summary='Full update of a post', tags=['Posts']),
    partial_update=extend_schema(summary='Partial update', tags=['Posts']),
    destroy=extend_schema(summary='Delete a post',         tags=['Posts']),
)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

#### On a custom `@action`:
```python
    @extend_schema(
        summary='Publish a post',
        description='Marks the post as published. Only the post owner can do this.',
        tags=['Posts'],
        request=None,   # no request body needed
        responses={
            200: OpenApiExample(
                'Success',
                value={'status': 'published'},
                response_only=True,
            ),
            403: OpenApiExample(
                'Forbidden',
                value={'detail': 'You do not have permission to perform this action.'},
                response_only=True,
            ),
        },
    )
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        ...
```

#### Documenting query parameters (filters):
```python
    @extend_schema(
        summary='List all posts',
        parameters=[
            OpenApiParameter('search',   OpenApiTypes.STR,  description='Search in title and content'),
            OpenApiParameter('ordering', OpenApiTypes.STR,  description='Sort by: created_at, -created_at, title'),
            OpenApiParameter('category', OpenApiTypes.INT,  description='Filter by category ID'),
            OpenApiParameter('page',     OpenApiTypes.INT,  description='Page number'),
            OpenApiParameter('page_size',OpenApiTypes.INT,  description='Items per page (max 100)'),
        ],
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

---

### 7.4 Documenting Auth Endpoints with Examples

```python
from drf_spectacular.utils import extend_schema, OpenApiExample

class LoginView(APIView):
    permission_classes = [AllowAny]

    @extend_schema(
        summary='Obtain JWT tokens',
        description='Login with username and password. Returns access and refresh JWT tokens.',
        tags=['Auth'],
        request=OpenApiExample(
            'Login Request',
            value={'username': 'john', 'password': 'mypassword123'},
            request_only=True,
        ),
        responses={
            200: OpenApiExample(
                'Success',
                value={
                    'access': 'eyJhbGciOiJIUzI1NiIs...',
                    'refresh': 'eyJhbGciOiJIUzI1NiIs...',
                },
                response_only=True,
            ),
            401: OpenApiExample(
                'Invalid credentials',
                value={'detail': 'No active account found with the given credentials'},
                response_only=True,
            ),
        },
    )
    def post(self, request):
        ...
```

---

### 7.5 `@extend_schema` on Serializer Fields

Add descriptions directly in serializers — they appear in the Swagger schema:

```python
from drf_spectacular.utils import extend_schema_field
from drf_spectacular.types import OpenApiTypes

class PostSerializer(serializers.ModelSerializer):
    word_count = serializers.SerializerMethodField()
    reading_time = serializers.SerializerMethodField()

    @extend_schema_field(OpenApiTypes.INT)
    def get_word_count(self, obj):
        return len(obj.content.split())

    @extend_schema_field({'type': 'integer', 'description': 'Estimated reading time in minutes'})
    def get_reading_time(self, obj):
        return max(1, len(obj.content.split()) // 200)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'word_count', 'reading_time']
```

---

### 7.6 Export Schema for Postman / Insomnia

```bash
# Export OpenAPI schema to a file
python manage.py spectacular --file schema.yml --validate

# Import schema.yml into:
# - Postman: Import → OpenAPI → choose schema.yml
# - Insomnia: Preferences → Data → Import Data → From File
```

This generates the full collection of all API endpoints automatically.

---

### 7.7 Hide Endpoints from Docs

```python
from drf_spectacular.utils import extend_schema

# Hide a single action (e.g., internal-only endpoint)
@extend_schema(exclude=True)
@action(detail=False, methods=['post'])
def internal_sync(self, request):
    ...

# Hide entire ViewSet from docs
@extend_schema(exclude=True)
class InternalWebhookView(APIView):
    ...
```

---

## 8. Summary: Where to Put Swagger Setup

| File | What goes there |
|------|----------------|
| `settings.py` | `INSTALLED_APPS`, `DEFAULT_SCHEMA_CLASS`, full `SPECTACULAR_SETTINGS` with JWT security |
| `urls.py` | The 3 schema/docs/redoc URL patterns |
| `views.py` / `viewsets.py` | `@extend_schema_view` on class, `@extend_schema` on each action |
| `serializers.py` | `@extend_schema_field` on `SerializerMethodField` methods |
| `manage.py spectacular` | Export schema to `.yml` for Postman import |

---

## 📝 Key Takeaways
1. Always test: authentication, permissions, validation, and business logic
2. Use `force_authenticate()` to bypass auth in tests — focus on the logic being tested
3. `factory_boy` makes test data creation clean and reusable
4. `pytest-django` with fixtures is more expressive than `unittest`-style tests
5. Set up `drf-spectacular` in **Lesson 1** and document as you build — don't leave it to the end
6. The `SPECTACULAR_SETTINGS` `COMPONENTS` block is how you wire JWT auth into the Swagger UI "Authorize" button
7. `@extend_schema_view` documents every standard action at the class level — cleaner than decorating each method
8. Export with `python manage.py spectacular --file schema.yml` to share with frontend teams or import into Postman
9. Aim for >80% test coverage on view and serializer code

