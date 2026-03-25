# Lesson 17: Microservices, Monitoring & Logging

## 🎯 Learning Objectives
By the end of this lesson, students will be able to:
- Explain the microservices architecture and when to use it
- Build a simple two-service system with Docker Compose
- Set up Prometheus + Grafana for metrics and dashboards
- Set up centralized logging with the ELK Stack (Elasticsearch, Logstash, Kibana)
- Use structured JSON logging in Django
- Implement distributed tracing basics with correlation IDs

---

## 1. Microservices vs Monolith

### Monolith (what you've built so far):
```
┌─────────────────────────────────┐
│         Blog Monolith            │
│  Posts │ Users │ Notifications   │
│  Auth  │ Media │ Search          │
└─────────────────────────────────┘
        Single codebase
        Single database
        Single deployment
```

**Pros:** Simple to develop, test, deploy. Good for teams < 10 engineers.
**Cons:** Hard to scale individual parts. One bug can take down everything. Long deployment times as codebase grows.

### Microservices:
```
┌──────────┐  ┌──────────┐  ┌──────────────┐
│ User Svc │  │ Post Svc │  │ Notification │
│ (Django) │  │ (Django) │  │ Svc (FastAPI)│
└────┬─────┘  └────┬─────┘  └──────┬───────┘
     │              │               │
     └──────────────┴───────────────┘
                    │
              API Gateway (Nginx)
                    │
                 Client
```

**Pros:** Independent scaling, independent deployment, technology freedom.
**Cons:** Network complexity, distributed debugging, data consistency challenges.

**Rule of thumb:** Start with a monolith. Split when a specific bottleneck demands it.

---

## 2. Simple Two-Service Project

We'll build:
- **User Service** — handles registration, login, user profiles
- **Post Service** — handles posts, reads user data from User Service via API call

### Project structure:
```
microservices-demo/
├── user-service/
│   ├── Dockerfile
│   ├── manage.py
│   ├── users/
│   └── requirements.txt
├── post-service/
│   ├── Dockerfile
│   ├── manage.py
│   ├── posts/
│   └── requirements.txt
├── nginx/
│   └── nginx.conf
└── docker-compose.yml
```

---

## 3. Inter-Service Communication

Services communicate over HTTP (REST):

```python
# post-service/posts/services.py
import requests
from django.conf import settings

def get_user_info(user_id: int, auth_token: str) -> dict:
    """Fetch user info from the User Service."""
    try:
        response = requests.get(
            f"{settings.USER_SERVICE_URL}/api/users/{user_id}/",
            headers={"Authorization": f"Bearer {auth_token}"},
            timeout=5,
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.Timeout:
        raise ServiceUnavailableError("User service timed out")
    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 404:
            raise UserNotFoundError(f"User {user_id} not found")
        raise ServiceUnavailableError(f"User service error: {e}")
```

```python
# post-service/posts/serializers.py
class PostSerializer(serializers.ModelSerializer):
    author = serializers.SerializerMethodField()

    def get_author(self, obj):
        # Fetch from User Service
        from .services import get_user_info
        request = self.context.get('request')
        token = request.auth if request else None
        try:
            return get_user_info(obj.author_id, str(token))
        except Exception:
            return {"id": obj.author_id, "username": "unknown"}

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at']
```

---

## 4. Docker Compose for Microservices

```yaml
# docker-compose.yml

services:
  # ─── API Gateway ────────────────────────────────────────────────────
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - user-service
      - post-service

  # ─── User Service ───────────────────────────────────────────────────
  user-service:
    build: ./user-service
    expose:
      - "8001"
    environment:
      DATABASE_URL: postgresql://user:pass@user-db:5432/userdb
      SECRET_KEY: ${USER_SERVICE_SECRET_KEY}
    depends_on:
      user-db:
        condition: service_healthy
    command: gunicorn userservice.wsgi:application --bind 0.0.0.0:8001

  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - user_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      retries: 5

  # ─── Post Service ───────────────────────────────────────────────────
  post-service:
    build: ./post-service
    expose:
      - "8002"
    environment:
      DATABASE_URL: postgresql://post:pass@post-db:5432/postdb
      SECRET_KEY: ${POST_SERVICE_SECRET_KEY}
      USER_SERVICE_URL: http://user-service:8001
    depends_on:
      post-db:
        condition: service_healthy
      user-service:
        condition: service_started
    command: gunicorn postservice.wsgi:application --bind 0.0.0.0:8002

  post-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: postdb
      POSTGRES_USER: post
      POSTGRES_PASSWORD: pass
    volumes:
      - post_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U post"]
      interval: 10s
      retries: 5

volumes:
  user_db_data:
  post_db_data:
```

### Nginx Gateway config:
```nginx
# nginx/nginx.conf
upstream user_service { server user-service:8001; }
upstream post_service { server post-service:8002; }

server {
    listen 80;

    location /api/users/ {
        proxy_pass http://user_service;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/posts/ {
        proxy_pass http://post_service;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 5. Monitoring with Prometheus + Grafana

```
Django App → exposes /metrics (via django-prometheus)
      ↓
Prometheus → scrapes /metrics every 15s → stores time-series data
      ↓
Grafana → queries Prometheus → displays dashboards
```

### Setup:
```bash
pip install django-prometheus
```

```python
# settings.py
INSTALLED_APPS = ['django_prometheus', ...]
MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # ... all other middleware ...
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]
```

```python
# urls.py
from django_prometheus import exports

urlpatterns = [
    path('metrics/', exports.ExportToDjangoView, name='prometheus-django-metrics'),
    ...
]
```

Now `/metrics` exposes:
- Request counts by endpoint/method/status
- Request duration histograms
- Database query counts
- Cache hit/miss rates

### `prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'user-service'
    static_configs:
      - targets: ['user-service:8001']
    metrics_path: '/metrics/'

  - job_name: 'post-service'
    static_configs:
      - targets: ['post-service:8002']
    metrics_path: '/metrics/'
```

### Add to Docker Compose:
```yaml
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
    depends_on:
      - prometheus
```

---

## 6. Custom Prometheus Metrics

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Track custom business events
post_created_total = Counter(
    'blog_posts_created_total',
    'Total number of posts created',
    ['user_type']  # label: 'free' or 'premium'
)

post_publish_duration = Histogram(
    'blog_post_publish_duration_seconds',
    'Time to process a post publish event',
)

active_users_gauge = Gauge(
    'blog_active_users',
    'Number of currently active users'
)
```

```python
# views.py
from .metrics import post_created_total

class PostViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
        user_type = 'premium' if self.request.user.profile.is_premium else 'free'
        post_created_total.labels(user_type=user_type).inc()
```

---

## 7. Structured Logging for Django

Instead of plain text logs, use JSON so they can be parsed by log aggregators:

```bash
pip install python-json-logger
```

```python
# settings/prod.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
    'loggers': {
        'django.request': {
            'handlers': ['console'],
            'level': 'WARNING',
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
    }
}
```

### Logging in views with context:
```python
import logging

logger = logging.getLogger(__name__)

class PostViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        post = serializer.save(author=self.request.user)
        logger.info(
            "Post created",
            extra={
                "post_id": post.id,
                "author_id": self.request.user.id,
                "request_id": getattr(self.request, 'request_id', None),
            }
        )
```

Output (JSON):
```json
{"asctime": "2026-03-25 10:00:01", "name": "posts.views", "levelname": "INFO",
 "message": "Post created", "post_id": 42, "author_id": 7, "request_id": "a1b2c3"}
```

---

## 8. ELK Stack for Log Aggregation

```
Django (JSON logs to stdout)
    ↓
Docker (captures stdout)
    ↓
Filebeat (ships logs to Logstash)
    ↓
Logstash (parses/transforms)
    ↓
Elasticsearch (stores & indexes)
    ↓
Kibana (search, visualize, alert)
```

### Docker Compose addition (simplified with just Elasticsearch + Kibana):
```yaml
  elasticsearch:
    image: elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: kibana:8.12.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: elastic/filebeat:8.12.0
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch
```

### `filebeat.yml`:
```yaml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "django-logs-%{+yyyy.MM.dd}"
```

---

## 9. Correlation ID — Tracing Across Services

```python
# middleware.py
import uuid
import logging

logger = logging.getLogger(__name__)

class CorrelationIDMiddleware:
    """
    Attach a unique correlation ID to every request.
    If the upstream service passes X-Correlation-ID, reuse it.
    This allows tracing a request across multiple microservices.
    """
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        correlation_id = (
            request.META.get('HTTP_X_CORRELATION_ID') or
            str(uuid.uuid4())
        )
        request.correlation_id = correlation_id

        response = self.get_response(request)
        response['X-Correlation-ID'] = correlation_id
        return response
```

```python
# When calling another service, pass the correlation ID:
def get_user_info(user_id, token, correlation_id=None):
    headers = {"Authorization": f"Bearer {token}"}
    if correlation_id:
        headers["X-Correlation-ID"] = correlation_id
    response = requests.get(f"{USER_SERVICE_URL}/api/users/{user_id}/", headers=headers)
    return response.json()
```

Now a single client request generates logs in both services with the **same correlation ID** — you can trace the full flow across services.

---

## 10. Alerting with Grafana

In Grafana, create alert rules:
- **High error rate**: if 5xx responses > 5% of total → alert
- **Slow responses**: if p95 latency > 2 seconds → alert
- **Database connection pool**: if connections > 80% of max → alert
- **Service down**: if `up` metric = 0 → alert immediately

Alerts can notify via Slack, Email, PagerDuty, etc.

---

## 📝 Key Takeaways
1. Start with a monolith — split to microservices only when scale demands it
2. Services communicate via HTTP — always handle timeouts and failures gracefully
3. Each microservice has its OWN database — never share databases between services
4. Prometheus scrapes metrics; Grafana visualizes them — a powerful, free combo
5. JSON structured logs are machine-parseable — critical for log aggregation with ELK
6. Correlation IDs are essential for tracing requests across multiple services
7. Monitor: request rate, error rate, latency (p50/p95/p99), and saturation

