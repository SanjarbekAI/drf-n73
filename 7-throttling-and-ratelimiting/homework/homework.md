# Lesson 7 — Homework

## 📌 Assignment: Production-Grade Throttling System

### Part 1: Tiered User Throttling
Implement three throttle tiers:

| Tier | Rate | Condition |
|------|------|-----------|
| `FreeUserThrottle` | 1000/day | Authenticated, not premium |
| `PremiumUserThrottle` | 10000/day | `profile.is_premium == True` |
| `AnonRateThrottle` | 100/day | Not authenticated |

Apply them globally. The first throttle class that returns a non-None cache key wins.

### Part 2: Endpoint-Specific Limits
Set these specific throttle scopes:

| Endpoint | Scope | Limit |
|----------|-------|-------|
| `POST /api/auth/login/` | `login` | 5/min |
| `POST /api/auth/register/` | `register` | 3/hour |
| `POST /api/auth/password-reset/` | `password_reset` | 3/hour |
| `POST /api/posts/<id>/like/` | `like_action` | 60/min |
| `POST /api/posts/` | `create_post` | 10/hour |
| `GET /api/products/` | `public_browse` | 300/hour |

### Part 3: Custom Exception Handler
Register a global custom exception handler in `settings.py` that:
- For `Throttled` exceptions: returns `429` with `retry_after_seconds` and human-readable message
- For `NotAuthenticated`: returns `401` with `{"error": "Authentication required", "login_url": "/api/auth/login/"}`
- For `PermissionDenied`: returns `403` with `{"error": "Permission denied", "detail": "..."}`
- For all other exceptions: falls back to default DRF behavior

```python
# exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import Throttled, NotAuthenticated, PermissionDenied
from rest_framework.response import Response
from rest_framework import status

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if isinstance(exc, Throttled):
        ...
    elif isinstance(exc, NotAuthenticated):
        ...
    elif isinstance(exc, PermissionDenied):
        ...
    return response
```

```python
# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'myapp.exceptions.custom_exception_handler',
}
```

### Part 4: Redis Throttle Storage
Configure Redis as the cache backend and verify:
- Throttle state survives server restart
- Throttle state is shared across two Django processes running simultaneously (simulate with two terminals)

### Bonus: Throttle Bypass for Internal Services
Create a `ServiceAccountThrottle` that:
- Checks for a special `X-Internal-Service-Key` header
- If the key matches a value in `settings.INTERNAL_SERVICE_KEY` → no throttling
- Otherwise → applies normal throttling

### Submission Checklist:
- [ ] Three-tier throttling works correctly
- [ ] Endpoint-specific scopes are applied
- [ ] Custom exception handler returns proper structured errors
- [ ] Redis is configured and throttle state persists
- [ ] All throttle rate values are defined in `settings.py` (not hardcoded)

