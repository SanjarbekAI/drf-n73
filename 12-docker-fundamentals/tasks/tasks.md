# Lesson 12 — In-Class Tasks

## Task 1: Your First Dockerfile (20 min)
Write a `Dockerfile` for your Django Blog API:
- Base image: `python:3.12-slim`
- Working directory: `/app`
- Install requirements
- Copy project
- Expose port 8000
- Run with `python manage.py runserver 0.0.0.0:8000`

Build and run it:
```bash
docker build -t blog-api .
docker run -p 8000:8000 blog-api
```

Visit `http://localhost:8000/api/` — does it work?

---

## Task 2: .dockerignore (10 min)
Create a `.dockerignore` file. Build again and compare the image sizes:
```bash
docker images blog-api
```

What was removed? How much smaller is the image?

---

## Task 3: Environment Variables (15 min)
Move these settings out of `settings.py` to environment variables:
- `SECRET_KEY`
- `DEBUG`
- `ALLOWED_HOSTS`
- `DATABASE_URL` (even if not yet using PostgreSQL in Docker)

Run with:
```bash
docker run -p 8000:8000 \
  -e SECRET_KEY='abc123supersecret' \
  -e DEBUG='True' \
  -e ALLOWED_HOSTS='localhost,127.0.0.1' \
  blog-api
```

---

## Task 4: Volume for Development (15 min)
Mount your local code as a bind mount so code changes auto-reload:
```bash
docker run -p 8000:8000 \
  -v ${PWD}:/app \
  -e SECRET_KEY='abc123' \
  -e DEBUG='True' \
  blog-api
```

Change a view in your code, refresh the browser — did it reload?

---

## Task 5: Docker Exec (10 min)
While your container is running:
```bash
# Open a shell inside the container
docker exec -it <container_id> bash

# Inside the container:
python manage.py shell
python manage.py migrate
python manage.py createsuperuser
```

---

## Task 6: Multi-Stage Build (Bonus, 20 min)
Create `Dockerfile.prod` with a multi-stage build:
- Stage 1: install dependencies
- Stage 2: production image with Gunicorn

Compare image sizes:
```bash
docker build -t blog-api:dev .
docker build -t blog-api:prod -f Dockerfile.prod .
docker images | grep blog-api
```

