# 🐳 Writing a Dockerfile

> Learn what each instruction does → use the template → build real projects.

---

## ◈ WHAT IS A DOCKERFILE?

A Dockerfile is a plain text file with instructions. Docker reads it top to bottom and builds your image step by step.

```
Dockerfile  →  docker build  →  IMAGE  →  docker run  →  CONTAINER
```

---

# ━━━ BEGINNER — THE CORE INSTRUCTIONS ━━━━━━━━━━━━━━━━━━━━━━

These 7 instructions cover 90% of what you'll write in any Dockerfile.

---

## FROM — Start With a Base

Every Dockerfile must start with `FROM`. It picks the base OS or runtime.

```dockerfile
FROM python:3.12-slim
FROM node:20-alpine
FROM ubuntu:22.04
FROM nginx:alpine
```

You're not starting from scratch — you're building on top of something that already has Python, Node, etc. installed.

```
slim    → smaller image, Debian-based, recommended for most apps
alpine  → smallest possible, but some packages behave differently
latest  → always pulls newest (avoid in prod — unpredictable)
```

---

## WORKDIR — Set Your Working Directory

Creates a folder and makes it the default location for everything that follows.

```dockerfile
WORKDIR /app
```

All `COPY`, `RUN`, `CMD` instructions after this run inside `/app`.  
If the folder doesn't exist, Docker creates it automatically.

---

## COPY — Bring Files In

```dockerfile
COPY source destination

COPY requirements.txt .        # Copy one file to WORKDIR
COPY . .                       # Copy everything from your project
COPY src/ /app/src/            # Copy a specific folder
```

The `.` on the right means "current WORKDIR" (which is `/app` if you set it).

---

## RUN — Execute Commands During Build

```dockerfile
RUN command

RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y curl
RUN npm install
```

Every `RUN` creates a new **layer** in the image.  
Chain commands with `&&` to keep it one layer:

```dockerfile
# ❌ 3 separate layers (bigger image)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ 1 layer (smaller image)
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

---

## ENV — Set Environment Variables

```dockerfile
ENV KEY=VALUE

ENV PYTHONUNBUFFERED=1
ENV NODE_ENV=production
ENV PORT=8000
```

These variables are available **inside the container** when it runs.  
Use them for config that shouldn't be hardcoded in your code.

---

## EXPOSE — Document the Port

```dockerfile
EXPOSE 8000
EXPOSE 3000
```

This is **documentation only** — it doesn't actually open the port.  
You still need `-p 8000:8000` when you `docker run`.  
Think of it as a label saying "this app listens on port 8000."

---

## CMD — The Default Command

```dockerfile
CMD ["command", "arg1", "arg2"]

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]
```

This runs when the container starts. Only **one CMD** per Dockerfile (last one wins).

> Always use the JSON array format `["cmd", "arg"]` — not the string format.  
> String format: `CMD python manage.py runserver` ← skip this in production.

---

# ━━━ BEGINNER — YOUR FIRST DOCKERFILE ━━━━━━━━━━━━━━━━━━━━━━

```dockerfile
# Simple Python app
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

That's it. 7 lines. This pattern works for almost any simple app.

---

# ━━━ INTERMEDIATE — IMPORTANT PATTERNS ━━━━━━━━━━━━━━━━━━━━━

---

## Layer Caching — Order Matters

```
Wrong order → slow builds every time:

COPY . .                         ← any code change → invalidates everything below
RUN pip install -r requirements.txt   ← reinstalls every time! ❌


Right order → fast builds:

COPY requirements.txt .          ← only re-runs if requirements change
RUN pip install -r requirements.txt
COPY . .                         ← code changes only affect this layer ✅
```

Rule: **copy dependency files first, install, then copy your code.**

---

## ARG — Build-Time Variables

```dockerfile
ARG NODE_ENV=production

docker build --build-arg NODE_ENV=development .
```

`ARG` is only available **during** `docker build`. Not at runtime.  
`ENV` is available **at runtime** inside the container.

---

## ENTRYPOINT vs CMD

```dockerfile
# CMD alone — full command, can be overridden
CMD ["python", "manage.py", "runserver"]

# ENTRYPOINT + CMD — ENTRYPOINT is fixed, CMD is the default arg
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver"]

# docker run myapp migrate   ← overrides CMD, keeps ENTRYPOINT
# → runs: python manage.py migrate
```

For most web apps, `CMD` alone is fine.  
Use `ENTRYPOINT` when your container should always run as a specific tool.

---

## HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1
```

Docker checks this regularly. Container shows as `healthy`, `unhealthy`, or `starting`.

---

## USER — Don't Run as Root

```dockerfile
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser
```

Running as root inside a container is a security risk.  
Always switch to a non-root user before `CMD`.

---

## .dockerignore — Keep Your Image Clean

Create this file next to your Dockerfile. Docker won't copy anything listed here.

```
# .dockerignore
__pycache__/
*.pyc
.env
.env.*
.git/
node_modules/
*.log
.DS_Store
dist/
build/
coverage/
*.sqlite3
media/
staticfiles/
```

Without `.dockerignore`, Docker copies `node_modules`, `.git`, and all junk into the image. Always include it.

---

# ━━━ EXPERIENCED — MULTI-STAGE BUILDS ━━━━━━━━━━━━━━━━━━━━━━

---

## Why Multi-Stage?

```
Single-stage:                          Multi-stage:
┌─────────────────────────┐            ┌────────────────────┐
│ Python + gcc + node     │            │ Build Stage        │  ← discarded
│ + all dev tools         │  900MB     │ (has all tools)    │
│ + your app              │            └────────┬───────────┘
└─────────────────────────┘                     │ copies only
                                                │ what's needed
                                       ┌────────▼───────────┐
                                       │ Final Stage        │  ← shipped
                                       │ python-slim + app  │  120MB
                                       └────────────────────┘
```

---

# ━━━ FULL EXAMPLE — Django + PostgreSQL + React ━━━━━━━━━━━━━

---

## Project Structure

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
└── docker-compose.yml
```

---

## backend/Dockerfile

```dockerfile
# ══════════════════════════════════════
# STAGE 1 — Install dependencies
# ══════════════════════════════════════
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# System packages needed to compile Python deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages into /install (separate from system)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ══════════════════════════════════════
# STAGE 2 — Production image
# ══════════════════════════════════════
FROM python:3.12-slim AS production

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    DJANGO_SETTINGS_MODULE=myproject.settings.production

WORKDIR /app

# Only runtime system lib (no gcc needed in final image)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages from builder stage
COPY --from=builder /install /usr/local

# Copy app source code
COPY . .

# Create directories Django needs
RUN mkdir -p /app/staticfiles /app/media

# Non-root user for security
RUN addgroup --system django && adduser --system --ingroup django django
RUN chown -R django:django /app
USER django

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

CMD ["gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "120", \
     "--access-logfile", "-"]
```

---

## backend/requirements.txt

```
django==5.0.4
djangorestframework==3.15.1
psycopg2-binary==2.9.9
gunicorn==22.0.0
python-decouple==3.8
django-cors-headers==4.3.1
whitenoise==6.6.0
celery==5.3.6
redis==5.0.3
```

---

## frontend/Dockerfile

```dockerfile
# ══════════════════════════════════════
# STAGE 1 — Build React app
# ══════════════════════════════════════
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies first (caching)
COPY package.json package-lock.json ./
RUN npm ci

# Build the production bundle
COPY . .
ARG VITE_API_URL=/api
ENV VITE_API_URL=${VITE_API_URL}
RUN npm run build


# ══════════════════════════════════════
# STAGE 2 — Serve with Nginx
# ══════════════════════════════════════
FROM nginx:alpine AS production

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/

# Copy the built React files
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## frontend/nginx.conf

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # All routes → React Router
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to Django
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /static/ { proxy_pass http://backend:8000; }
    location /media/  { proxy_pass http://backend:8000; }
}
```

---

## ◈ DOCKERFILE CHECKLIST

Before you ship to production:

```
□  Using slim or alpine base image (not full ubuntu)
□  Multi-stage build for compiled apps
□  requirements.txt / package.json copied BEFORE source code
□  --no-cache-dir on pip install
□  rm -rf /var/lib/apt/lists/* after apt-get
□  Non-root USER set before CMD
□  .dockerignore file exists
□  EXPOSE port declared
□  HEALTHCHECK added
□  CMD uses JSON array format (not shell string)
□  No secrets hardcoded in the Dockerfile
```