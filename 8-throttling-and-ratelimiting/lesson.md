# Lesson 8: Throttling & Rate Limiting

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain why rate limiting is necessary
- Configure built-in DRF throttle classes
- Write custom throttle classes
- Apply different rate limits per user type and per endpoint
- Test throttling behavior

---

## 1. Why Throttling?

Without rate limiting your API is vulnerable to:
- **DDoS attacks** — flood of requests overwhelms your server
- **Brute force attacks** — try thousands of passwords
- **Web scraping** — competitor downloads your entire product catalog
- **Accidental loops** — client bug sends 10,000 requests per second

Rate limiting = "You can make X requests per Y time period"

---

## 2. Built-in Throttle Classes

| Class | Identifies by | Use case |
|-------|--------------|---------|
| `AnonRateThrottle` | IP address | Limit unauthenticated users |
| `UserRateThrottle` | User ID | Limit authenticated users |
| `ScopedRateThrottle` | Named scope | Different limits per endpoint |

---

## 3. Global Configuration

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',      # unauthenticated: 100 requests/day
        'user': '1000/day',     # authenticated: 1000 requests/day
    }
}
```

### Rate format:
```
'100/second'
'100/minute'
'100/hour'
'100/day'
```

---

## 4. Per-View Throttling Override

```python
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'  # matches 'burst' key in THROTTLE_RATES

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/minute',
        'sustained': '1000/day',
    }
}

# views.py
class PostViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```

---

## 5. ScopedRateThrottle — Per-Endpoint Limits

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'login': '5/minute',
        'register': '3/hour',
        'upload': '10/hour',
        'public': '200/hour',
    }
}

# views.py
class LoginView(APIView):
    throttle_scope = 'login'
    # Max 5 login attempts per minute per IP

class RegisterView(APIView):
    throttle_scope = 'register'
    # Max 3 registrations per hour

class FileUploadView(APIView):
    throttle_scope = 'upload'
    # Max 10 uploads per hour
```

---

## 6. Custom Throttle Class

```python
# throttles.py
from rest_framework.throttling import SimpleRateThrottle

class PremiumUserThrottle(SimpleRateThrottle):
    """Higher rate limit for premium users."""
    scope = 'premium'

    def get_cache_key(self, request, view):
        if not request.user.is_authenticated:
            return None  # fall through to another throttle class
        if not request.user.profile.is_premium:
            return None  # not premium — use default throttle
        return f'throttle_premium_{request.user.id}'

class IPRateThrottle(SimpleRateThrottle):
    """Strict per-IP throttle for sensitive endpoints."""
    scope = 'ip_strict'

    def get_cache_key(self, request, view):
        ident = self.get_ident(request)  # gets client IP
        return f'throttle_ip_{ident}'
```

```python
# settings.py
'DEFAULT_THROTTLE_RATES': {
    'premium': '5000/day',
    'ip_strict': '10/minute',
}

# views.py — apply to sensitive endpoint
class PasswordResetView(APIView):
    throttle_classes = [IPRateThrottle]
```

---

## 7. Throttle Response Headers

When a request is throttled, DRF returns:
```
HTTP 429 Too Many Requests
Retry-After: 3600
```

You can inspect the response headers to tell clients when to retry:
```python
# Custom throttle with informative response
from rest_framework.exceptions import Throttled

class MyView(APIView):
    def handle_exception(self, exc):
        if isinstance(exc, Throttled):
            wait = exc.wait
            return Response(
                {
                    "error": "Rate limit exceeded.",
                    "retry_after_seconds": int(wait),
                    "message": f"Please wait {int(wait)} seconds before trying again."
                },
                status=status.HTTP_429_TOO_MANY_REQUESTS
            )
        return super().handle_exception(exc)
```

---

## 8. Throttling Storage — Redis Backend (Production)

By default, throttle state is stored in Django's cache (memory-based — resets on restart).

For production, use Redis:
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}
```

```bash
pip install django-redis
```

Now throttle state persists across server restarts and works across multiple server instances.

---

## 9. Practical Throttle Strategy

```python
# Good throttle strategy for a production API:

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '200/day',       # public browsing
        'user': '5000/day',      # authenticated general use
        'login': '10/minute',    # brute force protection
        'register': '5/hour',    # prevent mass account creation
        'password_reset': '3/hour',  # prevent enumeration
        'upload': '20/hour',     # resource-intensive operation
    }
}
```

---

## 📝 Key Takeaways
1. Rate limiting protects against abuse, DDoS, and accidental runaway clients
2. Use `AnonRateThrottle` for IP-based limits, `UserRateThrottle` for user-based
3. `ScopedRateThrottle` + `throttle_scope` gives different limits per endpoint
4. Use Redis for throttle storage in production (not in-memory)
5. Always return `429 Too Many Requests` with a `Retry-After` header
6. Sensitive endpoints (login, register, password reset) need much stricter limits

