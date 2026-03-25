# Lesson 4 — Homework

## 📌 Assignment: Blog API with Full URL Structure

### Goal
Build a complete Blog API with properly structured URLs using routers and nested routers.

### URL Structure to Implement:
```
/api/categories/                              → list/create categories
/api/categories/<id>/                         → retrieve/update/delete category
/api/posts/                                   → list/create posts
/api/posts/<id>/                              → retrieve/update/delete post
/api/posts/<post_pk>/comments/                → list/create comments for a post
/api/posts/<post_pk>/comments/<id>/           → retrieve/update/delete comment
/api/posts/<id>/like/                         → POST: toggle like on a post
/api/posts/<id>/bookmark/                     → POST: toggle bookmark for current user
/api/posts/feed/                              → GET: posts by authors the user follows
```

### Requirements:
1. Use `DefaultRouter` for top-level resources
2. Use `drf-nested-routers` for comments under posts
3. All ViewSets must use `get_queryset()` appropriately
4. The `feed` endpoint only returns posts (not soft-deleted)
5. Liking a post is idempotent — liking twice unlikes it

### Models Needed:
```python
class Like(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    class Meta:
        unique_together = ['user', 'post']

class Bookmark(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    class Meta:
        unique_together = ['user', 'post']
```

### Bonus:
- Add `GET /api/posts/<id>/likes/` that returns the list of users who liked the post
- Ensure comments can only be edited or deleted by the comment author

### Submission Checklist:
- [ ] All URLs are implemented using routers (no manual `path()` for ViewSet routes)
- [ ] Nested comments route works correctly
- [ ] Custom actions (`like`, `bookmark`, `feed`) work correctly
- [ ] Like toggle (idempotent) is implemented correctly
- [ ] Code is organized in separate files (models, serializers, views, urls)

