# Lesson 15: Deploying with Docker

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain the difference between development and production setups
- Configure Django for production (DEBUG, ALLOWED_HOSTS, static files)
- Serve static files with WhiteNoise or Nginx
- Use Gunicorn as a production WSGI server
- Set up HTTPS with Let's Encrypt
- Deploy to a cloud server (DigitalOcean/VPS)

---

## 1. Development vs Production Differences

| Setting | Development | Production |
|---------|-------------|------------|
| `DEBUG` | `True` | `False` |
| `ALLOWED_HOSTS` | `['*']` | `['yourdomain.com']` |
| Server | Django runserver | Gunicorn |
| Static files | Django serves | Nginx/WhiteNoise |
| Database | SQLite or local PG | Managed PostgreSQL |
| Secrets | In code / `.env` | Environment vars / secrets manager |
| Error pages | Debug traceback | Generic error page |
| HTTPS | Optional | Required |

---

## 2. Production `settings.py`

```python
# settings/prod.py
import os
from .base import *

DEBUG = False

SECRET_KEY = os.environ['SECRET_KEY']  # raises error if not set

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# Database
import dj_database_url
DATABASES = {
    'default': dj_database_url.parse(os.environ['DATABASE_URL'], conn_max_age=600)
}

# Security headers
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Static files with WhiteNoise
MIDDLEWARE = ['whitenoise.middleware.WhiteNoiseMiddleware'] + MIDDLEWARE
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
}
```

---

## 3. Gunicorn — Production WSGI Server

Django's built-in `runserver` is single-threaded and unsafe for production.

Gunicorn uses multiple worker processes:

```bash
pip install gunicorn
```

```bash
# Basic usage
gunicorn myproject.wsgi:application

# Production configuration
gunicorn myproject.wsgi:application \
  --bind 0.0.0.0:8000 \
  --workers 4 \           # 2 * CPU cores + 1
  --threads 2 \
  --worker-class gthread \
  --worker-tmp-dir /dev/shm \
  --timeout 30 \
  --keepalive 5 \
  --max-requests 1000 \
  --max-requests-jitter 100 \
  --log-level info \
  --access-logfile -      # log to stdout
```

---

## 4. Static Files with WhiteNoise

```bash
pip install whitenoise
```

WhiteNoise serves static files directly from Gunicorn — no separate Nginx needed for simple deployments:

```python
# settings/prod.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # right after security
    ...
]

STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

```bash
python manage.py collectstatic --noinput
```

---

## 5. Production `Dockerfile`

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    DJANGO_SETTINGS_MODULE=myproject.settings.prod

WORKDIR /app

# ─── Dependencies ─────────────────────────────────────────────────────
FROM base AS deps
COPY requirements/prod.txt .
RUN pip install --upgrade pip && pip install -r prod.txt

# ─── Application ──────────────────────────────────────────────────────
FROM base AS production
COPY --from=deps /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=deps /usr/local/bin /usr/local/bin

COPY . .

# Static files (require SECRET_KEY at build time — use placeholder)
ARG SECRET_KEY=build-only-not-real
RUN python manage.py collectstatic --noinput

# Non-root user
RUN addgroup --system django && \
    adduser --system --ingroup django --no-create-home django
RUN chown -R django:django /app
USER django

EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "30", \
     "--log-level", "info"]
```

---

## 6. Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file: .env.prod
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data

  web:
    build:
      context: .
      target: production
    restart: always
    ports:
      - "8000:8000"
    env_file: .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: >
      sh -c "python manage.py migrate --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4"

  celery:
    build:
      context: .
      target: production
    restart: always
    env_file: .env.prod
    depends_on:
      - db
      - redis
    command: celery -A myproject worker --loglevel=warning --concurrency=4

volumes:
  postgres_data:
  redis_data:
```

---

## 7. Deploying to a VPS (DigitalOcean / Hetzner)

```bash
# On your local machine:
# 1. Build and push image
docker build -t yourusername/blogapi:v1.0 .
docker push yourusername/blogapi:v1.0

# 2. SSH into your server
ssh root@your-server-ip

# 3. Install Docker on the server (Ubuntu)
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin

# 4. Create app directory
mkdir -p /opt/blogapi
cd /opt/blogapi

# 5. Upload files (from local)
scp docker-compose.prod.yml root@your-server-ip:/opt/blogapi/
scp .env.prod root@your-server-ip:/opt/blogapi/

# 6. On the server: pull and start
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d

# 7. Check it's running
docker compose -f docker-compose.prod.yml ps
curl http://localhost:8000/api/
```

---

## 8. SSL with Let's Encrypt (Certbot)

```bash
# Install certbot on server
apt install certbot python3-certbot-nginx

# Get certificate (Nginx must not be running on port 80)
certbot certonly --standalone -d yourdomain.com

# Certificates are at:
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/yourdomain.com/privkey.pem

# Auto-renew (already set up by certbot)
certbot renew --dry-run
```

---

## 📝 Key Takeaways
1. `DEBUG=False` is mandatory in production — it stops Django from showing tracebacks to users
2. Gunicorn replaces `runserver` in production — it's multi-process and battle-tested
3. WhiteNoise is the simplest way to serve static files without a separate Nginx
4. Always use `dj-database-url` to parse `DATABASE_URL` — works with any database
5. Never hardcode secrets — use environment variables loaded from `.env.prod`
6. `collectstatic` must run before the app starts (in entrypoint or Dockerfile)

