# Lesson 11 — In-Class Tasks

## Task 1: Signals — Auto Profile & Token (15 min)
Create signals so that when a new `User` is created:
1. A `Token` is automatically created
2. A `UserProfile` is automatically created (add a basic `Profile` model)
3. A `send_welcome_email` Celery task is queued

Test: Create a user through the admin panel. Verify the token and profile exist without any manual code.

---

## Task 2: Post Published Signal (15 min)
Create a custom signal `post_published` that fires when `post.published` changes from `False` to `True`.

To detect the change, compare the old value:
```python
@receiver(pre_save, sender=Post)
def track_publish_state(sender, instance, **kwargs):
    if instance.pk:
        try:
            old = Post.objects.get(pk=instance.pk)
            instance._was_published = old.published
        except Post.DoesNotExist:
            instance._was_published = False
    else:
        instance._was_published = False

@receiver(post_save, sender=Post)
def on_post_saved(sender, instance, created, **kwargs):
    if not created and instance.published and not instance._was_published:
        post_published.send(sender=Post, instance=instance)
```

---

## Task 3: Request Logging Middleware (20 min)
Implement `RequestLoggingMiddleware` that logs:
```
[a1b2c3d4] GET /api/posts/ → 200 (45.2ms) user=john
[e5f6g7h8] POST /api/auth/login/ → 401 (12.1ms) user=anonymous
```

Add a `correlation_id` header to every response so clients can reference it in support tickets:
```
X-Correlation-ID: a1b2c3d4
```

---

## Task 4: Fix N+1 Problems (25 min)
Given this ViewSet with a nested serializer:
```python
class PostSerializer(serializers.ModelSerializer):
    author_username = serializers.CharField(source='author.username')
    category_name = serializers.CharField(source='category.name')
    tags = TagSerializer(many=True)
    comment_count = serializers.SerializerMethodField()

    def get_comment_count(self, obj):
        return obj.comments.count()   # ← N+1 problem
```

1. Install `django-debug-toolbar` and open `/api/posts/` — count the queries
2. Add `select_related('author', 'category')`, `prefetch_related('tags')`, and `annotate(comment_count=Count('comments'))` to the queryset
3. Count queries again. How many did you save?

---

## Task 5: annotate() in Action (15 min)
Add these annotated fields to the Post queryset without any Python-level aggregation:
- `comment_count` — number of comments
- `like_count` — number of likes
- `avg_rating` — average review rating

Use them in the serializer as `IntegerField(read_only=True)`.

Confirm with debug toolbar that it's all done in ONE query.

