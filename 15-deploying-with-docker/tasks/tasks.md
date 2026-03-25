# Lesson 15 — In-Class Tasks

## Task 1: Production Settings (15 min)
Create `myproject/settings/prod.py`:
- `DEBUG = False`
- `SECRET_KEY` from env
- `ALLOWED_HOSTS` from env
- PostgreSQL via `dj_database_url`
- WhiteNoise for static files
- Security headers enabled

Run `python manage.py check --deploy` — fix all warnings.

---

## Task 2: Gunicorn (15 min)
1. Install gunicorn
2. Start your app with:
```bash
DJANGO_SETTINGS_MODULE=myproject.settings.prod \
SECRET_KEY=abc123 \
DATABASE_URL=postgresql://... \
gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 2
```

3. Compare: how does the behaviour differ from `runserver`?
4. Test: access `http://localhost:8000/admin/` — notice static files are missing. Why?
5. Fix: run `collectstatic` and use WhiteNoise.

---

## Task 3: Production Dockerfile (20 min)
Create a production `Dockerfile` using multi-stage build.

Build and test it:
```bash
docker build -t blog-api:prod .
docker run -p 8000:8000 \
  -e SECRET_KEY='abc123' \
  -e DEBUG='False' \
  -e ALLOWED_HOSTS='localhost' \
  -e DATABASE_URL='sqlite:///./db.sqlite3' \
  blog-api:prod
```

Does it work? Are static files served? Is admin CSS visible?

---

## Task 4: Production Docker Compose (20 min)
Create `docker-compose.prod.yml` that:
- Uses the production build target
- Reads secrets from `.env.prod`
- Has no bind mounts (no local code)
- Runs `migrate` before starting gunicorn

Run and verify:
```bash
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml exec web python manage.py check --deploy
```

---

## Task 5: `manage.py check --deploy` Audit
Run `python manage.py check --deploy` and fix every warning:
```
WARNINGS:
  ?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting.
  ?: (security.W008) Your SECRET_KEY has less than 50 characters.
  ?: (security.W012) SESSION_COOKIE_SECURE is not set to True.
  ...
```

Document what each security warning means.

