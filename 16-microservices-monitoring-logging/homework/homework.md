# Lesson 16 — Homework (Capstone Project)

## 📌 Capstone: Microservices Blog Platform with Full Observability

This is the final project of the entire curriculum. You will build a two-service system with production-grade monitoring, logging, and load balancing — everything from Lessons 13–17 combined.

---

## Architecture

```
                     Internet
                         │
                    Nginx (Gateway)
                    port 80/443
                   /             \
          /api/users/*        /api/posts/*
               │                   │
        User Service           Post Service
        (Django + PG)          (Django + PG)
        port 8001               port 8002
        2 replicas              3 replicas
               \                   /
                Redis (shared broker)
                         │
                   Celery Worker
                         │
            ┌────────────┼────────────┐
        Prometheus   Grafana        ELK
        (metrics)   (dashboards)  (logs)
```

---

## Part 1: User Service

### Endpoints:
```
POST   /api/auth/register/     → register, return JWT
POST   /api/auth/login/        → login, return JWT
POST   /api/auth/logout/       → blacklist token
GET    /api/users/<id>/        → get user profile (internal use by Post Service)
PATCH  /api/users/me/          → update own profile
GET    /api/users/me/          → get own profile
```

### Features:
- JWT authentication
- User profile with `bio`, `avatar`, `is_premium`
- Prometheus metrics on `/metrics/`
- JSON structured logging
- Correlation ID middleware
- `GET /api/health/` health check endpoint

---

## Part 2: Post Service

### Endpoints:
```
GET    /api/posts/             → list posts (paginated, filterable)
POST   /api/posts/             → create post
GET    /api/posts/<id>/        → get post (with author info from User Service)
PATCH  /api/posts/<id>/        → update post (owner only)
DELETE /api/posts/<id>/        → delete post (owner only)
POST   /api/posts/<id>/like/   → toggle like
GET    /api/posts/trending/    → top 10 most-liked posts (cached 15 min in Redis)
```

### Features:
- JWT authentication (validate token independently — don't call User Service for auth)
- Fetch author info from User Service on `GET` endpoints
- Graceful degradation if User Service is down (`{"username": "unknown"}`)
- Redis caching for trending posts
- Celery task: on post creation, send notification to author via User Service webhook
- Prometheus metrics on `/metrics/`
- JSON structured logging with `post_id`, `author_id`, `correlation_id` in every log
- Correlation ID middleware — pass ID through to User Service calls

---

## Part 3: Docker Compose Stack

### `docker-compose.yml` services:

| Service | Replicas | Purpose |
|---------|----------|---------|
| `nginx` | 1 | API Gateway + load balancer |
| `user-service` | 2 | User management |
| `post-service` | 3 | Post management |
| `user-db` | 1 | PostgreSQL for users |
| `post-db` | 1 | PostgreSQL for posts |
| `redis` | 1 | Cache + Celery broker |
| `celery` | 1 | Async task worker |
| `prometheus` | 1 | Metrics scraper |
| `grafana` | 1 | Metrics dashboard |
| `elasticsearch` | 1 | Log storage |
| `kibana` | 1 | Log dashboard |
| `filebeat` | 1 | Log shipper |

---

## Part 4: Nginx Configuration

```nginx
# Route /api/users/* to user-service (2 instances, round-robin)
# Route /api/posts/* to post-service (3 instances, round-robin)
# Rate limit: auth endpoints 1 req/s, others 20 req/s
# SSL on port 443
# Serve static from volume
# Gzip enabled
# X-Served-By header for debugging
```

---

## Part 5: Monitoring — Grafana Dashboards

Set up at least **2 dashboards**:

### Dashboard 1: API Overview
- Total requests/sec (by service)
- Error rate (5xx / total)
- Average response time
- Active database connections

### Dashboard 2: Business Metrics
- Posts created per hour
- New user registrations per hour
- Trending posts (by likes)
- Cache hit rate

### Alerts:
Set up Grafana alerts for:
- Error rate > 5% → notify (use Grafana webhook or email)
- Response latency p95 > 2s → notify
- Any service has `up = 0` → notify immediately

---

## Part 6: Logging — Kibana Setup

1. Configure Filebeat to ship Docker container logs to Elasticsearch
2. In Kibana, create an index pattern `django-logs-*`
3. Create a **saved search** that shows only ERROR level logs
4. Create a **visualization**: bar chart of log levels over time (INFO, WARNING, ERROR)
5. Create a **dashboard** combining the above

### Demonstrate:
Trigger an error intentionally (e.g., force a 500 error) and find it in Kibana within 30 seconds.

---

## Part 7: Correlation ID Demo

Record a video or write a step-by-step walkthrough showing:

1. Client sends `GET http://localhost/api/posts/1/`
2. Nginx adds/passes `X-Correlation-ID: abc-123`
3. Post Service logs: `{"message": "Post retrieved", "post_id": 1, "correlation_id": "abc-123"}`
4. Post Service calls User Service with same correlation ID
5. User Service logs: `{"message": "User info fetched", "user_id": 5, "correlation_id": "abc-123"}`
6. Client receives response with `X-Correlation-ID: abc-123` header
7. In Kibana, search `correlation_id: abc-123` → see ALL logs from BOTH services for this single request

---

## Part 8: Scripts

### `scripts/start.sh` — one command to start everything:
```bash
#!/bin/bash
echo "Starting microservices stack..."
docker compose pull
docker compose up -d
echo "Waiting for services to be healthy..."
sleep 15
docker compose exec user-service python manage.py migrate
docker compose exec post-service python manage.py migrate
echo ""
echo "🚀 All services running:"
echo "  API:         http://localhost/api/"
echo "  Grafana:     http://localhost:3000"
echo "  Kibana:      http://localhost:5601"
echo "  Prometheus:  http://localhost:9090"
echo "  Flower:      http://localhost:5555"
```

### `scripts/health_check.sh`:
```bash
#!/bin/bash
echo "Checking service health..."
curl -s http://localhost/api/users/health/ | python -m json.tool
curl -s http://localhost/api/posts/health/ | python -m json.tool
echo "All checks done."
```

---

## Submission Checklist

### Functionality:
- [ ] User registration/login works via `http://localhost/api/users/auth/register/`
- [ ] Post creation works via `http://localhost/api/posts/`
- [ ] Post detail shows author info fetched from User Service
- [ ] Stopping the User Service → Post Service degrades gracefully
- [ ] Redis caching works for trending posts
- [ ] Celery task fires on post creation

### Infrastructure:
- [ ] `docker compose up` starts all 14 services cleanly
- [ ] User Service runs on 2 replicas, Post Service on 3
- [ ] Nginx routes `/api/users/*` and `/api/posts/*` correctly
- [ ] Load balancing verified with `X-Served-By` header

### Monitoring:
- [ ] Prometheus scrapes both services (`http://localhost:9090/targets`)
- [ ] Grafana has API Overview dashboard
- [ ] Grafana has Business Metrics dashboard
- [ ] At least 1 alert rule configured

### Logging:
- [ ] All logs are JSON structured
- [ ] Kibana index pattern is set up
- [ ] Kibana dashboard shows log level breakdown
- [ ] Error logs are findable in Kibana within 30 seconds of occurring

### Observability:
- [ ] Correlation ID is present in every request/response header
- [ ] Correlation ID traces through both services' logs
- [ ] `GET /api/health/` works on both services
- [ ] `scripts/start.sh` starts the whole stack cleanly

---

## 🏆 Congratulations!

You have completed the full curriculum:
- ✅ 12 lessons of Django REST Framework
- ✅ 5 lessons of Docker, Compose, deployment, load balancing, microservices + observability

You now know how to build, test, secure, optimize, containerize, deploy, scale, monitor, and debug production-grade Django APIs.

