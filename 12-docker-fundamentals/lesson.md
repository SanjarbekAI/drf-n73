# Lesson 12: Docker Fundamentals

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain containers vs virtual machines
- Write a `Dockerfile` for a Django application
- Build and run Docker images
- Use volumes for persistent data and development
- Use environment variables in containers
- Push images to Docker Hub

---

## 1. Why Docker?

**The "works on my machine" problem:**
```
Developer's laptop:  Python 3.11, PostgreSQL 15, Redis 7  → Works ✅
Staging server:      Python 3.9,  PostgreSQL 13, Redis 6  → Breaks ❌
Production server:   Python 3.8,  PostgreSQL 14            → Breaks ❌
```

**With Docker:**
```
Same container image runs identically on:
  - Developer's laptop  ✅
  - CI/CD pipeline      ✅
  - Staging             ✅
  - Production          ✅
```

---

## 2. Containers vs Virtual Machines

```
Virtual Machine:              Container:
┌──────────────────┐          ┌──────────────────┐
│  Your App        │          │  Your App        │
│  Python 3.11     │          │  Python 3.11     │
│  Ubuntu OS       │          │  (thin layer)    │
│  Guest OS kernel │          ├──────────────────┤
│  Hypervisor      │          │  Host OS Kernel  │
│  Host OS         │          │  (shared)        │
└──────────────────┘          └──────────────────┘
  Size: ~2-10 GB                Size: ~100-500 MB
  Boot: 30-60 seconds           Boot: < 1 second
```

---

## 3. Core Docker Concepts

| Concept | What it is |
|---------|-----------|
| **Image** | Blueprint (like a class) — read-only snapshot |
| **Container** | Running instance of an image (like an object) |
| **Dockerfile** | Instructions to build an image |
| **Registry** | Storage for images (Docker Hub, ECR, GCR) |
| **Volume** | Persistent storage that survives container restart |
| **Network** | How containers communicate |

---

## 4. Writing a Dockerfile for Django

```dockerfile
# Dockerfile

# ─── Stage 1: Base image ────────────────────────────────────────────
FROM python:3.12-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# Set working directory inside container
WORKDIR /app

# ─── Install dependencies ────────────────────────────────────────────
# Copy only requirements first (Docker layer caching optimization)
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

# ─── Copy project files ──────────────────────────────────────────────
COPY . .

# ─── Collect static files ────────────────────────────────────────────
RUN python manage.py collectstatic --noinput

# ─── Expose port ─────────────────────────────────────────────────────
EXPOSE 8000

# ─── Run the application ─────────────────────────────────────────────
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

---

## 5. `.dockerignore` — Exclude Unnecessary Files

```
# .dockerignore
__pycache__/
*.pyc
*.pyo
.env
.git/
.gitignore
*.md
*.log
db.sqlite3
.venv/
venv/
node_modules/
media/
staticfiles/
```

---

## 6. Essential Docker Commands

```bash
# Build an image
docker build -t myblog:latest .
docker build -t myblog:v1.2 .

# List images
docker images

# Run a container
docker run -p 8000:8000 myblog:latest
docker run -p 8000:8000 -d myblog:latest      # -d = detached (background)
docker run -p 8000:8000 --name myblog myblog:latest

# List running containers
docker ps
docker ps -a  # all including stopped

# Container logs
docker logs myblog
docker logs -f myblog  # follow (live)

# Execute a command inside a running container
docker exec -it myblog bash
docker exec -it myblog python manage.py shell

# Stop and remove
docker stop myblog
docker rm myblog

# Remove image
docker rmi myblog:latest

# Clean up everything
docker system prune -a
```

---

## 7. Environment Variables

Never hardcode secrets. Pass them at runtime:

```bash
# Pass env vars at run time
docker run -p 8000:8000 \
  -e SECRET_KEY='your-secret-key' \
  -e DATABASE_URL='postgresql://user:pass@db:5432/mydb' \
  -e DEBUG='False' \
  myblog:latest

# Or use an env file
docker run -p 8000:8000 --env-file .env myblog:latest
```

```python
# settings.py — always read from environment
import os
from pathlib import Path

SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-key-change-in-prod')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost').split(',')
DATABASE_URL = os.environ.get('DATABASE_URL')
```

---

## 8. Volumes — Persistent & Development Storage

```bash
# Named volume (persistent — survives container removal)
docker run -v postgres_data:/var/lib/postgresql/data postgres:15

# Bind mount (development — mounts local code into container)
docker run -v $(pwd):/app -p 8000:8000 myblog:latest
# Any changes to local code immediately reflect in the container
```

---

## 9. Multi-Stage Build — Smaller Production Images

```dockerfile
# Dockerfile.prod

# ─── Stage 1: Build ──────────────────────────────────────────────────
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ─── Stage 2: Production ─────────────────────────────────────────────
FROM python:3.12-slim AS production
WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .

# Create non-root user (security best practice)
RUN addgroup --system django && adduser --system --ingroup django django
USER django

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "myproject.wsgi:application"]
```

---

## 10. Push to Docker Hub

```bash
# Login
docker login

# Tag with your username/repo
docker tag myblog:latest yourusername/myblog:latest
docker tag myblog:latest yourusername/myblog:v1.0

# Push
docker push yourusername/myblog:latest

# Pull on another machine
docker pull yourusername/myblog:latest
```

---

## 📝 Key Takeaways
1. Containers solve the "works on my machine" problem by packaging the app + environment together
2. Docker layers are cached — put rarely-changing instructions early in the Dockerfile
3. Copy `requirements.txt` and install dependencies BEFORE copying your code (cache optimization)
4. Never put secrets in a Dockerfile or image — use environment variables
5. Use `.dockerignore` to keep images small and build times fast
6. Multi-stage builds dramatically reduce production image size

