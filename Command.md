# 🐳 Docker Commands — Complete Reference

> **Production-ready cheat sheet** · All commands · All flags · Real usage

---

## ◈ THE DOCKER MENTAL MODEL

```
YOUR CODE
    │
    ▼
┌─────────────────────────────────────┐
│           Dockerfile                │  ← Blueprint / Recipe
└─────────────────┬───────────────────┘
                  │  docker build
                  ▼
┌─────────────────────────────────────┐
│              IMAGE                  │  ← Snapshot / Template (read-only)
└─────────────────┬───────────────────┘
                  │  docker run
                  ▼
┌─────────────────────────────────────┐
│            CONTAINER                │  ← Running instance (read-write)
└─────────────────────────────────────┘
```

---

## ◈ IMAGE COMMANDS

```bash
# ── Build ──────────────────────────────────────────────────────────────────
docker build .                          # Build from current dir
docker build -t myapp:1.0 .             # -t → tag it with name:version
docker build -t myapp:latest -f prod.Dockerfile .   # -f → custom Dockerfile name
docker build --no-cache -t myapp .      # --no-cache → fresh build, skip cache

# ── List / Inspect ──────────────────────────────────────────────────────────
docker images                           # List all local images
docker images -a                        # -a → include intermediate layers
docker image inspect myapp:latest       # Full metadata of image
docker history myapp:latest             # Layer-by-layer build history

# ── Remove ──────────────────────────────────────────────────────────────────
docker rmi myapp:latest                 # Remove image
docker rmi -f myapp:latest              # -f → force remove (even if container uses it)
docker image prune                      # Remove all dangling (untagged) images
docker image prune -a                   # -a → remove ALL unused images

# ── Pull / Push ─────────────────────────────────────────────────────────────
docker pull nginx:alpine                # Pull from Docker Hub
docker push myrepo/myapp:1.0            # Push to registry
docker tag myapp:latest myrepo/myapp:1.0  # Rename/tag before push
```

---

## ◈ CONTAINER COMMANDS

### `docker run` — The Most Important Command

```
docker run [OPTIONS] IMAGE [COMMAND]
              │
              ├── -d          → detached (background)
              ├── -it         → interactive terminal
              ├── -p          → port mapping   host:container
              ├── -v          → volume mount   host:container
              ├── -e          → environment variable
              ├── --name      → give container a name
              ├── --rm        → auto-delete when stopped
              ├── --network   → connect to network
              └── --restart   → restart policy
```

```bash
# ── Core Flags with Examples ────────────────────────────────────────────────

docker run nginx
# → Runs nginx, output in terminal, exits on Ctrl+C

docker run -d nginx
# -d → detached, runs in background, returns container ID

docker run -it ubuntu bash
# -i → keep STDIN open  
# -t → allocate a pseudo-TTY (terminal)
# Together: you get an interactive shell inside the container

docker run -it --rm ubuntu bash
# --rm → container is deleted automatically after you exit

docker run -d -p 8080:80 nginx
# -p host_port:container_port → access nginx at localhost:8080

docker run -d -p 8000:8000 -p 5555:5555 myapp
# Multiple -p flags for multiple ports

docker run -d --name my-nginx nginx
# --name → give it a readable name instead of random one

docker run -d \
  --name myapp \
  -p 8000:8000 \
  -e DEBUG=True \
  -e DATABASE_URL=postgres://... \
  -v $(pwd):/app \
  --restart unless-stopped \
  myapp:latest
# Full production-like run command
```

### Restart Policies

```
--restart no              → Never restart (default)
--restart always          → Always restart
--restart on-failure      → Restart only if exits with error
--restart unless-stopped  → Always restart unless manually stopped ✅ (use this in prod)
```

### Managing Containers

```bash
# ── Status ──────────────────────────────────────────────────────────────────
docker ps                   # Running containers
docker ps -a                # -a → all containers (including stopped)
docker ps -q                # -q → only container IDs (quiet)

# ── Control ─────────────────────────────────────────────────────────────────
docker stop myapp           # Graceful stop (SIGTERM → waits → SIGKILL)
docker kill myapp           # Immediate stop (SIGKILL)
docker start myapp          # Start a stopped container
docker restart myapp        # Stop + Start
docker pause myapp          # Freeze container (no CPU)
docker unpause myapp        # Resume frozen container

# ── Remove ──────────────────────────────────────────────────────────────────
docker rm myapp             # Remove stopped container
docker rm -f myapp          # -f → force remove (even if running)
docker rm $(docker ps -aq)  # Remove ALL stopped containers

# ── Inspect / Debug ─────────────────────────────────────────────────────────
docker logs myapp           # View logs
docker logs -f myapp        # -f → follow (live tail)
docker logs --tail 100 myapp     # Last 100 lines
docker logs --since 1h myapp     # Logs from last 1 hour

docker inspect myapp        # Full JSON config of container
docker stats                # Live CPU/RAM/Network usage of ALL containers
docker stats myapp          # Stats for one container
docker top myapp            # Processes running inside container

docker exec myapp ls /app              # Run command inside running container
docker exec -it myapp bash             # Enter running container shell
docker exec -it myapp sh               # Use sh if bash not available
docker exec -it -e DEBUG=1 myapp bash  # With extra env var
```

---

## ◈ VOLUME COMMANDS

```
HOST FILESYSTEM ◄──────────────────────► CONTAINER FILESYSTEM
/data/myapp/db             -v            /var/lib/postgresql/data
```

```bash
# ── Named Volumes ───────────────────────────────────────────────────────────
docker volume create mydata             # Create named volume
docker volume ls                        # List all volumes
docker volume inspect mydata            # Show where data lives on host
docker volume rm mydata                 # Delete volume
docker volume prune                     # Delete all unused volumes

# ── Using Volumes in Run ─────────────────────────────────────────────────────
docker run -v mydata:/app/data myapp           # Named volume mount
docker run -v $(pwd):/app myapp                # Bind mount (host dir → container)
docker run -v $(pwd):/app:ro myapp             # :ro → read-only inside container
```

### Volume Types

```
Bind Mount  →  -v /host/path:/container/path   (syncs a host folder)
Named Volume→  -v myvolume:/container/path     (Docker manages the folder)
tmpfs       →  --tmpfs /tmp                    (in-memory, not persisted)
```

---

## ◈ NETWORK COMMANDS

```
                        DOCKER NETWORK
                   ┌──────────────────────┐
  ┌────────────┐   │  ┌──────┐ ┌──────┐  │
  │  Internet  │◄──┼──│  db  │ │  app │  │
  └────────────┘   │  └──┬───┘ └───┬──┘  │
                   │     └────┬────┘      │
                   │          │           │
                   └──────────┼───────────┘
                              │
                    containers talk by name
```

```bash
docker network create mynet             # Create bridge network
docker network ls                       # List networks
docker network inspect mynet           # Inspect network
docker network rm mynet                 # Remove network

docker run --network mynet myapp        # Connect container to network
docker network connect mynet myapp      # Connect running container to network
docker network disconnect mynet myapp   # Disconnect
```

### Network Drivers

```
bridge    → Default. Containers on same host can communicate
host      → Container uses host's network directly (no isolation)
none      → No networking
overlay   → Multi-host networking (Swarm/Kubernetes)
```

---

## ◈ REGISTRY COMMANDS

```bash
docker login                            # Login to Docker Hub
docker login ghcr.io                   # Login to GitHub Container Registry
docker logout

docker pull redis:7-alpine             # Pull specific tag
docker push myrepo/myapp:prod          # Push to registry

# ── Tagging Strategy ────────────────────────────────────────────────────────
docker tag myapp:latest myrepo/myapp:latest
docker tag myapp:latest myrepo/myapp:1.0.0
docker tag myapp:latest myrepo/myapp:stable
```

---

## ◈ SYSTEM / CLEANUP

```bash
docker system df            # Show disk usage (images, containers, volumes)
docker system prune         # Remove all stopped containers + dangling images
docker system prune -a      # -a → also remove unused images (big cleanup)
docker system prune -a -f   # -f → skip confirmation prompt

# Individual prunes
docker container prune      # All stopped containers
docker image prune -a       # All unused images
docker volume prune         # All unused volumes
docker network prune        # All unused networks
```

---

## ◈ COPY FILES

```bash
docker cp myapp:/app/logs/error.log ./error.log   # Container → Host
docker cp ./config.json myapp:/app/config.json    # Host → Container
```

---

## ◈ QUICK REFERENCE TABLE

| Task | Command |
|------|---------|
| Run in background | `docker run -d IMAGE` |
| Interactive shell | `docker run -it IMAGE bash` |
| Map port | `docker run -p HOST:CONTAINER IMAGE` |
| Set env variable | `docker run -e KEY=VALUE IMAGE` |
| Mount volume | `docker run -v /host:/container IMAGE` |
| Enter running container | `docker exec -it NAME bash` |
| View live logs | `docker logs -f NAME` |
| Stop all containers | `docker stop $(docker ps -q)` |
| Remove all containers | `docker rm $(docker ps -aq)` |
| Full system cleanup | `docker system prune -a` |

---

## ◈ FLAG GLOSSARY

| Flag | Full Name | What it does |
|------|-----------|--------------|
| `-d` | `--detach` | Run in background |
| `-i` | `--interactive` | Keep STDIN open |
| `-t` | `--tty` | Allocate a terminal |
| `-p` | `--publish` | Map port host:container |
| `-v` | `--volume` | Mount volume |
| `-e` | `--env` | Set environment variable |
| `-f` | `--force` / `--file` | Force action OR specify file |
| `-a` | `--all` | Include stopped/all |
| `-q` | `--quiet` | Only show IDs |
| `--rm` | — | Auto-remove after exit |
| `--name` | — | Assign container name |
| `--network` | — | Connect to network |
| `--restart` | — | Restart policy |
| `--no-cache` | — | Build without cache |

---

> 💡 **Pro Tip** — Combine `-it --rm` for throwaway debug containers:  
> `docker run -it --rm python:3.12 python` → instant Python REPL, gone after exit