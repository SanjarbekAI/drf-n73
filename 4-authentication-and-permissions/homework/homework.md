# Lesson 4 — Homework

## 📌 Assignment: Full Auth System for the Blog API

### Part 1: Complete Auth Endpoints
Implement all auth endpoints:

| Method | URL | Description | Auth Required |
|--------|-----|-------------|--------------|
| POST | `/api/auth/register/` | Register + return JWT tokens | No |
| POST | `/api/auth/login/` | Login + return JWT tokens | No |
| POST | `/api/auth/logout/` | Blacklist refresh token | Yes |
| POST | `/api/auth/token/refresh/` | Get new access token | No (needs refresh token) |
| GET | `/api/auth/me/` | Get current user profile | Yes |
| PATCH | `/api/auth/me/` | Update current user profile | Yes |

### Part 2: Permission Levels for Post API
Implement these access rules:
- **Anonymous**: Can only `GET` (list, retrieve)
- **Authenticated user**: Can `GET`, `POST` (create)
- **Post owner**: Can `GET`, `PUT`, `PATCH`, `DELETE` on their own posts
- **Admin/Staff**: Can do everything including delete any post

### Part 3: Custom Permission — IsPremiumUser
Create a `IsPremiumUser` permission class:
- Check if `request.user.profile.is_premium == True`
- Return proper error message: `"This feature requires a Premium account."`
- Apply it to a `POST /api/posts/<id>/feature/` action (which marks a post as "featured")

### Part 4: Refresh Token Security
Add to `RegisterSerializer` validation:
- Password must be at least 8 characters
- Must contain at least one uppercase letter
- Must contain at least one digit
- Must contain at least one special character (`!@#$%^&*`)

### Bonus — Change Password Endpoint:
```
POST /api/auth/change-password/
Body: { "old_password": "...", "new_password": "...", "confirm_password": "..." }
```
- Verify old password is correct
- Apply the same password strength rules
- Invalidate old tokens after password change (blacklist all refresh tokens)

### Submission Checklist:
- [ ] All 6 auth endpoints work
- [ ] JWT tokens are returned on register and login
- [ ] Logout blacklists the refresh token
- [ ] Permission levels are correctly enforced
- [ ] Password strength validation is in place
- [ ] `IsPremiumUser` permission class is implemented

