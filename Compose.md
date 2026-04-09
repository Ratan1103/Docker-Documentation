# 🐳 Docker Compose

> Run multiple containers together with one file and one command.

---

## ◈ THE PROBLEM COMPOSE SOLVES

```
Without Compose — you run each service manually:

docker run -d --name db -e POSTGRES_PASSWORD=secret postgres
docker run -d --name redis redis
docker run -d --name backend -p 8000:8000 --link db --link redis myapp
docker run -d --name nginx -p 80:80 --link backend nginx

→ 4 commands, easy to mess up, hard to manage


With Compose — one file, one command:

docker compose up -d

→ Done. All 4 services running, networked, in the right order.
```

---

# ━━━ BEGINNER — COMPOSE BASICS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## The Compose File

Create a file named `docker-compose.yml` in your project root.

```yaml
version: "3.9"

services:
  service-name:
    image: nginx
    ports:
      - "8080:80"
```

Run it:
```bash
docker compose up -d
```

That's the minimum. One service, one image, one port.

---

## Core Compose Commands

```bash
# Start everything
docker compose up              # Foreground (see logs)
docker compose up -d           # Background (detached) ✅

# Stop everything
docker compose down            # Stop + remove containers & networks
docker compose down -v         # Also delete volumes ⚠️ (data gone)

# Rebuild images and start
docker compose up -d --build   # Use this after changing Dockerfile ✅

# Restart a single service
docker compose restart backend

# See what's running
docker compose ps

# Read logs
docker compose logs -f                  # All services
docker compose logs -f backend          # One service

# Enter a service shell
docker compose exec backend bash
docker compose exec backend sh          # For alpine images

# Run a one-off command
docker compose run --rm backend python manage.py migrate
docker compose run --rm backend python manage.py createsuperuser
```

---

## What goes inside a service?

```yaml
services:
  myservice:
    image: nginx:alpine           # Use an existing image
    build: ./backend              # OR build from a Dockerfile
    ports:
      - "8080:80"                 # host:container
    environment:
      - DEBUG=True                # Set env variables
      - DB_HOST=db
    volumes:
      - ./code:/app               # Bind mount (your code)
      - mydata:/app/data          # Named volume (persistent data)
    depends_on:
      - db                        # Start db first
    restart: unless-stopped       # Keep it alive
```

---

# ━━━ INTERMEDIATE — REAL EXAMPLES ━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## Example 1 — Django + PostgreSQL + Redis

```yaml
version: "3.9"

services:

  # ── Database ───────────────────────────────────
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
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp_user -d myapp_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Cache ──────────────────────────────────────
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 3

  # ── Django App ─────────────────────────────────
  backend:
    build:
      context: ./backend
      target: production            # Which stage to use (multi-stage)
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://myapp_user:supersecret@db:5432/myapp_db
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: your-secret-key
      DEBUG: "False"
    volumes:
      - static_files:/app/staticfiles
      - media_files:/app/media
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy    # ← Wait for healthcheck, not just start
      redis:
        condition: service_healthy
    networks:
      - app-net
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4"

  # ── Celery Worker ──────────────────────────────
  celery:
    build:
      context: ./backend
      target: production
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://myapp_user:supersecret@db:5432/myapp_db
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - backend
    networks:
      - app-net
    command: celery -A myproject worker --loglevel=info

volumes:
  pgdata:
  redisdata:
  static_files:
  media_files:

networks:
  app-net:
    driver: bridge
```

---

## Example 2 — Django + React + Nginx (Full Stack)

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
        VITE_API_URL: /api            # Relative — nginx proxies it
    networks:
      - app-net

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"                       # Only nginx exposes ports to outside
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

# ━━━ EXPERIENCED — PRODUCTION PATTERNS ━━━━━━━━━━━━━━━━━━━━━

---

## .env File — Keep Secrets Out of compose.yml

```bash
# .env  (in same folder as docker-compose.yml)
# Compose loads this automatically

DB_NAME=myapp_db
DB_USER=myapp_user
DB_PASSWORD=supersecret123
SECRET_KEY=django-insecure-abc123
```

Reference in compose:
```yaml
environment:
  DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
```

```bash
# Use a different env file
docker compose --env-file .env.production up -d
```

> Add `.env` to `.gitignore`. Never commit it.

---

## Dev vs Prod — Multiple Compose Files

```
docker-compose.yml          ← base (shared between dev and prod)
docker-compose.dev.yml      ← development overrides
docker-compose.prod.yml     ← production overrides
```

```yaml
# docker-compose.dev.yml — local development
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
      - /app/node_modules
    command: npm run dev
    ports:
      - "5173:5173"
```

```bash
# Run in development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Run in production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Healthchecks + depends_on — Done Right

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s       # Grace period — don't check for 30s on startup

backend:
  depends_on:
    db:
      condition: service_healthy    # ← This is the correct way
      # NOT just "- db" which only waits for container start, not DB ready
```

---

## Networks — Isolate Your Services

```yaml
services:
  backend:
    networks:
      - app-net

  db:
    networks:
      - app-net       # Only backend and db share this network

  nginx:
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

> Internal services like db and redis should **never** have `ports:` exposed.  
> Only your gateway (nginx) talks to the outside world.

---

## Quick Patterns

```bash
# Rebuild only one service
docker compose up -d --build backend

# Scale a service (run multiple instances)
docker compose up -d --scale backend=3

# Run a one-off command and remove the container after
docker compose run --rm backend python manage.py shell

# Copy file from a service
docker compose cp backend:/app/logs/error.log ./error.log

# See live resource usage
docker compose stats
```

---

## ◈ COMPOSE CHECKLIST

```
□  All services have restart: unless-stopped
□  Database and Redis have healthchecks
□  depends_on uses condition: service_healthy (not just service name)
□  Passwords and secrets are in .env (not hardcoded)
□  .env is in .gitignore
□  Named volumes declared at the bottom
□  Custom network defined
□  Only nginx (gateway) has ports: exposed
□  Static and media volumes shared between backend and nginx
```