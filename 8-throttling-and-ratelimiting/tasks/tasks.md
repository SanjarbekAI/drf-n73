# Lesson 8 — In-Class Tasks

## Task 1: Global Throttling Setup (15 min)
Configure global throttling in `settings.py`:
- Anonymous users: 50 requests/day
- Authenticated users: 500 requests/day

Test throttling by making repeated requests. What response code do you get when the limit is hit?

---

## Task 2: ScopedRateThrottle (20 min)
Apply different rate limits to these endpoints using `throttle_scope`:
- `POST /api/auth/login/` → 5 per minute
- `POST /api/auth/register/` → 3 per hour
- `POST /api/auth/password-reset/` → 3 per hour
- `GET /api/products/` → 200 per hour (public browsing)

Test login throttle by sending 6 rapid login requests and confirm the 6th returns `429`.

---

## Task 3: Custom Throttle — PremiumUserThrottle (20 min)
Implement `PremiumUserThrottle`:
- Premium users (check `request.user.profile.is_premium`) → 5000/day
- Non-premium users → use the default 500/day
- Unauthenticated → fall through to `AnonRateThrottle`

Add a `profile` model with an `is_premium` field if not already present.

---

## Task 4: Custom 429 Error Response (15 min)
Override the default 429 response to return:
```json
{
    "error": "Rate limit exceeded",
    "retry_after_seconds": 120,
    "message": "You have made too many requests. Please wait 2 minutes."
}
```

Hint: Override `handle_exception()` in your view or use a custom exception handler in settings.

---

## Task 5: Redis Cache Backend (10 min)
Install `django-redis` and configure the cache backend.
Verify throttle state now persists by:
1. Hitting the rate limit
2. Restarting the Django server
3. Confirming you're still throttled (state was saved in Redis, not memory)

