# Lesson 4: Authentication & Permissions

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain the difference between Authentication and Authorization
- Implement Session, Token, and JWT authentication
- Understand the internal structure of a JWT token
- Customize JWT claims and create custom token serializers
- Rotate, verify, and blacklist JWT tokens properly
- Use built-in DRF permission classes
- Write custom permission classes
- Apply object-level permissions
- Control permissions per ViewSet action
- Integrate drf-spectacular for API documentation with JWT support

---

## 1. Authentication vs Authorization

```
Authentication → WHO are you?  (Identity)
Authorization  → WHAT can you do? (Permissions)
```

A user can be authenticated but still not authorized:
- Logged-in regular user trying to access admin panel ← authenticated, NOT authorized
- Admin user accessing their own data ← authenticated AND authorized

---

## 2. Authentication Classes

### 2.1 Session Authentication (default Django)
Uses Django's session framework — good for browser-based clients.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

### 2.2 Token Authentication
Each user gets a token stored in the database. Sent in the `Authorization` header.

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}
```

```bash
python manage.py migrate  # creates authtoken_token table
```

#### Generate tokens:
```python
from rest_framework.authtoken.models import Token
from django.contrib.auth.models import User

# Auto-create token when user registers
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
```

#### Login endpoint:
```python
# urls.py
from rest_framework.authtoken.views import obtain_auth_token

urlpatterns = [
    path('api/auth/login/', obtain_auth_token),
]
```

#### Usage by client:
```
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

### 2.3 JWT Authentication (Recommended for modern APIs)
JSON Web Tokens — stateless, no database lookup needed per request.

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
from datetime import timedelta

INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ]
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/auth/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

#### JWT Flow:
```
1. POST /api/auth/token/ → { "access": "...", "refresh": "..." }
2. Use access token in header: Authorization: Bearer <access_token>
3. When access expires: POST /api/auth/token/refresh/ with refresh token
```

---

## 3. Deep Dive: How JWT Works Internally

### 3.1 JWT Structure

A JWT is three Base64URL-encoded parts joined by dots:

```
xxxxx.yyyyy.zzzzz
  │      │      └── Signature
  │      └───────── Payload (Claims)
  └──────────────── Header
```

#### Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### Payload (Claims)
```json
{
  "token_type": "access",
  "exp": 1700000000,
  "iat": 1699996400,
  "jti": "unique-token-id",
  "user_id": 42
}
```

| Claim | Meaning |
|-------|---------|
| `exp` | Expiration timestamp (Unix) |
| `iat` | Issued at timestamp |
| `jti` | JWT ID — unique identifier for this token |
| `token_type` | `"access"` or `"refresh"` |
| `user_id` | The authenticated user's PK |

#### Signature
```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  SECRET_KEY
)
```

> 💡 The signature prevents tampering. The payload is **readable** by anyone — never store passwords or sensitive data inside a JWT!

### 3.2 Decoding a Token (for debugging)

```python
import base64, json

def decode_jwt_payload(token: str) -> dict:
    """Decode JWT payload WITHOUT verifying the signature (debug only)."""
    payload_b64 = token.split(".")[1]
    # Add padding if necessary
    padding = 4 - len(payload_b64) % 4
    payload_b64 += "=" * padding
    return json.loads(base64.urlsafe_b64decode(payload_b64))

# Usage
token = "eyJ0eXAiOiJKV1..."
print(decode_jwt_payload(token))
# {'token_type': 'access', 'exp': 1700000000, 'user_id': 42, ...}
```

> ⚠️ Never use this to validate tokens in production. Always let simplejwt verify the signature.

---

## 4. Advanced simplejwt Configuration

### 4.1 Full SIMPLE_JWT Settings Reference

```python
# settings.py
from datetime import timedelta

SIMPLE_JWT = {
    # Token lifetimes
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),   # Short-lived for security
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),

    # Rotation: every time you use the refresh token, you get a NEW refresh token
    'ROTATE_REFRESH_TOKENS': True,
    # Invalidate the old refresh token after rotation
    'BLACKLIST_AFTER_ROTATION': True,

    # Algorithm (HS256 = symmetric, RS256 = asymmetric/public key)
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,           # Use env variable in production!

    # Header format
    'AUTH_HEADER_TYPES': ('Bearer',),    # Authorization: Bearer <token>
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',

    # Which user field maps to the token's user_id claim
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',

    # Claim that identifies token type
    'TOKEN_TYPE_CLAIM': 'token_type',

    # JWT ID claim — used for blacklisting
    'JTI_CLAIM': 'jti',

    # Custom serializer classes (see section 4.2)
    'TOKEN_OBTAIN_SERIALIZER': 'rest_framework_simplejwt.serializers.TokenObtainPairSerializer',
    'TOKEN_REFRESH_SERIALIZER': 'rest_framework_simplejwt.serializers.TokenRefreshSerializer',
    'TOKEN_VERIFY_SERIALIZER': 'rest_framework_simplejwt.serializers.TokenVerifySerializer',
}
```

### 4.2 Token Verification Endpoint

Add a `/verify/` endpoint so clients can check if an access token is still valid:

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,       # ← new
)

urlpatterns = [
    path('api/auth/token/',          TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/',  TokenRefreshView.as_view(),    name='token_refresh'),
    path('api/auth/token/verify/',   TokenVerifyView.as_view(),     name='token_verify'),
]
```

```bash
# POST /api/auth/token/verify/
# Body: { "token": "<access_token>" }
# Returns: 200 OK if valid, 401 if expired or tampered
```

### 4.3 Adding Custom Claims to Tokens

By default tokens only contain `user_id`. You can add extra data (e.g., `username`, `role`, `email`) so the frontend doesn't need extra API calls.

```python
# serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims — these appear in EVERY access token payload
        token['username'] = user.username
        token['email'] = user.email
        token['is_staff'] = user.is_staff
        # If you have a profile model:
        # token['role'] = user.profile.role

        return token
```

```python
# views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import MyTokenObtainPairSerializer

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

```python
# urls.py
urlpatterns = [
    path('api/auth/token/', MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
    ...
]
```

```python
# settings.py  — tell simplejwt to use your serializer globally
SIMPLE_JWT = {
    ...
    'TOKEN_OBTAIN_SERIALIZER': 'myapp.serializers.MyTokenObtainPairSerializer',
}
```

**Resulting token payload:**
```json
{
  "token_type": "access",
  "exp": 1700000900,
  "jti": "abc123",
  "user_id": 42,
  "username": "alice",
  "email": "alice@example.com",
  "is_staff": false
}
```

### 4.4 Customizing the Login Response

Return extra fields (like user info) alongside the tokens:

```python
# serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    def validate(self, attrs):
        data = super().validate(attrs)   # adds 'access' and 'refresh'

        # Append extra user info to the response body
        data['user'] = {
            'id': self.user.id,
            'username': self.user.username,
            'email': self.user.email,
            'is_staff': self.user.is_staff,
        }
        return data

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

**Response body:**
```json
{
  "access": "eyJ...",
  "refresh": "eyJ...",
  "user": {
    "id": 42,
    "username": "alice",
    "email": "alice@example.com",
    "is_staff": false
  }
}
```

### 4.5 Rotating Refresh Tokens in Practice

When `ROTATE_REFRESH_TOKENS = True`, calling `/token/refresh/` returns a **new** refresh token:

```
Client                          Server
  │                               │
  │── POST /token/refresh/ ──────>│  { "refresh": "<old_token>" }
  │<─ 200 OK ─────────────────────│  { "access": "<new_access>", "refresh": "<new_refresh>" }
  │                               │
  │  (old refresh token is now    │
  │   invalid/blacklisted)        │
```

> 🔐 This limits the window for refresh token theft — each token can only be used once.

---

## 5. JWT Blacklisting (Logout)

### 5.1 Setup

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt.token_blacklist',
]
```

```bash
python manage.py migrate  # creates outstanding_token and blacklisted_token tables
```

### 5.2 Logout View

```python
# views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError

class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        try:
            refresh_token = request.data["refresh"]
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response({"detail": "Successfully logged out."}, status=status.HTTP_200_OK)
        except KeyError:
            return Response({"detail": "Refresh token is required."}, status=status.HTTP_400_BAD_REQUEST)
        except TokenError:
            return Response({"detail": "Token is invalid or expired."}, status=status.HTTP_400_BAD_REQUEST)
```

### 5.3 Blacklist All Tokens for a User

Useful for "log out from all devices" functionality:

```python
from rest_framework_simplejwt.token_blacklist.models import OutstandingToken, BlacklistedToken

class LogoutAllView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        tokens = OutstandingToken.objects.filter(user=request.user)
        for token in tokens:
            BlacklistedToken.objects.get_or_create(token=token)
        return Response({"detail": "All sessions terminated."}, status=status.HTTP_200_OK)
```

---

## 6. Protecting Views with JWT

### 6.1 Global Authentication

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

### 6.2 Reading the Authenticated User

```python
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        user = self.request.user
        if user.is_staff:
            return Post.objects.all()
        return Post.objects.filter(author=user)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

### 6.3 Accessing Custom Claims in a View

If you added custom claims to the token, you can read them from the raw token payload:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class MeView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        # Standard user object from DB
        user = request.user

        # Raw JWT payload (already validated by JWTAuthentication)
        token_payload = request.auth.payload  # dict with all claims

        return Response({
            "id": user.id,
            "username": user.username,
            "token_expires": token_payload.get("exp"),
            "is_staff_from_token": token_payload.get("is_staff"),
        })
```

---

## 7. Permission Classes

### Built-in permissions:
| Class | Behavior |
|-------|---------|
| `AllowAny` | Anyone can access (public) |
| `IsAuthenticated` | Must be logged in |
| `IsAdminUser` | Must be `is_staff=True` |
| `IsAuthenticatedOrReadOnly` | Read-only for anonymous, full access for authenticated |

```python
# Global default in settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ]
}

# Override per view
from rest_framework.permissions import IsAuthenticated, AllowAny

class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

class PublicPostListView(generics.ListAPIView):
    permission_classes = [AllowAny]
```

---

## 8. Per-Action Permissions

```python
from rest_framework.permissions import IsAuthenticated, IsAdminUser, AllowAny

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_permissions(self):
        if self.action in ['list', 'retrieve']:
            return [AllowAny()]
        elif self.action in ['create']:
            return [IsAuthenticated()]
        else:  # update, destroy
            return [IsAdminUser()]
```

---

## 9. Custom Permission Classes

```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission: only the owner of an object can edit/delete it.
    """
    message = "You do not have permission to modify this resource."

    def has_permission(self, request, view):
        # Allow anyone to read; require auth for write
        if request.method in permissions.SAFE_METHODS:  # GET, HEAD, OPTIONS
            return True
        return request.user.is_authenticated

    def has_object_permission(self, request, view, obj):
        # Safe methods are always allowed
        if request.method in permissions.SAFE_METHODS:
            return True
        # Write methods require the request user to be the owner
        return obj.author == request.user


class IsVerifiedUser(permissions.BasePermission):
    """Only allow users who have verified their email."""
    message = "Email verification required."

    def has_permission(self, request, view):
        return (
            request.user.is_authenticated and
            request.user.profile.email_verified
        )
```

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsOwnerOrReadOnly]
```

---

## 10. Custom User Registration + JWT Login Flow

```python
# views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import AllowAny
from rest_framework_simplejwt.tokens import RefreshToken
from .serializers import RegisterSerializer

class RegisterView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        serializer = RegisterSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            # Generate JWT tokens immediately on registration
            refresh = RefreshToken.for_user(user)
            return Response({
                'user': {'id': user.id, 'username': user.username},
                'tokens': {
                    'refresh': str(refresh),
                    'access': str(refresh.access_token),
                }
            }, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

---

## 11. Integrating drf-spectacular (Swagger/OpenAPI)

`drf-spectacular` generates an OpenAPI 3 schema from your DRF views and supports JWT security out of the box.

### 11.1 Installation

```bash
pip install drf-spectacular
```

### 11.2 Configuration

```python
# settings.py
INSTALLED_APPS = [
    ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    ...
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'DESCRIPTION': 'Full-featured Django REST Framework API',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,

    # Tell spectacular that the API uses JWT (Bearer tokens)
    'SECURITY': [{'jwtAuth': []}],
    'COMPONENTS': {
        'securitySchemes': {
            'jwtAuth': {
                'type': 'http',
                'scheme': 'bearer',
                'bearerFormat': 'JWT',
            }
        }
    },
}
```

### 11.3 URL Setup

```python
# urls.py
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    # Auth endpoints
    path('api/auth/token/',         MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(),      name='token_refresh'),
    path('api/auth/token/verify/',  TokenVerifyView.as_view(),       name='token_verify'),
    path('api/auth/logout/',        LogoutView.as_view(),            name='logout'),
    path('api/auth/register/',      RegisterView.as_view(),          name='register'),

    # Schema & Docs
    path('api/schema/',             SpectacularAPIView.as_view(),    name='schema'),
    path('api/docs/',               SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/redoc/',              SpectacularRedocView.as_view(url_name='schema'),   name='redoc'),
]
```

### 11.4 Annotating Views with @extend_schema

Use `@extend_schema` to add descriptions, response examples, and control what appears in Swagger:

```python
from drf_spectacular.utils import extend_schema, OpenApiExample, OpenApiResponse
from rest_framework_simplejwt.views import TokenObtainPairView

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer

    @extend_schema(
        summary="Obtain JWT Token Pair",
        description="Send username and password to receive an access token (15 min) and a refresh token (7 days).",
        responses={
            200: OpenApiResponse(
                description="Tokens issued successfully",
                examples=[
                    OpenApiExample(
                        "Success",
                        value={
                            "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
                            "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
                            "user": {"id": 1, "username": "alice", "email": "alice@example.com"}
                        }
                    )
                ]
            ),
            401: OpenApiResponse(description="Invalid credentials"),
        },
        tags=["Authentication"],
    )
    def post(self, request, *args, **kwargs):
        return super().post(request, *args, **kwargs)
```

```python
class RegisterView(APIView):
    permission_classes = [AllowAny]

    @extend_schema(
        summary="Register a new user",
        request=RegisterSerializer,
        responses={201: CustomTokenObtainPairSerializer},
        tags=["Authentication"],
    )
    def post(self, request):
        ...
```

```python
class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    @extend_schema(
        summary="Logout (blacklist refresh token)",
        request={"application/json": {"type": "object", "properties": {"refresh": {"type": "string"}}}},
        responses={200: OpenApiResponse(description="Logged out successfully")},
        tags=["Authentication"],
    )
    def post(self, request):
        ...
```

### 11.5 Tagging ViewSets

```python
from drf_spectacular.utils import extend_schema_view, extend_schema

@extend_schema_view(
    list=extend_schema(summary="List all posts", tags=["Posts"]),
    create=extend_schema(summary="Create a post", tags=["Posts"]),
    retrieve=extend_schema(summary="Get a single post", tags=["Posts"]),
    update=extend_schema(summary="Update a post", tags=["Posts"]),
    destroy=extend_schema(summary="Delete a post", tags=["Posts"]),
)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsOwnerOrReadOnly]
```

### 11.6 How to Authenticate in Swagger UI

1. Navigate to `http://localhost:8000/api/docs/`
2. Click the **Authorize 🔓** button (top right)
3. In the `jwtAuth (http, Bearer)` field, enter your access token:
   ```
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   ```
4. Click **Authorize** → all subsequent requests will include `Authorization: Bearer <token>`

### 11.7 Generating the Schema File

You can export the OpenAPI schema as a YAML or JSON file (useful for sharing with frontend teams):

```bash
python manage.py spectacular --color --file schema.yml
```

---

## 12. Complete Auth Setup Checklist

```
✅  pip install djangorestframework-simplejwt drf-spectacular
✅  Add to INSTALLED_APPS:
      - rest_framework_simplejwt
      - rest_framework_simplejwt.token_blacklist
      - drf_spectacular
✅  Run: python manage.py migrate
✅  Configure SIMPLE_JWT settings (lifetimes, rotation, blacklisting)
✅  Set DEFAULT_SCHEMA_CLASS = 'drf_spectacular.openapi.AutoSchema'
✅  Configure SPECTACULAR_SETTINGS with JWT security scheme
✅  Add token endpoints + schema/docs endpoints to urls.py
✅  Use @extend_schema on views for clean Swagger documentation
✅  Test via Swagger UI at /api/docs/
```

---

## 📝 Key Takeaways
1. **Authentication** = identity (who you are); **Permission** = access (what you can do)
2. JWT has three parts: **Header** · **Payload** · **Signature** — the payload is readable, never store secrets in it
3. JWT is preferred for modern APIs — stateless and scalable
4. Use **custom serializers** to add extra claims (username, role) so the frontend doesn't need extra API calls
5. **Rotate refresh tokens** (`ROTATE_REFRESH_TOKENS = True`) to limit the impact of token theft
6. **Blacklist tokens** on logout — don't just delete them client-side
7. Always use `has_object_permission` for ownership checks — `has_permission` is for the view level
8. `SAFE_METHODS = ['GET', 'HEAD', 'OPTIONS']` — reads are usually safe to allow widely
9. `get_permissions()` lets you set different permissions per ViewSet action
10. **drf-spectacular** auto-generates OpenAPI docs and lets you test JWT-protected endpoints directly in Swagger UI
11. Use `@extend_schema` to add summaries, examples, and tags to keep your docs clean and professional
