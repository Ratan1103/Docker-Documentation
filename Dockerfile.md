# 🐳 Writing Dockerfiles — Templates & Patterns

> **Look at the template → know exactly what to write for any project**

---

## ◈ THE ANATOMY OF A DOCKERFILE

```
Dockerfile instruction flow:
─────────────────────────────────────────────────────

FROM        ← What base OS/runtime to start from
│
├── ARG     ← Build-time variables (only during docker build)
├── ENV     ← Runtime environment variables
│
├── WORKDIR ← Set working directory inside container
│
├── COPY / ADD  ← Bring files from host → container
│
├── RUN     ← Execute shell commands (installs, builds)
│
├── EXPOSE  ← Document which port the app uses
│
├── VOLUME  ← Declare mount points
│
└── CMD / ENTRYPOINT  ← What runs when container starts
```

---

## ◈ MASTER TEMPLATE

```dockerfile
# ════════════════════════════════════════════════════
# STAGE 1 — BASE (shared setup)
# ════════════════════════════════════════════════════
FROM python:3.12-slim AS base

# Build arguments (available only during build)
ARG APP_ENV=production
ARG VERSION=1.0.0

# Environment variables (available at runtime)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_ENV=${APP_ENV}

# Set working directory — all following commands run from here
WORKDIR /app

# Install system dependencies (apt packages, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*    # ← Always clean up to reduce image size


# ════════════════════════════════════════════════════
# STAGE 2 — DEPENDENCIES (install packages)
# ════════════════════════════════════════════════════
FROM base AS deps

# Copy ONLY dependency files first (Docker cache optimization)
# If requirements.txt doesn't change → Docker reuses this layer
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt


# ════════════════════════════════════════════════════
# STAGE 3 — FINAL IMAGE (production)
# ════════════════════════════════════════════════════
FROM deps AS production

# Copy application code
COPY . .

# Create non-root user (security best practice)
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Document the port (doesn't actually publish — just metadata)
EXPOSE 8000

# Healthcheck — Docker will monitor this
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# Default command
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

## ◈ INSTRUCTION QUICK REFERENCE

### `FROM` — Base Image

```dockerfile
FROM ubuntu:22.04            # Full Ubuntu OS
FROM python:3.12-slim        # Python + minimal Debian (recommended)
FROM python:3.12-alpine      # Python + Alpine Linux (smallest, but quirks)
FROM node:20-alpine          # Node.js on Alpine
FROM nginx:alpine            # Nginx web server

# Multi-stage naming
FROM python:3.12-slim AS builder
FROM python:3.12-slim AS production
```

### `RUN` — Execute Commands

```dockerfile
# Bad — creates unnecessary layers
RUN apt-get update
RUN apt-get install -y gcc
RUN pip install django

# ✅ Good — chain with && to keep it one layer
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Python packages
RUN pip install --no-cache-dir -r requirements.txt

# Node packages
RUN npm ci --only=production
```

### `COPY` vs `ADD`

```dockerfile
COPY src/ /app/src/           # Copy files/dirs from host
COPY . .                      # Copy everything (respect .dockerignore)

ADD https://example.com/file.tar.gz /tmp/   # ADD can fetch URLs
ADD archive.tar.gz /app/                    # ADD auto-extracts tar files
# Use COPY unless you need ADD's special features
```

### `ENV` vs `ARG`

```
ARG  → Only during build     →  docker build --build-arg KEY=val
ENV  → During build + runtime →  docker run -e KEY=val
```

```dockerfile
ARG NODE_ENV=production           # only in build
ENV DATABASE_URL=postgres://...   # persists in container

# Can chain ARG → ENV
ARG APP_VERSION
ENV VERSION=${APP_VERSION}
```

### `CMD` vs `ENTRYPOINT`

```
ENTRYPOINT  →  The fixed executable, always runs
CMD         →  Default arguments, can be overridden

Together: ENTRYPOINT + CMD = full command
Alone:    CMD = the whole command (easily overridden)
```

```dockerfile
# CMD only (most common for apps)
CMD ["python", "manage.py", "runserver"]

# ENTRYPOINT + CMD
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver"]                          # default: runserver
# docker run myapp migrate                 # override: migrate

# Shell form (avoid in prod — can't handle signals properly)
CMD python manage.py runserver             # ❌ don't use
CMD ["python", "manage.py", "runserver"]  # ✅ exec form
```

### `WORKDIR`, `EXPOSE`, `VOLUME`, `USER`

```dockerfile
WORKDIR /app                  # Create + cd into dir
WORKDIR /app/src              # Can chain, relative to previous

EXPOSE 8000                   # Documentation only, not published
EXPOSE 8000/tcp               # TCP (default)
EXPOSE 8000/udp

VOLUME ["/app/media"]         # Declare mount point

USER appuser                  # Switch to non-root user
USER 1001                     # By UID
```

---

## ◈ .dockerignore — Always Include This

```
# .dockerignore
__pycache__/
*.pyc
*.pyo
.env
.env.*
.git/
.gitignore
node_modules/
*.log
.DS_Store
dist/
build/
coverage/
.pytest_cache/
*.sqlite3
media/
staticfiles/
README.md
docker-compose*.yml
Dockerfile*
```

---

## ◈ MULTI-STAGE BUILD — WHY IT MATTERS

```
Without multi-stage:
┌─────────────────────────────────────┐
│  Python + gcc + node + build tools  │  ← 800MB+ image
│  + all dev dependencies + app code  │
└─────────────────────────────────────┘

With multi-stage:
┌──────────────────────────┐
│  Build Stage (discarded) │  ← Has all build tools
│  python + node + gcc     │
└────────────┬─────────────┘
             │  copies only built artifacts
             ▼
┌──────────────────────────┐
│  Final Stage (shipped)   │  ← Only 120MB
│  python-slim + your app  │
└──────────────────────────┘
```

---

## ◈ FULL EXAMPLE — Django + PostgreSQL + React

### Project Structure

```
myproject/
├── backend/
│   ├── manage.py
│   ├── requirements.txt
│   ├── myproject/
│   └── Dockerfile
├── frontend/
│   ├── package.json
│   ├── src/
│   └── Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── Dockerfile
└── docker-compose.yml
```

---

### `backend/Dockerfile`

```dockerfile
# ════════════════════════════════════════
# STAGE 1 — Python deps
# ════════════════════════════════════════
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ════════════════════════════════════════
# STAGE 2 — Production image
# ════════════════════════════════════════
FROM python:3.12-slim AS production

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    DJANGO_SETTINGS_MODULE=myproject.settings.production

WORKDIR /app

# Only install runtime system libs (no gcc needed anymore)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages from builder
COPY --from=builder /install /usr/local

# Copy app source
COPY . .

# Create static/media directories
RUN mkdir -p /app/staticfiles /app/media

# Security: non-root user
RUN addgroup --system django && adduser --system --ingroup django django
RUN chown -R django:django /app
USER django

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# Use gunicorn in production
CMD ["gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "120", \
     "--access-logfile", "-"]
```

---

### `backend/requirements.txt`

```
django==5.0.4
djangorestframework==3.15.1
psycopg2-binary==2.9.9
gunicorn==22.0.0
python-decouple==3.8
django-cors-headers==4.3.1
celery==5.3.6
redis==5.0.3
whitenoise==6.6.0
```

---

### `frontend/Dockerfile`

```dockerfile
# ════════════════════════════════════════
# STAGE 1 — Build React app
# ════════════════════════════════════════
FROM node:20-alpine AS builder

WORKDIR /app

# Install deps first (cache layer)
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY . .
ARG VITE_API_URL=http://localhost:8000
ENV VITE_API_URL=${VITE_API_URL}
RUN npm run build


# ════════════════════════════════════════
# STAGE 2 — Serve with Nginx
# ════════════════════════════════════════
FROM nginx:alpine AS production

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy our nginx config
COPY nginx.conf /etc/nginx/conf.d/

# Copy built React files
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=10s \
    CMD wget -qO- http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

---

### `frontend/nginx.conf`

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # React Router — serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to Django
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Proxy static/media from Django
    location /static/ { proxy_pass http://backend:8000; }
    location /media/  { proxy_pass http://backend:8000; }
}
```

---

### `nginx/Dockerfile` (Optional — main reverse proxy)

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
```

---

## ◈ LAYER CACHING STRATEGY

```
WRONG ORDER (slow builds):
  COPY . .                  ← Any code change → ALL layers below re-run
  RUN pip install -r req.txt

✅ RIGHT ORDER (fast builds):
  COPY requirements.txt .   ← Only re-runs if requirements.txt changes
  RUN pip install -r req.txt
  COPY . .                  ← Code changes only affect this + below
```

---

## ◈ DOCKERFILE CHECKLIST

```
□ Using slim/alpine base image (not full ubuntu/python)
□ Multi-stage build (separate builder and production)
□ Dependency files copied before source code (cache optimization)
□ --no-cache-dir on pip install
□ rm -rf /var/lib/apt/lists/* after apt install
□ Non-root USER set
□ .dockerignore file created
□ EXPOSE port declared
□ HEALTHCHECK added
□ CMD uses exec form (JSON array, not shell string)
□ Secrets not hardcoded (use ENV or build args from outside)
```

---

> 💡 **Rule of thumb** — Every `RUN` is a layer. Fewer layers = smaller image.  
> Chain commands with `&&` and clean up in the same `RUN`.