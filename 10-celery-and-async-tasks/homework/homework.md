# Lesson 10 — Homework

## 📌 Assignment: Full Async Feature Set

### Part 1: Email Notification System
Implement async email notifications for these events:
- User registers → welcome email
- Post receives a comment → notify post author
- Someone follows a user → notify the followed user
- Weekly digest → send every Monday 8:00 AM with user's top 5 unread posts

Use `django.core.mail.backends.console.EmailBackend` for development (prints to console).

Each task should:
- Accept only primitive arguments (IDs, not model instances)
- Have `max_retries=3` with exponential backoff: `countdown=2 ** self.request.retries * 60`
- Log success/failure to the console

### Part 2: Image Processing Pipeline
When a user uploads a profile picture via `PATCH /api/users/me/`:
1. Save the original image
2. Queue a `process_profile_image` task that:
   - Creates a 200x200 thumbnail
   - Creates a 50x50 avatar
   - Saves both versions
   - Updates the user profile with the new image paths
3. Return an immediate response with `{"status": "processing", "task_id": "..."}`

Add a `GET /api/users/me/image-status/<task_id>/` endpoint to check progress.

### Part 3: Bulk Operation Task
Implement `POST /api/admin/posts/bulk-publish/` that:
- Accepts a list of post IDs: `{"post_ids": [1, 2, 3, ...]}`
- Creates a Celery task group to publish all posts in parallel
- Returns a task group ID
- `GET /api/admin/posts/bulk-status/<group_id>/` returns:
  ```json
  {
      "total": 50,
      "completed": 32,
      "failed": 2,
      "pending": 16,
      "percent": 64
  }
  ```

### Part 4: Scheduled Cleanup Tasks
Add these periodic tasks with Celery Beat:

| Task | Schedule | Action |
|------|----------|--------|
| `cleanup_expired_tokens` | Daily 3:00 AM | Delete tokens older than 30 days |
| `update_post_view_counts` | Every 5 min | Sync Redis view counters to DB |
| `clear_old_task_results` | Weekly Sunday 2:00 AM | Delete Celery task results older than 7 days |
| `send_weekly_digest` | Monday 8:00 AM | Email users their weekly reading digest |

### Bonus — Task Priority Queue
Configure two Celery queues:
- `high_priority` → time-sensitive tasks (emails, notifications)
- `low_priority` → background tasks (reports, cleanup)

```python
@shared_task(queue='high_priority')
def send_notification(...): ...

@shared_task(queue='low_priority')
def generate_report(...): ...
```

Start two workers — one per queue:
```bash
celery -A myproject worker -Q high_priority --concurrency=4
celery -A myproject worker -Q low_priority --concurrency=2
```

### Submission Checklist:
- [ ] Welcome email fires on registration (async)
- [ ] Comment + follow notifications work
- [ ] Weekly digest scheduled with Celery Beat
- [ ] Image processing task returns task_id immediately
- [ ] Bulk publish works with parallel task group
- [ ] All 4 periodic tasks are configured
- [ ] Task retries are implemented with exponential backoff

