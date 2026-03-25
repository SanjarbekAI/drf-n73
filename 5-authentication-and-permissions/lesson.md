# Lesson 5: Authentication & Permissions

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain the difference between Authentication and Authorization
- Implement Session, Token, and JWT authentication
- Use built-in DRF permission classes
- Write custom permission classes
- Apply object-level permissions
- Control permissions per ViewSet action

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

## 3. Permission Classes

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

## 4. Per-Action Permissions

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

## 5. Custom Permission Classes

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

## 6. Custom User Registration + JWT Login Flow

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

## 7. Blacklisting JWT Tokens (Logout)

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt.token_blacklist',
]
```

```python
# views.py
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework.permissions import IsAuthenticated

class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        try:
            refresh_token = request.data["refresh"]
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response({"detail": "Successfully logged out."}, status=status.HTTP_200_OK)
        except Exception:
            return Response(status=status.HTTP_400_BAD_REQUEST)
```

---

## 8. `request.user` in Views

```python
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        user = self.request.user
        if user.is_staff:
            return Post.objects.all()  # admins see everything
        return Post.objects.filter(author=user)  # users see only their posts

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

---

## 📝 Key Takeaways
1. **Authentication** = identity (who you are); **Permission** = access (what you can do)
2. JWT is preferred for modern APIs — stateless and scalable
3. Always use `has_object_permission` for ownership checks — `has_permission` is for the view level
4. `SAFE_METHODS = ['GET', 'HEAD', 'OPTIONS']` — reads are usually safe to allow widely
5. `get_permissions()` lets you set different permissions per ViewSet action
6. Never hardcode tokens — use `Authorization: Bearer <token>` headers

