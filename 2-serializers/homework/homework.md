# Lesson 2 — Homework

## 📌 Assignment: User Registration & Profile Serializer

### Part 1: Registration Serializer
Build a `RegisterSerializer` that:
- Accepts: `username`, `email`, `password`, `confirm_password`, `first_name`, `last_name`
- Validates that `password` and `confirm_password` match
- Validates that `email` is unique (check the database)
- Validates password strength: min 8 chars, at least 1 number
- On `create()`, hashes the password properly using `create_user()`
- Does NOT return `password` or `confirm_password` in the response

### Part 2: Profile Serializer
Build a `UserProfileSerializer` that:
- Shows: `id`, `username`, `email`, `first_name`, `last_name`, `date_joined`
- Adds a computed field `full_name` = first_name + " " + last_name
- Adds a computed field `member_since_days` = days since `date_joined`
- `email` is read-only (cannot be changed through this serializer)
- `username` is read-only

### Part 3: Nested Profile in a Post
Create a `PostSerializer` where:
- The author field shows a nested `UserProfileSerializer` (read-only)
- When creating a post, the author is automatically set to `request.user` (hint: use `create()` override or `HiddenField`)

### Bonus:
Add a validator that prevents two posts with the same title by the same author:
```python
def validate(self, data):
    # check if Post.objects.filter(title=..., author=...).exists()
    ...
```

### Submission Checklist:
- [ ] Registration with password hashing works
- [ ] All validations fire correctly with proper error messages
- [ ] Profile serializer computed fields are correct
- [ ] Nested author in Post serializer works
- [ ] No sensitive data (passwords) leaks in any response

