# 🚀 Full DRF + Docker & Deployment Curriculum

> **Prerequisites:** Python, Django, PostgreSQL  
> **Total:** 17 Lessons — 12 DRF + 5 Docker/Deployment  
> **Teaching Strategy:** Scaffold → Practice → Apply (Bloom's Taxonomy aligned)

---

## 📚 Curriculum Structure

Each lesson contains:
- `lesson.md` — Theory, explanations, and full code examples
- `tasks/` — In-class hands-on exercises (guided)
- `homework/` — Take-home assignments (independent practice)

---

## 🐍 Part 1: Django REST Framework (Lessons 1–12)

| # | Topic | Key Concepts |
|---|-------|-------------|
| 1 | [Introduction to DRF](./1-introduction/) | REST principles, DRF setup, first API |
| 2 | [Serializers](./2-serializers/) | ModelSerializer, validation, nested serializers |
| 3 | [Views & ViewSets](./3-views-and-viewsets/) | APIView, GenericViews, ViewSets, CRUD |
| 4 | [Routers & URL Patterns](./4-routers-and-urls/) | DefaultRouter, nested routes, URL design |
| 5 | [Authentication & Permissions](./5-authentication-and-permissions/) | Session, Token, JWT, custom permissions |
| 6 | [Filtering, Searching & Ordering](./6-filtering-searching-ordering/) | django-filter, SearchFilter, OrderingFilter |
| 7 | [Pagination & Versioning](./7-pagination-and-versioning/) | PageNumber, Cursor, URL/Header versioning |
| 8 | [Throttling & Rate Limiting](./8-throttling-and-ratelimiting/) | AnonRateThrottle, UserRateThrottle, custom |
| 9 | [Caching with Redis](./9-caching-with-redis/) | Redis setup, cache_page, manual caching |
| 10 | [Celery & Async Tasks](./10-celery-and-async-tasks/) | Celery, Redis broker, periodic tasks |
| 11 | [Testing & Documentation](./11-testing-and-documentation/) | APITestCase, pytest, Swagger/ReDoc |
| 12 | [Signals, Middleware & Optimization](./12-signals-custom-middleware-optimization/) | Signals, custom middleware, select_related, N+1 |

---

## 🐳 Part 2: Docker & Deployment (Lessons 13–17)

| # | Topic | Key Concepts |
|---|-------|-------------|
| 13 | [Docker Fundamentals](./13-docker-fundamentals/) | Images, containers, Dockerfile, volumes |
| 14 | [Docker Compose](./14-docker-compose/) | Multi-service apps, networks, env vars |
| 15 | [Deploying with Docker](./15-deploying-with-docker/) | Production Dockerfile, Gunicorn, static files |
| 16 | [Load Balancing with Nginx](./16-load-balancing-with-nginx/) | Nginx reverse proxy, upstream, load balancing |
| 17 | [Microservices + Monitoring + Logging](./17-microservices-monitoring-logging/) | Microservice project, Prometheus, Grafana, ELK |

---

## 🎯 Teaching Philosophy

- **Scaffold First:** Every lesson starts with "why" before "how"
- **Real Projects:** All examples use a consistent Blog/E-commerce project
- **Active Learning:** Tasks reinforce concepts immediately after explanation
- **Spaced Repetition:** Later lessons revisit earlier concepts in new contexts

