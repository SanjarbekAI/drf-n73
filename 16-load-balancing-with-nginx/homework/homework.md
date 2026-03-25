# Lesson 16 — Homework

## 📌 Assignment: Nginx + Load Balanced Production Stack

### Goal
Build a fully load-balanced production deployment with Nginx in front of multiple Django instances.

### Part 1: Complete Nginx Configuration
Create `nginx/nginx.conf` that:
- Redirects all HTTP → HTTPS (port 80 → 443)
- Terminates SSL (use self-signed cert for testing)
- Serves static files directly (bypass Gunicorn)
- Serves media files directly
- Rate limits the API (10 req/s) and auth endpoints (1 req/s, burst 5)
- Enables Gzip compression for JSON responses
- Sets all security headers (HSTS, X-Frame-Options, etc.)
- Logs all requests in the `combined` format

### Part 2: Load-Balanced Docker Compose
Create `docker-compose.prod.yml` with:

```
Internet
    │
  Nginx (1 instance) — port 80/443
    │  round-robin
  ┌─┼─┐
web1 web2 web3  (3 Django/Gunicorn instances)
    │
PostgreSQL + Redis (shared by all web instances)
    │
Celery Worker + Beat
```

Prove load balancing works:
1. Add `X-Served-By: {{ hostname }}` header to all responses
2. Run `for i in {1..9}; do curl -s -I http://localhost/api/ | grep X-Served-By; done`
3. Show that responses rotate across all 3 instances

### Part 3: Health Check Endpoint
Add `GET /api/health/` to Django that returns:
```json
{
    "status": "ok",
    "database": "connected",
    "redis": "connected",
    "hostname": "web-abc123"
}
```

Configure Nginx to use this as an upstream health check and remove unhealthy instances from rotation automatically.

### Part 4: Nginx Caching
Add Nginx proxy caching for public endpoints:
```nginx
# In http block
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m
                 max_size=100m inactive=10m use_temp_path=off;

# In location block
location /api/products/ {
    proxy_cache api_cache;
    proxy_cache_valid 200 5m;      # cache 200 responses for 5 min
    proxy_cache_key "$host$request_uri";
    add_header X-Cache-Status $upstream_cache_status;  # HIT / MISS
    proxy_pass http://django;
}
```

Test: first request shows `X-Cache-Status: MISS`, subsequent shows `HIT`.

### Part 5: Stress Test
Use `ab` (Apache Benchmark) or `wrk` to load test:
```bash
# With 1 web instance
docker compose up -d --scale web=1
ab -n 1000 -c 50 http://localhost/api/products/

# With 3 web instances
docker compose up -d --scale web=3
ab -n 1000 -c 50 http://localhost/api/products/
```

Record:
- Requests per second
- Average response time
- Failed requests
- How much improvement did adding 2 more instances give?

### Submission Checklist:
- [ ] Nginx redirects HTTP → HTTPS
- [ ] Static files served by Nginx (visible in access logs)
- [ ] 3 web instances running with round-robin load balancing
- [ ] `X-Served-By` header rotates across instances
- [ ] Rate limiting fires correctly (429 after threshold)
- [ ] `GET /api/health/` works and returns JSON
- [ ] Nginx caching headers show HIT/MISS
- [ ] Stress test results documented

