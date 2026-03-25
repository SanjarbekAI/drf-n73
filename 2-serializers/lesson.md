# Lesson 2: Serializers — The Heart of DRF

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Understand why serializers exist and what they do
- Use `ModelSerializer` effectively with all field types
- Write custom validation (field-level and object-level)
- Build nested serializers for related data
- Use `SerializerMethodField` for computed/read-only fields
- Control what fields are readable vs writable

---

## 1. What is a Serializer?

A **serializer** does two jobs:
1. **Serialization** — Python object → JSON (for responses)
2. **Deserialization** — JSON → Python object (for requests, with validation)

```
Python Object (QuerySet / Model instance)
        ↕  serializer
        JSON
```

Without serializers you'd have to manually convert every model field to/from JSON — very tedious and error-prone.

---

## 2. Base Serializer vs ModelSerializer

### Base `Serializer` (manual — useful for non-model data):
```python
from rest_framework import serializers

class LoginSerializer(serializers.Serializer):
    username = serializers.CharField()
    password = serializers.CharField(write_only=True)
```

### `ModelSerializer` (auto-generates fields from the model):
```python
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'         # all fields
        # fields = ['id', 'title'] # specific fields
        # exclude = ['created_at'] # all except these
```

---

## 3. Field Options

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at']
        read_only_fields = ['id', 'created_at']   # cannot be set by client
        extra_kwargs = {
            'content': {'min_length': 10},
            'author': {'write_only': True},        # only accepted on input, not returned
        }
```

### Common field arguments:
| Argument | Purpose |
|----------|---------|
| `read_only=True` | Only included in output, ignored in input |
| `write_only=True` | Only accepted in input, not in output |
| `required=False` | Field is optional |
| `allow_null=True` | Field can be null/None |
| `source='field_name'` | Map to a different model field name |

---

## 4. Custom Field-Level Validation

Prefix method with `validate_<fieldname>`:

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters long.")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Title cannot contain 'spam'.")
        return value  # always return the value!
```

---

## 5. Object-Level Validation (Cross-field)

Use `validate()` for validations that involve multiple fields:

```python
class EventSerializer(serializers.ModelSerializer):
    class Meta:
        model = Event
        fields = ['title', 'start_date', 'end_date']

    def validate(self, data):
        if data['start_date'] >= data['end_date']:
            raise serializers.ValidationError("End date must be after start date.")
        return data
```

---

## 6. SerializerMethodField — Computed Fields

```python
class PostSerializer(serializers.ModelSerializer):
    word_count = serializers.SerializerMethodField()
    is_long_post = serializers.SerializerMethodField()

    def get_word_count(self, obj):
        return len(obj.content.split())

    def get_is_long_post(self, obj):
        return len(obj.content.split()) > 500

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'word_count', 'is_long_post']
```

---

## 7. Nested Serializers — Related Data

### Forward relationship (ForeignKey):
```python
# models.py
class Category(models.Model):
    name = models.CharField(max_length=100)

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='posts')
```

```python
# serializers.py
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class PostSerializer(serializers.ModelSerializer):
    # Nested read — shows full category object
    category = CategorySerializer(read_only=True)
    # Write — accepts category id
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(), source='category', write_only=True
    )

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'category', 'category_id']
```

### Reverse relationship (related objects):
```python
class CategorySerializer(serializers.ModelSerializer):
    posts = PostSerializer(many=True, read_only=True)

    class Meta:
        model = Category
        fields = ['id', 'name', 'posts']
```

---

## 8. Overriding `create()` and `update()`

Use when the default behavior isn't enough:

```python
class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)
    confirm_password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'confirm_password']

    def validate(self, data):
        if data['password'] != data['confirm_password']:
            raise serializers.ValidationError("Passwords do not match.")
        return data

    def create(self, validated_data):
        validated_data.pop('confirm_password')
        # Use create_user to properly hash the password
        user = User.objects.create_user(**validated_data)
        return user
```

---

## 9. `source` Argument — Rename Fields

```python
class PostSerializer(serializers.ModelSerializer):
    post_title = serializers.CharField(source='title')      # rename field
    author_name = serializers.CharField(source='author.username')  # traverse FK

    class Meta:
        model = Post
        fields = ['id', 'post_title', 'author_name']
```

---

## 10. Real World Pattern: Separate Read/Write Serializers

```python
# For writing (input) — minimal, strict
class PostWriteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category']

# For reading (output) — rich, nested
class PostReadSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    word_count = serializers.SerializerMethodField()

    def get_word_count(self, obj):
        return len(obj.content.split())

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'category', 'word_count', 'created_at']

# In the view:
def get_serializer_class(self):
    if self.request.method in ['POST', 'PUT', 'PATCH']:
        return PostWriteSerializer
    return PostReadSerializer
```

---

## 📝 Key Takeaways
1. Serializers handle both validation (input) and representation (output)
2. `ModelSerializer` auto-generates fields — saves a lot of boilerplate
3. Always return the `value` at the end of `validate_<field>` methods
4. Use `read_only` and `write_only` to control data flow direction
5. Nested serializers are powerful but can cause N+1 query problems — always use `select_related`/`prefetch_related`

