# Lesson 9 — In-Class Tasks

## Task 1: Celery Setup (20 min)
1. Create `myproject/celery.py` with the Celery app configuration
2. Update `myproject/__init__.py`
3. Add `CELERY_BROKER_URL` and `CELERY_RESULT_BACKEND` to `settings.py`
4. Start the Celery worker: `celery -A myproject worker --loglevel=info`
5. Test with a simple task:
```python
@shared_task
def add(x, y):
    return x + y

# In Django shell:
# from myapp.tasks import add
# result = add.delay(3, 4)
# result.status   # 'SUCCESS'
# result.result   # 7
```

---

## Task 2: Welcome Email Task (20 min)
Build a registration flow where:
1. `POST /api/auth/register/` creates the user
2. Immediately queues a `send_welcome_email` task
3. Returns `201 Created` response without waiting for the email

Simulate the email using `print()` or `django.core.mail.backends.console.EmailBackend`.

Verify the response is fast (< 100ms) even though the email task takes 2 seconds (use `time.sleep(2)` in the task).

---

## Task 3: Long-Running Task with Status Polling (25 min)
Implement a "report generation" feature:

1. `POST /api/reports/` → starts a `generate_report` task, returns `{"task_id": "abc123"}`
2. `GET /api/reports/status/<task_id>/` → returns task status:
   - `PENDING` → `{"status": "pending"}`
   - `SUCCESS` → `{"status": "done", "result": {...}}`
   - `FAILURE` → `{"status": "failed", "error": "..."}`

Use `time.sleep(5)` in the task to simulate slow work.

---

## Task 4: Retry on Failure (15 min)
Modify `send_welcome_email` to:
- Raise a `ConnectionError` 50% of the time (use `random.random() < 0.5`)
- Retry up to 3 times with a 5-second delay between retries
- Log each retry attempt

Run the task 5 times and observe retry behavior in the Celery worker logs.

---

## Task 5: Celery Beat — Periodic Task (20 min)
Set up a periodic task that runs every 1 minute and:
- Counts how many new posts were created in the last minute
- Prints the count to the worker log

Start `celery beat` and verify it fires every minute.

