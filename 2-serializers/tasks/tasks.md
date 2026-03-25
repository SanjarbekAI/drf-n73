# Lesson 2 — In-Class Tasks

## Task 1: ModelSerializer Deep Dive (15 min)
Given this model:
```python
class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published = models.BooleanField(default=False)
    views_count = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

Create a serializer where:
- `id`, `created_at`, `updated_at` are read-only
- `views_count` is read-only (clients can't set view count)
- `author` is write-only (don't expose it in output)
- Add a computed field `reading_time` that estimates minutes to read (assume 200 words per minute)

---

## Task 2: Validation (20 min)
Add these validations to your Article serializer:
1. `title` must be between 10 and 200 characters
2. `title` cannot start with a number
3. `body` must be at least 50 words long
4. If `published=True`, the body must be at least 100 words long (object-level validation)

Test each validation rule by sending invalid data.

---

## Task 3: Nested Serializer (25 min)
Add a `Tag` model:
```python
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

class Article(models.Model):
    # ... existing fields
    tags = models.ManyToManyField(Tag, blank=True)
```

Create serializers so that:
- `GET /api/articles/` returns each article with its tags as a list of tag names
- `POST /api/articles/` accepts a list of tag IDs to associate

---

## Task 4: Debug Challenge
The following serializer has 3 bugs. Find and fix them:
```python
class BuggySerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()
    
    # Bug 1
    def get_full_name(self, obj):
        return obj.first_name + obj.last_name
    
    def validate_email(self, value):
        if '@' not in value:
            raise serializers.ValidationError("Invalid email")
        # Bug 2: missing return
    
    def validate(self, data):
        if data['start'] > data['end']:
            raise serializers.ValidationError("Start must be before end")
        # Bug 3: missing return
    
    class Meta:
        model = User
        fields = ['id', 'first_name', 'last_name', 'full_name', 'email']
```

