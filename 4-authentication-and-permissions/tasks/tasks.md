# Lesson 4 — In-Class Tasks

## Task 1: Token Authentication Setup (20 min)
1. Install and configure `rest_framework.authtoken`
2. Create a `POST /api/auth/login/` endpoint using `obtain_auth_token`
3. Make the `PostViewSet` require Token authentication
4. Test: get a token by posting credentials, then use it to access the post list

---

## Task 2: Switch to JWT (20 min)
1. Install `djangorestframework-simplejwt`
2. Replace Token auth with JWT in `settings.py`
3. Add `/api/auth/token/` and `/api/auth/token/refresh/` endpoints
4. Test the full flow:
   - Get access + refresh tokens
   - Use access token to call a protected endpoint
   - Refresh when the access token expires
   - What happens when you use an expired token?

---

## Task 3: Custom Permission — IsOwnerOrReadOnly (20 min)
Implement `IsOwnerOrReadOnly` permission class.

Apply it to the Post ViewSet so that:
- Anyone (even anonymous) can read posts
- Only authenticated users can create posts
- Only the post author can edit or delete their own post

**Test:**
- Can user A delete user B's post? (Should get 403)
- Can anonymous user read posts? (Should get 200)
- Can authenticated user create a post? (Should get 201)

---

## Task 4: Registration + JWT Response (20 min)
Build a `POST /api/auth/register/` endpoint that:
- Creates a new user
- Immediately returns JWT access and refresh tokens
- Returns user info (id, username, email) in the response

---

## Task 5: Discussion — Authentication Strategies
When would you use:
1. Session Authentication?
2. Token Authentication?
3. JWT Authentication?

What are the trade-offs of each? (Think: mobile apps vs web apps, microservices vs monolith)

