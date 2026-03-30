# Lesson 14 — Homework

## 📌 Assignment: Full Production Deployment

### Goal
Deploy your Blog API to a real server, accessible over the internet at `http://your-server-ip/api/`.

### Part 1: Production Settings Hardening
Ensure your `prod.py` settings:
- `DEBUG = False` (verify error pages don't show tracebacks)
- `SECRET_KEY` is at least 50 characters, from environment
- `ALLOWED_HOSTS` doesn't contain `*`
- All security headers are set (HSTS, XSS, Content-Type, etc.)
- `CONN_MAX_AGE = 600` (persistent database connections)
- Email backend is configured for production (not console)
- `LOGGING` writes warnings/errors to console (Docker captures stdout)
- `python manage.py check --deploy` outputs zero warnings

### Part 2: Production Docker Stack
Your `docker-compose.prod.yml` must include:
- `db` with PostgreSQL 15, named volume, healthcheck
- `redis` with Redis 7, named volume
- `web` with Gunicorn, 4 workers, waits for healthy db/redis
- `celery` worker
- `celery-beat` scheduler
- Auto-migration on startup via entrypoint script

### Part 3: Deployment Script
Write `scripts/deploy.sh`:
```bash
#!/bin/bash
set -e  # exit on any error

echo "==> Pulling latest image..."
docker compose -f docker-compose.prod.yml pull

echo "==> Stopping old containers..."
docker compose -f docker-compose.prod.yml down

echo "==> Starting new containers..."
docker compose -f docker-compose.prod.yml up -d

echo "==> Running migrations..."
docker compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput

echo "==> Collecting static files..."
docker compose -f docker-compose.prod.yml exec web python manage.py collectstatic --noinput

echo "==> Deploy complete!"
docker compose -f docker-compose.prod.yml ps
```

### Part 4: CI/CD Pipeline (GitHub Actions)
Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_TOKEN }}
          docker build -t ${{ secrets.DOCKER_USERNAME }}/blogapi:${{ github.sha }} .
          docker push ${{ secrets.DOCKER_USERNAME }}/blogapi:${{ github.sha }}

      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/blogapi
            bash scripts/deploy.sh
```

### Part 5: Backup Strategy
Write a `scripts/backup_db.sh` that:
- Dumps the PostgreSQL database
- Compresses it with gzip
- Names it `backup_YYYY-MM-DD.sql.gz`
- Removes backups older than 7 days

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
docker compose -f docker-compose.prod.yml exec db \
  pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > backups/backup_$DATE.sql.gz

# Remove old backups
find backups/ -name "*.sql.gz" -mtime +7 -delete
echo "Backup saved: backup_$DATE.sql.gz"
```

### Submission Checklist:
- [ ] `python manage.py check --deploy` → zero warnings
- [ ] Production stack starts with `docker compose -f docker-compose.prod.yml up -d`
- [ ] API is accessible at `http://server-ip/api/`
- [ ] Admin panel is accessible with correct CSS/JS
- [ ] Celery tasks execute in production
- [ ] `scripts/deploy.sh` works end-to-end
- [ ] Database backup script works
- [ ] No secrets in any committed file

