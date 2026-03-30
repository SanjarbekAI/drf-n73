# Lesson 15 — In-Class Tasks

## Task 1: Add Nginx to Compose (20 min)
Add an `nginx` service to your `docker-compose.prod.yml`:
```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    - static_files:/app/staticfiles:ro
  depends_on:
    - web
```

Create `nginx/nginx.conf` with a basic reverse proxy to `http://web:8000`.

Test: `curl http://localhost/api/` — does it reach Django through Nginx?

---

## Task 2: Static Files via Nginx (15 min)
Move static file serving from Gunicorn/WhiteNoise to Nginx:
1. Add `location /static/` block pointing to the static volume
2. Rebuild and restart
3. Open `/admin/` — is the CSS loading? Check Nginx access logs to confirm it's serving the CSS files.

```bash
docker compose logs nginx  # watch static file requests
```

---

## Task 3: Load Balancing — 3 Web Instances (20 min)
1. Scale web service to 3 instances:
```bash
docker compose up -d --scale web=3
```
2. Update `nginx.conf` upstream block to list all instances
3. Add a unique identifier to each response to verify load balancing:
```python
# In any view, temporarily:
import socket
response['X-Served-By'] = socket.gethostname()
```
4. Make 6 requests and watch the `X-Served-By` header cycle through different container IDs.

---

## Task 4: Nginx Rate Limiting (15 min)
Add these rate limiting zones to `nginx.conf`:
- General API: 10 requests/second per IP
- Auth endpoints: 1 request/second per IP (burst of 5)

Test: use a loop to send 20 rapid requests to `/api/auth/login/`:
```bash
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    http://localhost/api/auth/login/ \
    -H "Content-Type: application/json" \
    -d '{"username":"u","password":"p"}'
done
```

Count how many `429` responses you get.

---

## Task 5: Nginx Logs Analysis (10 min)
```bash
docker compose exec nginx cat /var/log/nginx/access.log | tail -50
```
Identify:
- Which routes get the most traffic?
- Are there any 4xx or 5xx errors?
- How large are the responses?
- What are the response times?

