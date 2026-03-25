# Lesson 14: Docker Compose

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain why Docker Compose is needed for multi-service apps
- Write `docker-compose.yml` files for Django + PostgreSQL + Redis + Celery
- Use networks and volumes in Compose
- Use environment variable files (`.env`)
- Override configs with `docker-compose.override.yml`
- Run migrations, management commands in Compose

---

## 1. Why Docker Compose?

Your app has multiple services:
- Django app
- PostgreSQL database
- Redis (cache + broker)
- Celery worker
- Celery Beat
- Nginx

Without Compose, you'd run each container manually, configure network links by hand, and manage them separately. Compose defines ALL services in one file and starts them all with one command.

```bash
docker compose up        # start everything
docker compose down      # stop everything
docker compose logs -f   # see all logs
```

---

## 2. Full `docker-compose.yml` for Django

```yaml
# docker-compose.yml

version: '3.9'

services:

  # ─── PostgreSQL Database ──────────────────────────────────────────
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-blogdb}
      POSTGRES_USER: ${POSTGRES_USER:-bloguser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-blogpass}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-bloguser}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Redis ───────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # ─── Django App ──────────────────────────────────────────────────
  web:
    build: .
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - static_files:/app/staticfiles
      - media_files:/app/media
    environment:
      SECRET_KEY: ${SECRET_KEY}
      DEBUG: ${DEBUG:-False}
      ALLOWED_HOSTS: ${ALLOWED_HOSTS:-localhost}
      DATABASE_URL: postgresql://${POSTGRES_USER:-bloguser}:${POSTGRES_PASSWORD:-blogpass}@db:5432/${POSTGRES_DB:-blogdb}
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/0
      CELERY_RESULT_BACKEND: redis://redis:6379/1
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4"

  # ─── Celery Worker ───────────────────────────────────────────────
  celery:
    build: .
    restart: unless-stopped
    environment:
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgresql://${POSTGRES_USER:-bloguser}:${POSTGRES_PASSWORD:-blogpass}@db:5432/${POSTGRES_DB:-blogdb}
      CELERY_BROKER_URL: redis://redis:6379/0
      CELERY_RESULT_BACKEND: redis://redis:6379/1
    depends_on:
      - db
      - redis
    command: celery -A myproject worker --loglevel=info --concurrency=4

  # ─── Celery Beat (Scheduler) ─────────────────────────────────────
  celery-beat:
    build: .
    restart: unless-stopped
    environment:
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgresql://${POSTGRES_USER:-bloguser}:${POSTGRES_PASSWORD:-blogpass}@db:5432/${POSTGRES_DB:-blogdb}
      CELERY_BROKER_URL: redis://redis:6379/0
    depends_on:
      - db
      - redis
    command: celery -A myproject beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler

# ─── Volumes ─────────────────────────────────────────────────────────
volumes:
  postgres_data:
  redis_data:
  static_files:
  media_files:
```

---

## 3. Environment File (`.env`)

```bash
# .env — NEVER commit this to git
SECRET_KEY=your-very-secret-django-key-here-make-it-long
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1

POSTGRES_DB=blogdb
POSTGRES_USER=bloguser
POSTGRES_PASSWORD=supersecretpassword

DJANGO_SETTINGS_MODULE=myproject.settings.prod
```

```bash
# .env.example — commit this as a template
SECRET_KEY=change-me
DEBUG=True
ALLOWED_HOSTS=localhost

POSTGRES_DB=blogdb
POSTGRES_USER=bloguser
POSTGRES_PASSWORD=change-me
```

---

## 4. Development Override (`docker-compose.override.yml`)

Compose automatically merges this with `docker-compose.yml` in development:

```yaml
# docker-compose.override.yml (development only)
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app        # mount code for live reload
    ports:
      - "8000:8000"
    environment:
      DEBUG: "True"
    command: python manage.py runserver 0.0.0.0:8000

  # Flower — Celery monitoring UI (dev only)
  flower:
    build: .
    ports:
      - "5555:5555"
    environment:
      CELERY_BROKER_URL: redis://redis:6379/0
    depends_on:
      - redis
    command: celery -A myproject flower --port=5555
```

---

## 5. Docker Compose Commands

```bash
# Start all services
docker compose up

# Start in background
docker compose up -d

# Start specific service
docker compose up web

# Stop all
docker compose down

# Stop and remove volumes (DANGER — deletes data)
docker compose down -v

# View logs
docker compose logs
docker compose logs -f web      # follow web service logs
docker compose logs web celery  # multiple services

# Run a one-off command
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
docker compose exec web python manage.py shell

# Run without starting: (creates container, runs command, removes it)
docker compose run --rm web python manage.py migrate

# Rebuild images
docker compose build
docker compose up --build   # build + start

# Scale a service
docker compose up --scale celery=3   # 3 celery workers
```

---

## 6. Healthchecks — Service Dependencies

`depends_on` alone doesn't wait for the database to be READY (just started):

```yaml
db:
  image: postgres:15
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s

web:
  depends_on:
    db:
      condition: service_healthy   # wait until healthcheck passes
```

---

## 7. Networks — Service Isolation

```yaml
services:
  web:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend  # only reachable by web and celery, not exposed

  nginx:
    networks:
      - frontend  # only frontend

networks:
  frontend:
  backend:
    internal: true  # cannot be reached from outside Docker
```

---

## 📝 Key Takeaways
1. Docker Compose manages multi-service apps with a single config file
2. Use `depends_on` with `condition: service_healthy` to wait for real readiness
3. Store secrets in `.env` files — never hardcode them in `docker-compose.yml`
4. Use `docker-compose.override.yml` for development-specific settings
5. Named volumes persist data across container restarts
6. `docker compose exec web python manage.py ...` runs management commands inside the running container

