# Lesson 15: Load Balancing with Nginx

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain what a reverse proxy does and why it's needed
- Configure Nginx as a reverse proxy for Gunicorn
- Set up load balancing across multiple Django instances
- Serve static and media files directly from Nginx
- Configure rate limiting and caching at the Nginx level
- Use SSL/TLS termination at Nginx

---

## 1. Why Nginx in Front of Gunicorn?

```
Internet → Nginx → Gunicorn → Django → Database
```

Nginx handles:
- **SSL termination** — decrypt HTTPS, pass HTTP to Gunicorn
- **Static files** — serve without hitting Python at all
- **Load balancing** — distribute requests across multiple Gunicorn instances
- **Rate limiting** — block abuse at the network level
- **Gzip compression** — compress responses before sending
- **Request buffering** — protect Gunicorn from slow clients

Gunicorn's job: run Python. That's it.

---

## 2. Basic Nginx Config — Reverse Proxy

```nginx
# nginx/nginx.conf

upstream django {
    server web:8000;  # 'web' = Docker service name
}

server {
    listen 80;
    server_name localhost yourdomain.com;

    # Proxy all API requests to Gunicorn
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Serve static files directly — bypass Gunicorn
    location /static/ {
        alias /app/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Serve media files directly
    location /media/ {
        alias /app/media/;
        expires 7d;
    }
}
```

---

## 3. Load Balancing — Multiple Django Instances

```nginx
# nginx/nginx.conf — round-robin load balancing

upstream django {
    # Round-robin (default) — distributes evenly
    server web1:8000;
    server web2:8000;
    server web3:8000;
}

# Least connections — send to whichever has fewest active requests
upstream django_leastconn {
    least_conn;
    server web1:8000;
    server web2:8000;
    server web3:8000;
}

# IP hash — same client always hits same server (sticky sessions)
upstream django_iphash {
    ip_hash;
    server web1:8000;
    server web2:8000;
    server web3:8000;
}

# Weighted — web1 gets 3x more traffic than web2
upstream django_weighted {
    server web1:8000 weight=3;
    server web2:8000 weight=1;
}
```

---

## 4. Docker Compose with Nginx + Multiple Web Instances

```yaml
# docker-compose.prod.yml

services:
  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - static_files:/app/staticfiles:ro
      - media_files:/app/media:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - web

  web:
    build:
      context: .
      target: production
    restart: always
    # No ports exposed — only accessible via Nginx
    expose:
      - "8000"
    env_file: .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    deploy:
      replicas: 3          # 3 instances for load balancing
    command: gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4

  db:
    image: postgres:15-alpine
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file: .env.prod

  redis:
    image: redis:7-alpine
    restart: always

volumes:
  postgres_data:
  redis_data:
  static_files:
  media_files:
```

---

## 5. Full Production Nginx Config

```nginx
# nginx/nginx.conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log warn;

    # Performance
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=1r/s;

    # Upstream Django instances
    upstream django {
        server web:8000;
        # Add more for load balancing:
        # server web2:8000;
        # server web3:8000;
    }

    server {
        listen 80;
        server_name yourdomain.com www.yourdomain.com;

        # Redirect all HTTP → HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name yourdomain.com www.yourdomain.com;

        # SSL certificates
        ssl_certificate     /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;

        # Max upload size
        client_max_body_size 10M;

        # Static files — no Gunicorn involved
        location /static/ {
            alias /app/staticfiles/;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        location /media/ {
            alias /app/media/;
            expires 30d;
        }

        # Auth endpoints — stricter rate limit
        location ~ ^/api/(auth/login|auth/register|auth/password) {
            limit_req zone=auth burst=5 nodelay;
            limit_req_status 429;
            proxy_pass http://django;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # All other API requests
        location / {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://django;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }
    }
}
```

---

## 6. Scaling Horizontally

```bash
# Scale to 3 web instances
docker compose up -d --scale web=3

# Nginx automatically distributes load across all 3
```

To make this work with Nginx's upstream, all replicas must be on the same Docker network. Docker Compose handles this automatically.

---

## 📝 Key Takeaways
1. Nginx sits in front of Gunicorn as a reverse proxy — handles network-level concerns
2. Static/media files served by Nginx directly — 100x faster than going through Python
3. Upstream block in Nginx defines load balancing targets
4. `replicas: 3` in Compose + Nginx upstream = horizontal scaling
5. SSL termination happens at Nginx — Gunicorn only sees plain HTTP
6. Rate limiting at Nginx level is more efficient than at the DRF level

