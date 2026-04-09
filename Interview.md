# 🐳 Docker Interview Questions — Production Engineer Edition

> **Humanized answers that feel natural to explain**  
> Sorted by topic · From basics to advanced

---

## ◈ FUNDAMENTALS

---

**Q1. What is Docker and why do we use it?**

> Docker is a containerization platform. Instead of saying "it works on my machine," Docker lets you package your application with everything it needs — the runtime, libraries, config — into a container that runs the same way everywhere.
>
> The core value is **consistency**. Your dev environment, CI pipeline, and production server all run the exact same thing. No more "works locally but fails in prod."

---

**Q2. What's the difference between a container and a virtual machine?**

> A VM emulates an entire OS — it has its own kernel, its own memory, and takes minutes to boot. Heavy and resource-intensive.
>
> A container shares the host OS kernel. It only packages the app and its dependencies. So containers are **lightweight** (MBs vs GBs), start in seconds, and you can run dozens on the same machine.
>
> Think of it this way — a VM is like renting an entire apartment. A container is like renting a room in a shared house. Same building, shared foundation, but your own isolated space.

---

**Q3. What is a Docker Image vs a Docker Container?**

> An image is like a **blueprint** or a recipe — it's read-only and defines what the environment looks like. A container is a **running instance** of that image — it's alive, it can read/write, it does the actual work.
>
> One image can spin up 10 containers. Like one class definition can create 10 objects in OOP.

---

**Q4. What is a Dockerfile?**

> A Dockerfile is a text file with step-by-step instructions to build a Docker image. You define the base OS, install dependencies, copy your code, and specify what command to run. Docker reads this file top to bottom and creates your image layer by layer.

---

**Q5. What is Docker Hub?**

> Docker Hub is the default public registry where Docker images are stored and shared. When you do `docker pull nginx`, it fetches it from Docker Hub. You can also push your own images there — public or private. Companies often use private registries like AWS ECR, GitHub Container Registry, or GCP Artifact Registry instead.

---

## ◈ IMAGES & LAYERS

---

**Q6. How does Docker's layer caching work?**

> Every instruction in a Dockerfile creates a layer. Docker caches each layer. When you rebuild, it reuses cached layers that haven't changed.
>
> This is why order matters. If you `COPY . .` before `RUN pip install`, any code change invalidates the pip install layer — forcing a full reinstall every time.
>
> The trick is: copy dependency files first, install packages, then copy your code. That way, pip install only reruns when `requirements.txt` changes, not on every code edit.

---

**Q7. What is a multi-stage build and why use it?**

> Multi-stage builds let you use multiple `FROM` statements in one Dockerfile. You do all your heavy building in an early stage, then copy only the final output into a clean, slim final stage.
>
> Why? Your production image doesn't need gcc, build tools, or dev packages. With multi-stage, you can go from a 900MB image to a 120MB image because the final stage only has what's needed to run the app.

---

**Q8. What is the difference between CMD and ENTRYPOINT?**

> `ENTRYPOINT` is the fixed command that always runs — you can't override it easily. `CMD` provides default arguments that can be overridden when you run the container.
>
> In practice, for most web apps I use `CMD` alone for flexibility. But for a container meant to be a single-purpose tool — like a CLI — I use `ENTRYPOINT` so the container always behaves like that tool, and `CMD` for the default subcommand.

---

**Q9. What's the difference between COPY and ADD?**

> Both copy files into the image, but `ADD` has extra powers — it can fetch from a URL and auto-extract tar files. In practice, you should always use `COPY` unless you specifically need those features, because `COPY` is more predictable and explicit.

---

## ◈ CONTAINERS & RUNTIME

---

**Q10. What is the difference between `docker stop` and `docker kill`?**

> `docker stop` is graceful — it sends SIGTERM to the process, gives it 10 seconds to clean up, then sends SIGKILL. `docker kill` is immediate — it sends SIGKILL straight away, no grace period.
>
> Always use `stop` in production. Your app might need to finish handling a request or flush data to disk before shutting down.

---

**Q11. What is the `-d` flag and when do you use it?**

> `-d` stands for detached. Without it, the container runs in your terminal and you lose the session if you close it. With `-d`, it runs in the background and gives you back the terminal. In production, you almost always use `-d`.

---

**Q12. What does `-it` mean?**

> `-i` keeps STDIN open so you can type input. `-t` allocates a pseudo-terminal so you get a proper shell experience. Together, `-it` gives you an interactive terminal inside the container — exactly what you need when you do `docker run -it ubuntu bash` or `docker exec -it myapp sh`.

---

**Q13. How do containers communicate with each other?**

> Through Docker networks. When containers are on the same Docker network, they can talk to each other using the **container name** as the hostname — Docker's built-in DNS handles the resolution.
>
> So if my Django app needs to connect to a Postgres container named `db`, I just point `DATABASE_HOST=db` and Docker resolves it. No IP addresses needed.

---

**Q14. What is a bind mount vs a named volume?**

> A bind mount maps a specific folder from your host machine into the container. It's great for development — your code changes reflect instantly inside the container.
>
> A named volume is managed entirely by Docker. Docker decides where to store it on the host. It's better for production — database data, uploaded files — because it persists independently of the container and performs better on Linux.

---

## ◈ DOCKER COMPOSE

---

**Q15. What is Docker Compose and when do you use it?**

> Docker Compose lets you define and run multi-container applications using a single YAML file. Instead of running a dozen `docker run` commands manually, you define all your services — web app, database, cache, worker — and start everything with one `docker compose up`.
>
> It's essential for local development and perfectly good for small-to-medium production deployments. For large-scale production, you'd move to Kubernetes.

---

**Q16. What does `depends_on` do? Does it guarantee the database is ready?**

> `depends_on` controls startup order — it makes Docker start services in a certain sequence. But by itself, it only waits for the container to **start**, not for the service inside to be **ready**.
>
> Postgres takes a few seconds to initialize after the container starts. So you need to combine `depends_on` with healthchecks and use `condition: service_healthy`. That way, your app only starts after the database is actually accepting connections.

---

**Q17. How do you manage secrets in Docker Compose?**

> The right way is to use a `.env` file alongside your compose file. Compose picks it up automatically and you reference variables with `${VAR_NAME}`. The `.env` file stays out of version control — it's in `.gitignore`.
>
> For enterprise production, you'd use Docker Secrets, HashiCorp Vault, or cloud-native solutions like AWS Secrets Manager. Never hardcode passwords in the compose file.

---

## ◈ NETWORKING & SECURITY

---

**Q18. What are the Docker network drivers?**

> The main ones are bridge, host, and none.
>
> **Bridge** is the default — containers are isolated from the host but can talk to each other through the network. This is what you use 99% of the time.
>
> **Host** removes network isolation — the container uses the host's network stack directly. Slightly faster, but you lose port mapping and isolation.
>
> **None** means no networking at all — fully isolated.

---

**Q19. Why should you not run containers as root?**

> If a container is compromised and it's running as root, the attacker has root-equivalent access which can potentially break out to the host. By creating a non-root user in your Dockerfile and switching to it with `USER`, you limit the blast radius of any security issue.
>
> It's a security best practice that most production setups require — some Kubernetes policies won't even schedule pods that run as root.

---

**Q20. What is `docker system prune` and when would you use it?**

> It cleans up unused Docker resources — stopped containers, dangling images, unused networks. With `-a` it also removes unused images. I use it when disk space is getting low on a server or after a lot of development work has created a pile of orphaned images and containers.
>
> Be careful in production — always double check what it's going to delete.

---

## ◈ ADVANCED

---

**Q21. What is the difference between `docker compose up --build` and `docker compose build`?**

> `docker compose build` just rebuilds the images without starting anything. `docker compose up --build` rebuilds and then starts all containers. I typically use `up --build` when I've changed the Dockerfile or requirements and want to redeploy immediately.

---

**Q22. How do you handle database migrations in a Docker + Django setup?**

> A clean pattern is to run the migration as part of the startup command in the `backend` service:
> ```
> command: sh -c "python manage.py migrate && gunicorn ..."
> ```
> This ensures migrations always run before the server starts. The `depends_on` with `service_healthy` on the db ensures the database is ready when migrate runs.
>
> For more complex setups, I use an init container or a separate migration service that runs and exits before the backend starts.

---

**Q23. What is a `.dockerignore` file?**

> It works exactly like `.gitignore` — it tells Docker what to exclude when it copies files into the image. Without it, Docker might copy `node_modules`, `.git`, `.env`, and all your local cache files into the image, making it unnecessarily large and slow to build.
>
> Always include `.dockerignore` with your Dockerfile. It's not optional in production.

---

**Q24. How would you reduce Docker image size?**

> A few key things:
> - Use slim or alpine base images instead of full OS images
> - Use multi-stage builds so build tools don't end up in production
> - Chain `RUN` commands with `&&` to reduce layers
> - Add `--no-cache-dir` to pip and clean up after apt-get
> - Use `.dockerignore` to avoid copying unnecessary files
>
> Going from 800MB to 120MB is realistic with these changes.

---

**Q25. What's the difference between EXPOSE and publishing a port with `-p`?**

> `EXPOSE` in a Dockerfile is just documentation — it signals which port the app listens on, but it doesn't actually make it accessible from outside.
>
> `-p 8000:8000` in `docker run` (or `ports:` in compose) actually opens the port and maps it to the host. That's what makes it reachable.
>
> Think of `EXPOSE` as a label on the box saying "this side up." The `-p` flag is you actually opening the box.

---

**Q26. How do you check if a container is healthy?**

> Docker has a native `HEALTHCHECK` instruction. You define a command to run periodically — like curling a `/health` endpoint — and Docker marks the container as `healthy`, `unhealthy`, or `starting`. You can see the status in `docker ps`. Compose can use this to manage `depends_on` ordering.

---

**Q27. What happens when you do `docker compose down -v`?**

> It stops and removes all containers, removes the networks, and the `-v` flag also deletes all named volumes. This is destructive — your database data is gone. Useful for a clean reset in dev, dangerous in production. Always backup before using `-v` anywhere important.

---

**Q28. How is Docker different from Kubernetes?**

> Docker (and Compose) manages containers on a single machine. Kubernetes is an orchestration platform that manages containers across a cluster of machines.
>
> Kubernetes handles things like automatic scaling, self-healing (restarting failed containers), rolling deployments, load balancing across nodes, and cluster networking. It's much more complex to set up and operate.
>
> For a small startup or internal tools, Docker Compose on a VPS is totally fine. When you need high availability, auto-scaling, or you're managing 50+ services across multiple servers — that's when Kubernetes makes sense.

---

## ◈ QUICK SCENARIO QUESTIONS

---

**"Your container keeps restarting. How do you debug it?"**

> First, `docker ps -a` to confirm it's restarting. Then `docker logs <container>` to see the last error output — usually it's a missing env variable, can't connect to the database, or a crash in the startup command. If I need to get inside it, I'd override the command with `sh` to prevent the crash and explore the filesystem manually.

---

**"How do you deploy a new version with zero downtime?"**

> In a simple Docker Compose setup, you'd build and push the new image, then do a rolling update. With Compose, `docker compose up -d --build` will recreate containers one by one if you have multiple replicas.
>
> For real zero downtime, you'd use a load balancer (like Nginx) in front, update containers in batches, and only route traffic to healthy containers. At scale, Kubernetes handles this natively with rolling deployments.

---

**"How do you persist data in Docker?"**

> Using volumes. For a database, you'd mount a named volume to the data directory — like `/var/lib/postgresql/data`. Named volumes survive container restarts and removals. The data lives on the host managed by Docker. Never store important data inside the container layer — it's ephemeral.

---

> 💡 **Interview tip** — Interviewers love when you mention the WHY.  
> Don't just say "I use `-d`" — say "I use `-d` so it runs in the background without blocking the terminal, which is important in production."