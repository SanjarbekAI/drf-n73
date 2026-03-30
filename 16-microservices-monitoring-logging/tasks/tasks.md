# Lesson 16 — In-Class Tasks

## Task 1: Two-Service Setup (30 min)

Create the folder structure:
```
microservices-demo/
├── user-service/    (copy your Blog API, keep only User/Auth)
├── post-service/    (copy your Blog API, keep only Post)
├── nginx/
│   └── nginx.conf
└── docker-compose.yml
```

Write the `docker-compose.yml` with:
- `user-service` on internal port `8001`
- `post-service` on internal port `8002`
- Separate PostgreSQL databases for each service
- `nginx` as the API gateway routing:
  - `/api/users/*` → user-service
  - `/api/posts/*` → post-service

Start everything and test:
```bash
curl http://localhost/api/users/   # reaches user-service
curl http://localhost/api/posts/   # reaches post-service
```

---

## Task 2: Inter-Service Call (25 min)

In the `post-service`, when returning a post, fetch the author's username from the `user-service`:

1. Add `USER_SERVICE_URL=http://user-service:8001` to post-service environment
2. Write a `get_user_info(user_id, token)` function using `requests`
3. Use it in the Post serializer's `get_author` method
4. Add a 3-second timeout and return `{"username": "unknown"}` on failure

Test: does `GET /api/posts/` return author info fetched live from the user-service?

What happens if you stop the user-service? Does the post-service crash or degrade gracefully?

---

## Task 3: Prometheus + Grafana (25 min)

1. Install `django-prometheus` in both services
2. Add to `INSTALLED_APPS` and `MIDDLEWARE` in both
3. Expose `/metrics/` in both services' `urls.py`
4. Create `prometheus.yml` scraping both services
5. Add `prometheus` and `grafana` to `docker-compose.yml`
6. Open Grafana at `http://localhost:3000` (admin/admin)
7. Add Prometheus as a data source
8. Import dashboard ID `17658` (Django Prometheus dashboard from Grafana.com)

Generate traffic: `ab -n 200 -c 10 http://localhost/api/posts/`

Watch the request rate graph update in Grafana!

---

## Task 4: JSON Structured Logging (20 min)

1. Install `python-json-logger`
2. Configure JSON logging in both services' `settings/prod.py`
3. Add log statements in your views:
```python
logger.info("Post retrieved", extra={"post_id": post.id, "user_id": request.user.id})
logger.warning("User service unavailable", extra={"user_id": user_id})
```

4. Run the services and observe:
```bash
docker compose logs post-service | python -m json.tool | head -50
```

Each log line should be valid JSON.

---

## Task 5: Correlation ID Tracing (20 min)

Add `CorrelationIDMiddleware` to BOTH services.

When the post-service calls the user-service, pass the correlation ID in the `X-Correlation-ID` header.

Make a request and find the correlation ID in both services' logs:
```bash
# Make a request, grab the correlation ID from response headers
CORR_ID=$(curl -sI http://localhost/api/posts/1/ | grep -i x-correlation-id | awk '{print $2}')

# Find this ID in BOTH service logs
docker compose logs user-service | grep $CORR_ID
docker compose logs post-service | grep $CORR_ID
```

This shows the full request path across both services!

