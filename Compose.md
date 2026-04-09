# 🐳 Docker Compose — Complete Guide

> **Orchestrate multi-container apps with one file**  
> Examples-first. Minimal explanation.

---

## ◈ WHAT COMPOSE DOES

```
Without Compose:                    With Compose:
────────────────                    ─────────────
docker run -d \                     docker compose up -d
  --name db \
  -e POSTGRES_DB=app \              → Starts ALL services
  -e POSTGRES_PASSWORD=secret \     → Creates networks automatically
  -v pgdata:/var/lib/... \          → Handles volumes
  --network mynet \                 → Correct startup order
  postgres:16                       → One command to rule them all

docker run -d \
  --name redis ...

docker run -d \
  --name backend ...

docker run -d \
  --name frontend ...
```

---

## ◈ COMPOSE COMMANDS

```bash
# ── Start / Stop ────────────────────────────────────────────────────────────
docker compose up               # Start all services (foreground)
docker compose up -d            # -d → background (detached)
docker compose up --build       # Rebuild images before starting
docker compose up -d --build    # Rebuild + run in background ✅ most used

docker compose down             # Stop + remove containers & networks
docker compose down -v          # -v → also delete volumes (⚠️ data loss)
docker compose down --rmi all   # Also remove images

docker compose stop             # Stop containers (keep them)
docker compose start            # Start stopped containers
docker compose restart          # Restart all or specific service
docker compose restart backend  # Restart one service

# ── Logs ─────────────────────────────────────────────────────────────────────
docker compose logs             # All service logs
docker compose logs -f          # Follow live
docker compose logs -f backend  # Follow one service
docker compose logs --tail=50 backend

# ── Manage ───────────────────────────────────────────────────────────────────
docker compose ps               # Status of all services
docker compose exec backend bash        # Shell into service
docker compose exec backend python manage.py migrate   # Run command
docker compose run --rm backend pytest  # One-off command, removes after

# ── Build ────────────────────────────────────────────────────────────────────
docker compose build            # Build all images
docker compose build backend    # Build one service
docker compose pull             # Pull latest images from registry

# ── Scale ────────────────────────────────────────────────────────────────────
docker compose up -d --scale backend=3  # Run 3 instances of backend
```

---

## ◈ THE COMPOSE FILE STRUCTURE

```yaml
# docker-compose.yml

version: "3.9"              # Compose file format version

services:                   # Your containers go here
  service-name:
    image: ...              # Use existing image
    build: ...              # Or build from Dockerfile
    ports: ...              # Port mappings
    environment: ...        # Env variables
    volumes: ...            # Mounts
    depends_on: ...         # Startup order
    networks: ...           # Which network to join
    restart: ...            # Restart policy
    healthcheck: ...        # Health monitoring

volumes:                    # Named volumes declaration
  volume-name:

networks:                   # Custom networks
  network-name:
```

---

## ◈ EXAMPLE 1 — Django + PostgreSQL + Redis

```yaml
# docker-compose.yml

version: "3.9"

services:

  # ── PostgreSQL ─────────────────────────────────────
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp_db
      POSTGRES_USER: myapp_user
      POSTGRES_PASSWORD: supersecret
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp_user -d myapp_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Redis ──────────────────────────────────────────
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data
    networks:
      - backend-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  # ── Django Backend ────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: production         # Multi-stage: which stage to use
    restart: unless-stopped
    environment:
      DEBUG: "False"
      DATABASE_URL: postgres://myapp_user:supersecret@db:5432/myapp_db
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: your-secret-key-here
      ALLOWED_HOSTS: "localhost,127.0.0.1"
    volumes:
      - static_files:/app/staticfiles
      - media_files:/app/media
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy   # Wait for DB to be ready
      redis:
        condition: service_healthy
    networks:
      - backend-net
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4"

  # ── Celery Worker ─────────────────────────────────
  celery:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: production
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://myapp_user:supersecret@db:5432/myapp_db
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - backend
      - redis
    networks:
      - backend-net
    command: celery -A myproject worker --loglevel=info

# ── Volumes ──────────────────────────────────────────
volumes:
  pgdata:
  redisdata:
  static_files:
  media_files:

# ── Networks ─────────────────────────────────────────
networks:
  backend-net:
    driver: bridge
```

---

## ◈ EXAMPLE 2 — Full Stack (Django + React + Nginx)

```yaml
version: "3.9"

services:

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      retries: 5

  backend:
    build:
      context: ./backend
      target: production
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      SECRET_KEY: ${SECRET_KEY}
      DEBUG: "False"
    volumes:
      - static_files:/app/staticfiles
      - media_files:/app/media
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-net
    command: >
      sh -c "python manage.py migrate --noinput &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000"

  frontend:
    build:
      context: ./frontend
      args:
        VITE_API_URL: /api       # Relative — nginx will proxy
    networks:
      - app-net

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_files:/app/staticfiles:ro
      - media_files:/app/media:ro
    depends_on:
      - backend
      - frontend
    networks:
      - app-net
    restart: unless-stopped

volumes:
  pgdata:
  static_files:
  media_files:

networks:
  app-net:
    driver: bridge
```

---

## ◈ ENVIRONMENT VARIABLES — `.env` FILE

```bash
# .env  (same directory as docker-compose.yml)
# Compose picks this up automatically

DB_NAME=myapp_db
DB_USER=myapp_user
DB_PASSWORD=supersecret123
SECRET_KEY=django-insecure-abc123xyz

# Never commit .env to git!
```

```yaml
# In compose file — reference with ${VAR_NAME}
environment:
  DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
```

```bash
# Use different env file
docker compose --env-file .env.production up -d
```

---

## ◈ MULTIPLE COMPOSE FILES (dev vs prod)

```
docker-compose.yml          ← base (shared config)
docker-compose.dev.yml      ← development overrides
docker-compose.prod.yml     ← production overrides
```

```yaml
# docker-compose.dev.yml — override for local dev
services:
  backend:
    volumes:
      - ./backend:/app        # Live code reload
    environment:
      DEBUG: "True"
    command: python manage.py runserver 0.0.0.0:8000

  frontend:
    volumes:
      - ./frontend:/app
      - /app/node_modules     # anonymous volume to preserve node_modules
    command: npm run dev
    ports:
      - "5173:5173"
```

```bash
# Development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## ◈ HEALTHCHECKS & DEPENDS_ON

```yaml
# Ensure service is truly ready before starting dependent service

db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s      # Check every 10s
    timeout: 5s        # Wait max 5s for response
    retries: 5         # Mark unhealthy after 5 fails
    start_period: 30s  # Grace period on startup

backend:
  depends_on:
    db:
      condition: service_healthy  # Wait for healthcheck to pass
```

---

## ◈ RESTART POLICIES

```yaml
restart: no              # Never (default)
restart: always          # Always, even on clean exit
restart: on-failure      # Only on non-zero exit code
restart: unless-stopped  # Always, unless docker stop was used ✅ production
```

---

## ◈ VOLUMES IN COMPOSE

```yaml
services:
  backend:
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume
      - ./backend:/app                    # bind mount (host dir)
      - /app/node_modules                 # anonymous volume (don't touch)
      - ./config.json:/app/config.json:ro # bind mount, read-only

volumes:
  pgdata:              # Docker manages this
    driver: local
```

---

## ◈ NETWORKS IN COMPOSE

```yaml
services:
  backend:
    networks:
      - app-net
      - monitoring-net     # Can be on multiple networks

  db:
    networks:
      - app-net            # Only backend can reach db

  prometheus:
    networks:
      - monitoring-net

networks:
  app-net:
    driver: bridge
  monitoring-net:
    driver: bridge
    internal: true         # No external internet access
```

---

## ◈ QUICK PATTERNS

### Run a migration one-off

```bash
docker compose run --rm backend python manage.py migrate
docker compose run --rm backend python manage.py createsuperuser
docker compose run --rm backend python manage.py shell
```

### Rebuild just one service

```bash
docker compose up -d --build backend
```

### Scale a service

```bash
docker compose up -d --scale backend=3
```

### Copy file from container

```bash
docker compose cp backend:/app/logs/error.log ./error.log
```

---

## ◈ COMPOSE FILE CHECKLIST

```
□ All services have restart: unless-stopped
□ Healthchecks added for db and redis
□ depends_on uses condition: service_healthy (not just service name)
□ Secrets in .env file (not hardcoded)
□ .env added to .gitignore
□ Named volumes declared at bottom
□ Custom network defined (don't rely on default)
□ No ports exposed for internal services (db, redis)
□ Static/media volumes shared between backend and nginx
```

---

> 💡 **Golden Rule** — Internal services (db, redis, celery) should NOT have `ports:` exposed.  
> Only Nginx (or your gateway) should expose ports to the outside world.