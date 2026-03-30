# Lesson 13 — In-Class Tasks

## Task 1: Basic Compose — Django + PostgreSQL (25 min)
Create a `docker-compose.yml` with two services: `web` (Django) and `db` (PostgreSQL).

Requirements:
- Django connects to PostgreSQL via `DATABASE_URL`
- PostgreSQL data is persisted with a named volume
- `web` depends on `db` with a healthcheck
- Running `docker compose up` starts both services

Test:
```bash
docker compose up -d
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
# Visit http://localhost:8000/admin
```

---

## Task 2: Add Redis (15 min)
Add a `redis` service to your Compose file.

Update Django settings to use:
- Redis as the cache backend: `redis://redis:6379/0`
- Redis as the Celery broker: `redis://redis:6379/0`

Test Redis connection from inside the web container:
```bash
docker compose exec web python -c "
from django.core.cache import cache
cache.set('hello', 'docker compose works!')
print(cache.get('hello'))
"
```

---

## Task 3: Add Celery Worker (15 min)
Add a `celery` service that runs:
```
celery -A myproject worker --loglevel=info
```

Trigger a task from the Django shell and watch the Celery worker logs:
```bash
docker compose exec web python manage.py shell
>>> from myapp.tasks import send_welcome_email
>>> send_welcome_email.delay(1)

# In another terminal:
docker compose logs -f celery
```

---

## Task 4: Development Override (15 min)
Create `docker-compose.override.yml` for development:
- Mount local code as a volume (live reload)
- Use `python manage.py runserver` instead of gunicorn
- Add Flower for Celery monitoring on port 5555

Verify: change a view file and refresh — does it auto-reload?

---

## Task 5: Environment File (10 min)
1. Create a `.env` file with all secrets
2. Create a `.env.example` template (safe to commit)
3. Add `.env` to `.gitignore`
4. Update `docker-compose.yml` to read all variables from `.env`

---

## Task 6: Commands Practice
Try each command and describe what it does:
```bash
docker compose ps
docker compose logs --tail=50 web
docker compose exec db psql -U bloguser -d blogdb
docker compose run --rm web python manage.py check
docker compose down -v   # careful: deletes volumes!
```

