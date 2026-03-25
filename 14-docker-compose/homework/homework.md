# Lesson 14 — Homework

## 📌 Assignment: Full Development Environment with Docker Compose

### Goal
Build a complete, production-like Docker Compose setup for your Blog API that any developer can spin up with a single command.

### Required Services

| Service | Image/Build | Purpose |
|---------|-------------|---------|
| `db` | `postgres:15-alpine` | Database |
| `redis` | `redis:7-alpine` | Cache + Celery broker |
| `web` | Your Dockerfile | Django app |
| `celery` | Your Dockerfile | Async task worker |
| `celery-beat` | Your Dockerfile | Scheduled tasks |
| `flower` | Your Dockerfile | Celery monitoring UI |

### Requirements

#### `docker-compose.yml` (production-like):
- All services use healthchecks
- `web` depends on `db` and `redis` being healthy
- `celery` and `celery-beat` depend on `db` and `redis` being healthy
- Static and media files use named volumes
- No hardcoded secrets — everything from `.env`

#### `docker-compose.override.yml` (development):
- Code mounted as bind volume for live reload
- `web` runs with `runserver` (not gunicorn)
- `DEBUG=True`
- Port `5555` exposed for Flower

#### Init Script
Create a `scripts/entrypoint.sh` that the `web` service runs:
```bash
#!/bin/sh
echo "Waiting for database..."
# wait for DB to be ready
echo "Running migrations..."
python manage.py migrate
echo "Collecting static files..."
python manage.py collectstatic --noinput
echo "Starting server..."
exec "$@"
```

```dockerfile
# In Dockerfile:
COPY scripts/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

#### `Makefile` (developer convenience):
```makefile
.PHONY: up down logs shell migrate createsuperuser test

up:
	docker compose up -d

down:
	docker compose down

logs:
	docker compose logs -f

shell:
	docker compose exec web python manage.py shell

migrate:
	docker compose exec web python manage.py migrate

createsuperuser:
	docker compose exec web python manage.py createsuperuser

test:
	docker compose exec web pytest

build:
	docker compose build --no-cache
```

#### `README.md` — Setup instructions:
Write a clear setup guide:
```
## Quick Start
1. Clone the repo
2. Copy env file: cp .env.example .env
3. Edit .env with your values
4. Start: make up
5. Run migrations: make migrate
6. Visit: http://localhost:8000/api/
7. Admin: http://localhost:8000/admin/
8. Flower: http://localhost:5555
```

### Submission Checklist:
- [ ] `docker compose up` starts ALL services without errors
- [ ] Migrations run automatically on first start
- [ ] Redis cache works (verify with `make shell`)
- [ ] Celery task execution works (trigger a task, see it in Flower)
- [ ] Celery Beat fires scheduled tasks
- [ ] `docker compose.override.yml` enables live code reload
- [ ] `.env.example` is committed; `.env` is gitignored
- [ ] `Makefile` works with at least 6 targets
- [ ] README explains setup in < 10 steps

