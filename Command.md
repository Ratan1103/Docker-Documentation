# 🐳 Docker Commands

> Start simple → grow with it. Every command you need, in the order you'll need it.

---

## ◈ UNDERSTAND THIS FIRST

```
You write a Dockerfile
      ↓
Docker builds it into an IMAGE   ← like a zip / snapshot of your app
      ↓
You run that IMAGE → becomes a CONTAINER   ← the actual running app
```

That's the whole model. Everything else is commands around this.

---

# ━━━ BEGINNER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 1 · Pull an Image

```bash
docker pull IMAGE_NAME
```

Downloads an image from Docker Hub to your machine.

```bash
docker pull nginx
docker pull postgres
docker pull python
```

Pull a **specific version**:

```bash
docker pull IMAGE_NAME:version

docker pull nginx:alpine
docker pull postgres:16
docker pull python:3.12-slim
```

> 💡 No version = Docker pulls `latest` automatically.

---

## 2 · See Your Images

```bash
docker images
```

Lists everything you've downloaded or built locally.

```
REPOSITORY   TAG       IMAGE ID       SIZE
nginx        alpine    abc123def      45MB
postgres     16        456ghi789      380MB
```

---

## 3 · Run a Container

```bash
docker run IMAGE_NAME

docker run nginx
```

Starts a container in your terminal (foreground). `Ctrl+C` to stop.

---

## 4 · Run With an Interactive Terminal

```bash
docker run -it IMAGE_NAME

docker run -it ubuntu bash
```

`-i` → keep STDIN open so you can type  
`-t` → give you a real terminal  
Together → you get a shell **inside** the container

```bash
root@a1b2c3:/# ls
root@a1b2c3:/# exit     ← leave the container
```

---

## 5 · Stop and Start a Container

```bash
docker stop CONT_NAME or CONT_ID
docker start CONT_NAME or CONT_ID

docker stop my-nginx
docker start my-nginx
```

`stop` → graceful (lets the app finish what it's doing)  
`start` → brings it back up without recreating it

---

## 6 · See Running Containers

```bash
docker ps          # Only running containers
docker ps -a       # ALL containers — running AND stopped
```

---

## 7 · Delete Container and Image

```bash
docker rm CONT_NAME      # Delete a container
docker rmi IMAGE_NAME    # Delete an image

docker rm my-nginx
docker rmi nginx
```

> ⚠️ Stop the container first before removing it.  
> Or force it: `docker rm -f my-nginx`

---

# ━━━ INTERMEDIATE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 8 · Run in Background (Detached)

```bash
docker run -d IMAGE_NAME

docker run -d nginx
```

`-d` → runs silently in the background, gives your terminal back.

---

## 9 · Give Your Container a Name

```bash
docker run --name CONT_NAME -d IMAGE_NAME

docker run --name my-nginx -d nginx
```

Now you use `my-nginx` instead of a random ID everywhere.

---

## 10 · Port Binding — Reach the App From Your Browser

Containers are **isolated by default** — you can't access them from your browser until you map the port.

```bash
docker run -p HOST_PORT:CONTAINER_PORT IMAGE_NAME

docker run -p 8080:80 nginx
```

```
Your browser → localhost:8080
                    ↓  Docker maps it
              Container's port 80
```

```bash
docker run -p 3000:3000 node-app
docker run -p 8000:8000 django-app
docker run -p 8080:80 -p 443:443 nginx    # Multiple ports
```

---

## 11 · Environment Variables

Pass configuration into the container at runtime.

```bash
docker run -e KEY=VALUE IMAGE_NAME

docker run -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=mydb postgres
```

---

## 12 · Mount a Volume — Don't Lose Your Data

When a container is deleted, its data is gone. Volumes keep data alive.

```bash
docker run -v HOST_PATH:CONTAINER_PATH IMAGE_NAME

docker run -v /mydata:/var/lib/postgresql/data postgres
docker run -v $(pwd):/app myapp           # Current folder ↔ /app in container
```

---

## 13 · The Full docker run — Real World Example

```bash
docker run -d \
  --name my-postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v pgdata:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16
```

---

## 14 · Restart Policies

```bash
--restart no                # Never restart (default)
--restart always            # Always restart no matter what
--restart on-failure        # Only if container crashed (non-zero exit)
--restart unless-stopped    # Always restart, unless YOU manually stop it  ✅ use in prod
```

---

# ━━━ TROUBLESHOOTING ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 15 · Read Container Logs

```bash
docker logs CONT_ID

docker logs my-postgres                   # All logs
docker logs -f my-postgres                # -f → follow live (like tail -f)
docker logs --tail 100 my-postgres        # Last 100 lines
docker logs --since 1h my-postgres        # Logs from last 1 hour
```

---

## 16 · Enter a Running Container

```bash
docker exec -it CONT_ID /bin/bash         # If bash is available
docker exec -it CONT_ID /bin/sh           # Fallback for alpine images

docker exec -it my-postgres bash
docker exec -it my-nginx sh
```

> This is your go-to for debugging — inspect files, run commands, check what's happening inside.

---

## 17 · Stats and Inspection

```bash
docker stats                        # Live CPU/RAM for ALL containers
docker stats my-postgres            # Stats for one
docker inspect my-postgres          # Full config as JSON
docker top my-postgres              # Processes inside the container
```

---

# ━━━ EXPERIENCED / PRODUCTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 18 · Build Your Own Image

```bash
docker build .                               # Build from current folder
docker build -t myapp:1.0 .                  # Name + tag the image
docker build -t myapp:latest -f Dockerfile.prod .   # Custom Dockerfile name
docker build --no-cache -t myapp .           # Fresh build, skip all cache
```

---

## 19 · Tag and Push to a Registry

```bash
docker tag myapp:latest myrepo/myapp:1.0    # Rename for registry
docker push myrepo/myapp:1.0                # Push to Docker Hub
docker pull myrepo/myapp:1.0               # Pull on another machine
```

---

## 20 · Copy Files Between Host and Container

```bash
docker cp CONT_NAME:/path/file ./local         # Container → Host
docker cp ./local-file CONT_NAME:/path/file    # Host → Container

docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf
```

---

## 21 · Cleanup

```bash
docker rm -f $(docker ps -aq)         # Delete ALL containers
docker rmi -f $(docker images -q)     # Delete ALL images
docker volume prune                   # Delete all unused volumes
docker system prune -a                # Full cleanup (images + containers + networks)
docker system df                      # See how much disk Docker is using
```

---

## ◈ COMMAND CHEAT SHEET

| What you want | Command |
|---|---|
| Download image | `docker pull nginx` |
| Download specific version | `docker pull nginx:alpine` |
| List local images | `docker images` |
| Run in background | `docker run -d nginx` |
| Interactive shell | `docker run -it ubuntu bash` |
| Name your container | `docker run --name myapp -d nginx` |
| Map a port | `docker run -p 8080:80 nginx` |
| Set env variable | `docker run -e KEY=val image` |
| Mount a volume | `docker run -v /host:/cont image` |
| See running containers | `docker ps` |
| See ALL containers | `docker ps -a` |
| Stop container | `docker stop myapp` |
| Delete container | `docker rm myapp` |
| Delete image | `docker rmi nginx` |
| Read logs | `docker logs -f myapp` |
| Enter shell | `docker exec -it myapp bash` |
| Full cleanup | `docker system prune -a` |

---

## ◈ FLAGS AT A GLANCE

| Flag | Full Name | What it does |
|---|---|---|
| `-d` | detach | Run in background |
| `-i` | interactive | Keep input open |
| `-t` | tty | Allocate a terminal |
| `-it` | both | Interactive shell inside container |
| `-p` | publish | Map port host:container |
| `-v` | volume | Mount host folder or named volume |
| `-e` | env | Set environment variable |
| `-f` | force / file | Force delete OR specify custom Dockerfile |
| `-a` | all | Include stopped containers |
| `--name` | — | Give container a readable name |
| `--rm` | — | Auto-delete container after it exits |
| `--restart` | — | Restart policy |
| `--no-cache` | — | Rebuild without using cached layers |