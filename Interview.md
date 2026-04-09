# 🐳 Docker Interview Questions

> Real questions. Human answers. Easy to explain, professional to deliver.

---

# ━━━ BEGINNER LEVEL ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

**Q1. What is Docker? Why do we use it?**

Docker is a containerization platform that lets you package your app with everything it needs — the runtime, libraries, and config — into a container that runs the same way on any machine.

The real value is **consistency**. The classic problem is "it works on my machine but breaks in production." Docker eliminates that because you're shipping the exact same environment everywhere — dev, CI, and production all run identical containers.

---

**Q2. What is the difference between a Container and a Virtual Machine?**

A VM runs a full operating system with its own kernel. It's heavy — takes minutes to boot, uses GBs of disk, and each VM needs its own OS installed.

A container shares the host OS kernel. It only packages your app and its dependencies. Containers start in seconds, use MBs, and you can run dozens on one machine.

A simple way to explain it: a VM is like renting a separate house. A container is like renting a room in a shared house. You have your own space, but you share the foundation.

---

**Q3. What is the difference between a Docker Image and a Container?**

An image is a static blueprint — read-only, like a template or a class definition in code. A container is a running instance of that image — it's alive, it can read and write, it does actual work.

You can run 10 containers from the same image, just like you can create 10 objects from one class.

---

**Q4. What is a Dockerfile?**

A Dockerfile is a plain text file with step-by-step instructions to build a Docker image. You define the base OS, install dependencies, copy your code, and specify what command to run when the container starts. Docker reads it top to bottom, executing each instruction as a layer.

---

**Q5. What is Docker Hub?**

Docker Hub is the default public registry where Docker images are stored and shared. When you run `docker pull nginx`, it fetches it from Docker Hub. You can push your own images there too — public or private. Companies often use private registries like AWS ECR, GitHub Container Registry, or GCP Artifact Registry instead of Docker Hub in production.

---

**Q6. What are the basic Docker commands you use every day?**

```
docker pull         → download an image
docker images       → list local images
docker run          → start a container
docker run -it      → start with interactive terminal
docker run -d       → start in background
docker ps           → see running containers
docker ps -a        → see all containers including stopped
docker stop         → stop a container
docker start        → start a stopped container
docker rm           → delete a container
docker rmi          → delete an image
docker logs -f      → follow live logs
docker exec -it     → enter a running container shell
```

---

# ━━━ INTERMEDIATE LEVEL ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

**Q7. What does the `-d` flag do in `docker run`?**

`-d` stands for detached. Without it, the container runs in your terminal and you lose it if you close the session. With `-d`, it runs in the background and returns the container ID. In production, you always use `-d`.

---

**Q8. What does `-it` mean in `docker run -it`?**

`-i` keeps STDIN open so you can type input. `-t` allocates a pseudo-terminal so you get a proper shell experience. Together, they give you an interactive terminal inside the container — exactly what you need when you want to explore, debug, or run commands inside.

---

**Q9. What is port binding and how does it work?**

Containers are isolated — their ports aren't visible to the outside by default. Port binding maps a port on your host machine to a port inside the container.

`docker run -p 8080:80 nginx` means: any traffic hitting your machine's port 8080 gets forwarded to port 80 inside the container. The format is always `host:container`.

---

**Q10. What is `docker stop` vs `docker kill`?**

`docker stop` is graceful — it sends SIGTERM, gives the app 10 seconds to finish what it's doing (finish requests, flush data), then sends SIGKILL. `docker kill` is immediate — no grace period, SIGKILL straight away.

Always use `stop` in production. Your app might be in the middle of processing something important.

---

**Q11. How do you debug a container that keeps restarting?**

First, `docker ps -a` to confirm it's crash-looping. Then `docker logs <container>` — the last error output is usually there. Common causes are missing environment variables, can't connect to the database, port conflict, or a crash in the startup command. If I need to get inside the container to explore, I override the command with `sh` to prevent the crash: `docker run -it myimage sh`.

---

**Q12. What is a bind mount vs a named volume?**

A bind mount maps a specific folder on your host machine directly into the container — `./mycode:/app`. It's great for development because code changes reflect immediately.

A named volume is managed entirely by Docker — Docker decides where the data lives on the host. It's better for production because it's more portable, performs better on Linux, and persists independently of any container.

---

**Q13. What is the purpose of `EXPOSE` in a Dockerfile?**

`EXPOSE` is documentation — it signals which port the app listens on, but it doesn't actually open or publish the port. You still need `-p` in `docker run` to make it accessible from outside. Think of `EXPOSE` as a label. The `-p` flag is what actually does the work.

---

**Q14. What are the Dockerfile instructions you use most?**

```
FROM        → base image (OS or runtime)
WORKDIR     → set working directory inside container
COPY        → copy files from host to container
RUN         → run shell commands during build (installs, builds)
ENV         → set environment variables
EXPOSE      → document the port
CMD         → default command to run when container starts
```

---

# ━━━ EXPERIENCED LEVEL ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

**Q15. How does Docker layer caching work?**

Every instruction in a Dockerfile creates a layer. Docker caches each layer by its content hash. When you rebuild, it reuses cached layers until it hits something that changed — then everything below that runs again from scratch.

This is why order matters. If you `COPY . .` before `pip install`, then every code change triggers a full dependency reinstall. The fix is simple: copy dependency files first, install, then copy source code. That way pip only reruns when `requirements.txt` actually changes.

---

**Q16. What is a multi-stage build and why is it important?**

Multi-stage builds use multiple `FROM` statements in one Dockerfile. You do all your heavy building — compiling, installing dev tools, running build scripts — in an early stage. Then you copy only the final output into a clean, minimal final image.

Why it matters: the production image doesn't need gcc, build tools, or dev packages. A well-done multi-stage build can take an 800MB image down to 120MB. Smaller image = faster pulls, less attack surface, lower cloud storage costs.

---

**Q17. What is the difference between CMD and ENTRYPOINT?**

`ENTRYPOINT` is the fixed executable — it always runs and you can't easily override it. `CMD` provides the default arguments, which can be overridden when you do `docker run`.

Combined example: `ENTRYPOINT ["python", "manage.py"]` and `CMD ["runserver"]`. Default run is `python manage.py runserver`. But `docker run myapp migrate` becomes `python manage.py migrate` — CMD was overridden, ENTRYPOINT stayed.

For most web apps, `CMD` alone is fine. Use `ENTRYPOINT` when the container should always behave like a specific tool.

---

**Q18. What is Docker Compose and when do you use it?**

Docker Compose lets you define and run multi-container applications with a single YAML file and one command. Instead of manually running five `docker run` commands in the right order, you declare all your services — app, database, cache, worker — and `docker compose up -d` handles everything.

It's essential for local development and perfectly good for small to medium production deployments. When you need to scale across multiple machines, that's when Kubernetes enters the picture.

---

**Q19. What does `depends_on` do? Does it guarantee the database is ready?**

`depends_on` controls startup order — Docker starts services in sequence. But by default, it only waits for the container to **start**, not for the service inside to be **ready**. Postgres might still be initializing when your app tries to connect.

The proper way is to add healthchecks to your database service and use `condition: service_healthy` in `depends_on`. That way, your backend only starts after the database is genuinely accepting connections.

---

**Q20. How do you handle secrets and environment variables in production?**

Never hardcode secrets in the Dockerfile or compose file. Use a `.env` file alongside your compose file — Compose picks it up automatically, and you reference variables with `${VAR_NAME}`. The `.env` file goes in `.gitignore`.

For proper production, you'd use Docker Secrets, HashiCorp Vault, or cloud-native solutions like AWS Secrets Manager. The compose `.env` approach is fine for small deployments, but shouldn't be used if multiple team members share production server access.

---

**Q21. Why should containers not run as root?**

If a container is compromised while running as root, the attacker can potentially escape to the host system with root-equivalent privileges. By creating a non-root user in your Dockerfile and switching to it with `USER`, you limit the blast radius of any security breach.

In practice, most Kubernetes environments enforce this at the cluster level — they reject pods that run as root. It's a baseline security requirement for any production workload.

---

**Q22. How do you reduce Docker image size?**

A few things that make a real difference:

- Use `slim` or `alpine` base images instead of full OS images
- Use multi-stage builds so build tools never land in the final image
- Chain `RUN` commands with `&&` to reduce layer count
- Add `--no-cache-dir` to pip install and clean up after apt-get
- Use `.dockerignore` to avoid copying node_modules, .git, logs, etc.

Going from 800MB down to 100-150MB is very achievable with these steps.

---

**Q23. What happens when you run `docker compose down -v`?**

It stops and removes all containers and networks created by Compose, and the `-v` flag also deletes all named volumes. Your database data is gone. This is useful for a clean reset in development, but you should never run it in production without a backup. It's one of those commands where the `-v` flag is easy to forget is destructive.

---

**Q24. How is Docker different from Kubernetes?**

Docker (and Compose) manages containers on a **single machine**. Kubernetes is a container orchestration platform that manages containers across a **cluster of machines**.

Kubernetes adds automatic scaling, self-healing (restarting failed containers), rolling deployments, cross-node load balancing, and complex networking. But it also comes with significant operational complexity.

For a small startup or internal tooling — Docker Compose on a VPS is totally valid. When you need high availability, horizontal scaling, or you're managing 50+ services across multiple servers — that's when Kubernetes pays off.

---

**Q25. What is a Docker network and why does it matter?**

Docker networks let containers talk to each other by name. When containers are on the same Docker network, you don't need to know each other's IP addresses — Docker has built-in DNS. So if my Django app needs to reach a Postgres container named `db`, I just set `DB_HOST=db` and Docker resolves it automatically.

In Compose, a default network is created for all services. But it's better to define custom networks explicitly so you can isolate services — for example, keeping your database only reachable by the backend, not exposed to every container.

---

# ━━━ SCENARIO QUESTIONS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

**"How do you deploy a new version with zero downtime?"**

With Docker Compose on a single server, you build the new image, push it, then do `docker compose up -d --build`. With multiple replicas and a load balancer (like Nginx) in front, you update containers in batches — take one offline, update it, wait for it to become healthy, move to the next.

For real zero-downtime deployments at scale, Kubernetes handles this natively with rolling updates — it spins up new pods before killing old ones, and only routes traffic to healthy instances.

---

**"How do you persist data in Docker?"**

Using volumes. For a database, you mount a named volume to the data directory — for Postgres that's `/var/lib/postgresql/data`. Named volumes survive container restarts, stops, and even removals. The data lives on the host, managed by Docker.

Never store important data inside the container layer — that layer is ephemeral. It disappears when the container is removed.

---

**"A container is running but the app inside is broken. How do you debug?"**

First, `docker logs -f myapp` — most errors are right there. If I need more, `docker exec -it myapp bash` to get inside and inspect files, check env variables with `env`, or manually run parts of the startup command to see where it fails. `docker inspect myapp` gives me the full config to verify mounts, network, and env vars are what I expected.

---

> 💡 **Interview tip** — Always explain the WHY, not just the WHAT.  
> Don't say "I use named volumes." Say "I use named volumes because container storage is ephemeral — data disappears when the container is removed, and volumes persist independently of any container lifecycle."