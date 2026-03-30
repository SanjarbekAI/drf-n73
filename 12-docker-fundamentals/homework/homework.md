# Lesson 12 — Homework

## 📌 Assignment: Dockerize the Blog API

### Goal
Create a clean, production-ready Docker setup for your Blog API.

### Part 1: Development Dockerfile
Create `Dockerfile.dev`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install -r requirements.txt
# Code is mounted as volume — don't COPY here
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### Part 2: Production Dockerfile
Create `Dockerfile` (production):
- Multi-stage build
- Non-root user (`django` user for security)
- Uses `gunicorn` with 4 workers
- Runs `collectstatic` during build
- Small final image (use `python:3.12-slim`)

### Part 3: requirements split
Create:
- `requirements/base.txt` — production dependencies
- `requirements/dev.txt` — includes `base.txt` + dev tools
- `requirements/prod.txt` — includes `base.txt` + production extras

```
# requirements/dev.txt
-r base.txt
pytest
pytest-django
factory-boy
django-debug-toolbar
ipython
```

```
# requirements/prod.txt
-r base.txt
gunicorn
```

### Part 4: Settings Split
Create:
- `myproject/settings/base.py`
- `myproject/settings/dev.py`
- `myproject/settings/prod.py`

Each environment reads its own settings:
```bash
# Development
DJANGO_SETTINGS_MODULE=myproject.settings.dev

# Production
DJANGO_SETTINGS_MODULE=myproject.settings.prod
```

### Part 5: Test Your Image
```bash
# Build
docker build -t blog-api:prod .

# Run
docker run -d \
  --name blog-api \
  -p 8000:8000 \
  -e SECRET_KEY='your-prod-secret' \
  -e DEBUG='False' \
  -e ALLOWED_HOSTS='localhost' \
  -e DATABASE_URL='sqlite:///./db.sqlite3' \
  blog-api:prod

# Verify
curl http://localhost:8000/api/
docker logs blog-api

# Clean up
docker stop blog-api && docker rm blog-api
```

### Submission Checklist:
- [ ] `Dockerfile` (production, multi-stage) works and builds successfully
- [ ] `Dockerfile.dev` works with bind mount for live reloading
- [ ] Non-root user is used in the production image
- [ ] `requirements/` is split into base/dev/prod
- [ ] `settings/` is split into base/dev/prod
- [ ] All secrets come from environment variables (nothing hardcoded)
- [ ] `.dockerignore` excludes: `.git`, `.env`, `*.pyc`, `__pycache__`, `venv/`, `media/`
- [ ] Image size is under 400MB

