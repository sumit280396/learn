# Docker Interview Preparation Guide
### Container Creation, Management & Troubleshooting — From Zero to Confident

---

## How to use this guide

This is built in layers, like Docker itself:

1. **Part 0** gives you the *mental model* — the picture in your head that everything else hangs off. Read this even if you're in a hurry. Without it, commands are just magic spells you memorize and forget.
2. **Parts 1–9** cover concepts + interview questions + commands, grouped by theme (images, containers, networking, storage, compose, resources, security, logs).
3. **Part 10** is the big one — **scenario-based troubleshooting**. Each scenario gives you a *reusable approach*, not just an answer, so you can handle a scenario you've never seen before.
4. **Part 11** is a command cheat sheet.
5. **Part 12** is rapid-fire Q&A for last-minute review.

Every answer is written twice, effectively: once as "what to say in the interview," and once as "why this is true," so you're not just reciting — you actually understand it.

---

# PART 0: The Mental Model (read this first)

### What problem does Docker actually solve?

Imagine you write an app on your laptop. It works. You give it to a teammate — it breaks, because they have a different OS version, a different Python version, a missing library. This is the **"it works on my machine"** problem.

Docker solves this by packaging your app **together with everything it needs to run** (code, runtime, libraries, system tools, settings) into one unit called a **container**. That container runs *identically* on your laptop, your teammate's laptop, and a server in the cloud — because it's not relying on whatever happens to be installed on the host machine. It carries its own little world with it.

### Container vs Virtual Machine — the analogy that unlocks everything

- A **Virtual Machine (VM)** is like building a separate house for every tenant — each house has its own foundation, plumbing, electrical system (its own full OS kernel). Very isolated, but heavy and slow to build.
- A **Container** is like an apartment in one building — each apartment (container) has its own rooms, furniture, and door lock (its own filesystem, processes, network), but they all share the same building foundation and plumbing (the host machine's OS kernel).

**Why this matters technically:** containers don't boot a full OS. They share the host's kernel and just get an isolated *view* of processes, network, and filesystem using Linux kernel features called **namespaces** (isolation) and **cgroups** (resource limiting). This is *why* containers start in milliseconds/seconds while VMs take minutes, and why containers are lightweight (megabytes) while VMs are heavy (gigabytes).

> **Interview one-liner:** "Containers virtualize the OS; VMs virtualize the hardware."

### The four nouns you must never confuse

This confusion is the #1 source of beginner panic. Get this straight and 50% of Docker becomes obvious:

| Term | What it is | Real-world analogy |
|---|---|---|
| **Dockerfile** | A text file with instructions on how to build an image | A recipe |
| **Image** | A read-only template/snapshot built from a Dockerfile | A frozen meal you can reheat anytime |
| **Container** | A running (or stopped) instance of an image | The actual meal, plated and being eaten |
| **Registry** (e.g. Docker Hub) | A place to store and share images | The supermarket where frozen meals are stocked |

You write a **Dockerfile** → you `build` it into an **image** → you `run` the image to get a **container** → you can `push` the image to a **registry** to share it.

One image can produce many containers, the same way one cookie cutter (image) can stamp out many cookies (containers). Each container has its own writable layer on top, so changes in one container don't affect another, or the original image.

### The layered filesystem mental model

Docker images are built in **layers**, stacked like transparent sheets on an overhead projector. Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, etc.) creates a new layer. Layers are cached and reused — this is *why* Docker builds can be fast, and *why* the order of instructions in a Dockerfile matters (explained in Part 2).

When a container runs, Docker adds one more layer on top: a **thin writable layer**. All the layers below (from the image) stay read-only. This is why:
- Multiple containers can share the same image layers efficiently (saves disk space).
- If you delete a container, the writable layer goes with it — **any data written inside a container is lost when the container is removed**, unless you explicitly persisted it (this is the reason volumes exist — see Part 5).

### The mental model for troubleshooting (use this everywhere)

Whenever anything goes wrong with Docker, ask these questions in order. This single flow solves 80% of the scenarios in Part 10:

1. **Is the daemon even running?** (`docker info` / `docker ps`)
2. **Does the container exist, and what state is it in?** (`docker ps -a`)
3. **Why did it stop / what is it doing?** (`docker logs`, `docker inspect`)
4. **Is it a build-time problem or a run-time problem?** (Did the image build fine but the container fails to start? Or did the build itself fail?)
5. **Is it inside the container (app/config issue) or outside it (host/network/permissions issue)?** (`docker exec` in to check from inside)
6. **What changed?** (new image, new host, new config, resource limits, disk full)

Keep this flow in your head — I'll refer back to it as "**the diagnostic flow**" throughout Part 10.

---

# PART 1: Docker Fundamentals — Q&A

**Q1: What is Docker?**
Docker is a platform for building, shipping, and running applications inside lightweight, isolated units called containers. It packages an application with everything it needs (code, runtime, system libraries, settings) so it behaves the same everywhere.

**Q2: Why use Docker instead of just installing software directly on a server?**
- **Consistency**: eliminates "works on my machine."
- **Isolation**: one app's dependencies can't clash with another's (e.g., two apps needing different versions of the same library).
- **Speed**: containers start in seconds, unlike VMs.
- **Density**: many containers can run on one host because they share the kernel and are lightweight, so you use hardware more efficiently.
- **Portability**: the same image runs on a laptop, on-prem server, or any cloud.

**Q3: What is a Docker image?**
A read-only template containing the application and everything it needs to run. Think of it as a snapshot/blueprint. Images are built from a Dockerfile and are made of stacked, cached layers.

**Q4: What is a container?**
A running instance of an image — the image plus a thin writable layer on top, plus an isolated process space, network stack, and filesystem view. It's a live, running process with its own little world.

**Q5: What is the difference between a container and a process?**
Under the hood, a container *is* a regular process on the host machine — Docker doesn't run a "mini OS." What makes it look like a separate isolated machine is Linux **namespaces** (it gets its own view of process IDs, network interfaces, hostnames, mount points) and **cgroups** (it gets restricted/metered CPU, memory, I/O). So a container is "just a process, wearing a very convincing costume."

**Q6: Explain Docker architecture (client-server model).**
- **Docker Client**: the `docker` command you type.
- **Docker Daemon (dockerd)**: the background service that does the actual work — building images, running containers, managing networks/volumes. The client talks to the daemon via a REST API (usually over a Unix socket).
- **Docker Registry**: stores images (e.g., Docker Hub, AWS ECR, private registries).
- **containerd / runc**: lower-level components the daemon delegates to for actually creating and running containers using kernel primitives.

Mental model: you (client) place an order at a counter (daemon), the kitchen (containerd/runc) actually cooks (creates the container) using ingredients from the pantry (registry/local image cache).

**Q7: What operating systems does Docker run on, and how does it run on Windows/Mac if containers need a Linux kernel?**
Docker containers share the *host* Linux kernel. On Windows and Mac (which don't have a Linux kernel), Docker Desktop runs a lightweight Linux VM in the background, and your containers actually run inside that VM. This is invisible to you day-to-day but explains why Docker Desktop needs virtualization enabled, and why file-sharing between your Mac/Windows filesystem and a container can sometimes be slower (it's crossing a VM boundary).

**Q8: What is Docker Hub?**
A public registry (cloud repository) of Docker images — official images (like `nginx`, `postgres`, `ubuntu`) and community/user-published images. `docker pull` fetches from here by default.

**Q9: What's the difference between `docker` and `docker-compose` (or `docker compose`)?**
`docker` manages one container/image at a time via CLI commands. `docker compose` (a tool built on top) lets you define and run *multi-container* applications using a single YAML file, starting/stopping/scaling the whole stack with one command. Covered in depth in Part 6.

---

# PART 2: Images & Dockerfile — Container *Creation*

### The core idea

A **Dockerfile** is a step-by-step recipe. Each line becomes a **layer** in the resulting image. Docker builds top-to-bottom and **caches each layer**. If a layer hasn't changed (and nothing above it changed either), Docker reuses the cached layer instead of redoing the work — this is why build order matters enormously.

### Anatomy of a typical Dockerfile (explained line by line)

```dockerfile
FROM node:18-alpine          # Start from a base image (a pre-built OS + Node.js runtime)
WORKDIR /app                 # Set the working directory inside the container (like "cd /app")
COPY package*.json ./        # Copy just dependency manifests first (see caching trick below)
RUN npm install               # Install dependencies — this becomes a cached layer
COPY . .                     # Now copy the rest of the application code
EXPOSE 3000                  # Documents that the app listens on port 3000 (doesn't actually publish it)
CMD ["node", "server.js"]    # The default command to run when a container starts
```

**Q10: Why copy `package*.json` before copying the rest of the code, instead of just doing `COPY . .` once?**
This is a classic caching optimization. Dependency installs (`npm install`) are slow. If you `COPY . .` first, then *any* code change (even a one-line typo fix) invalidates the cache for every layer after it, forcing a full reinstall of dependencies every time you rebuild. By copying only the dependency manifest files first and running install, that expensive layer stays cached as long as dependencies haven't changed — only your actual source code copy gets re-run, which is fast. **Mental model: put things that change rarely near the top, and things that change often near the bottom.**

**Q11: What's the difference between `CMD` and `ENTRYPOINT`?**
- `ENTRYPOINT` defines the *fixed* command that always runs.
- `CMD` defines *default arguments* — easily overridden when you run the container.
- Used together: `ENTRYPOINT ["python", "app.py"]` + `CMD ["--port", "8080"]` means the container always runs `python app.py`, but the `--port 8080` part can be overridden at `docker run` time (e.g., `docker run myimage --port 9090`).
- Used alone, `CMD` is fully replaceable if you pass a command at `docker run` time; `ENTRYPOINT` is not (unless you use `--entrypoint`).

**Q12: What's the difference between `COPY` and `ADD`?**
Both copy files into the image. `ADD` has extra "magic": it can auto-extract local tar archives and fetch remote URLs. Best practice: **prefer `COPY`** because it's predictable and explicit; only use `ADD` when you specifically need its extraction feature. Interviewers like hearing "prefer COPY unless you need ADD's specific behavior."

**Q13: What is a multi-stage build, and why use it?**
A Dockerfile can have multiple `FROM` stages. You build/compile in one stage (which needs heavy build tools, compilers, etc.) and then copy *only the final compiled artifact* into a clean, minimal final stage — discarding all the build tools.

```dockerfile
# Stage 1: build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: final minimal image
FROM alpine:latest
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
```
**Why**: the final image doesn't contain the entire Go compiler toolchain (hundreds of MBs) — just the small compiled binary. This dramatically shrinks image size, reduces attack surface (fewer tools an attacker could abuse), and speeds up deployment.

**Q14: What is a base image, and how do you choose one?**
It's the starting point (`FROM ...`) your image builds on. Considerations:
- **Alpine-based images** (e.g., `node:18-alpine`) are very small (~5MB base) using musl libc — great for minimizing size, but occasionally have compatibility quirks with some compiled binaries.
- **Slim/Debian-based images** are a bit bigger but more broadly compatible.
- **`scratch`** is an empty base (no OS at all) — used for truly minimal images (e.g., a statically compiled Go binary with zero dependencies).
Choosing smaller base images = smaller attack surface, faster pulls/deploys, but sometimes more debugging friction (fewer tools installed to poke around with).

**Q15: What does `docker build` actually do?**
```bash
docker build -t myapp:1.0 .
```
- `-t myapp:1.0` tags the resulting image with a name and version.
- `.` is the **build context** — the directory sent to the Docker daemon (all files here are potentially available to `COPY`/`ADD`).
- The daemon reads the Dockerfile, executes each instruction in order, creates/caches a layer per instruction, and produces the final image.

**Q16: What is `.dockerignore` and why does it matter?**
Just like `.gitignore`, it excludes files/folders (like `node_modules`, `.git`, log files) from the build context. **Why it matters**: a large build context slows down every build (it all gets sent to the daemon), can accidentally bake in secrets or huge unnecessary files, and can invalidate cache layers unpredictably (e.g., copying a `.git` folder whose hash changes on every commit).

**Q17: How do you reduce Docker image size? (Common but important question)**
- Use a smaller base image (alpine/slim/distroless).
- Use multi-stage builds to discard build-time tools.
- Combine `RUN` commands with `&&` to reduce the number of layers, and clean up caches in the *same* layer they were created (e.g., `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*`) — if you clean up in a *separate* `RUN`, the earlier layer still has the bloat baked in, because layers are immutable once created.
- Use `.dockerignore` to avoid copying unnecessary files.
- Avoid installing unnecessary debugging tools in production images.

**Q18: What are image tags, and what does `latest` really mean?**
A tag is a human-readable label pointing to a specific image version (`myapp:1.0`, `myapp:latest`). `latest` is **not magic** — it's just a default tag applied when you don't specify one; it doesn't automatically mean "the newest version." **Best practice**: always use explicit version tags in production (e.g., `myapp:2.3.1`) rather than relying on `latest`, because `latest` can silently change and break reproducibility.

**Q19: What is `docker history` used for?**
`docker history <image>` shows each layer of an image, its size, and the command that created it — useful for figuring out *which instruction* bloated your image size.

---

# PART 3: Container Lifecycle Management

### The mental model: a container's states

```
   docker create/run
         |
         v
     [Created] --start--> [Running] --stop/kill--> [Stopped/Exited]
                              |                          |
                            pause                      start
                              v                          |
                          [Paused]                       v
                              |                     [Running again]
                            unpause
```
A stopped container is **not deleted** — its writable layer and metadata still exist on disk until you `docker rm` it. This is a common point of confusion: "stopped" ≠ "gone."

**Q20: Difference between `docker run`, `docker start`, and `docker create`?**
- `docker create` — creates a container from an image but doesn't start it (rarely used directly).
- `docker start` — starts an existing (stopped) container.
- `docker run` — the common one: it's shorthand for `docker create` + `docker start`, and optionally attaches to it. It always makes a **new** container from the image.

**Q21: Explain common `docker run` flags.**
```bash
docker run -d --name myweb -p 8080:80 -e ENV=prod -v mydata:/data --restart unless-stopped nginx:1.25
```
- `-d` — detached mode (runs in background, doesn't block your terminal).
- `--name myweb` — gives the container a friendly name instead of a random one.
- `-p 8080:80` — **port mapping**: `hostPort:containerPort`. Traffic to port 8080 on your machine is forwarded to port 80 inside the container.
- `-e ENV=prod` — sets an environment variable inside the container.
- `-v mydata:/data` — mounts a **volume** named `mydata` into `/data` inside the container (persists data — see Part 5).
- `--restart unless-stopped` — restart policy (see Q28).
- `-it` — interactive + TTY, used when you want a live shell, e.g. `docker run -it ubuntu bash`.
- `--rm` — automatically deletes the container once it exits (great for throwaway/test containers).

**Q22: What's the difference between `docker stop` and `docker kill`?**
- `docker stop` sends `SIGTERM` (a polite "please shut down") and waits a grace period (default 10s) for the process to exit cleanly, then sends `SIGKILL` if it hasn't.
- `docker kill` sends `SIGKILL` immediately — an abrupt, forceful termination.
**Why it matters**: `stop` gives your app a chance to close database connections, finish in-flight requests, flush buffers, etc. `kill` doesn't — use it only when a container is unresponsive and won't stop gracefully.

**Q23: What does `docker exec` do, and how is it different from `docker attach`?**
- `docker exec -it mycontainer bash` — runs a *new* process (e.g., a shell) inside an already-running container. This is your main tool for "going inside" a container to poke around, check files, run diagnostic commands. Exiting this shell does **not** stop the container.
- `docker attach` — connects your terminal to the container's *main* process (PID 1) input/output stream directly. If you exit/Ctrl+C here, you can actually stop the container's main process. This is why `exec` is almost always the safer, preferred way to "get a shell into a container."

**Q24: What happens to a container's data when it's removed (`docker rm`)?**
The writable layer is deleted — **any data not stored in a volume or bind mount is lost forever.** This is one of the most important things to internalize: **containers are meant to be disposable; data that must survive should live in a volume, not in the container's own filesystem.**

**Q25: What is `docker ps` and what do its variants show?**
- `docker ps` — lists **running** containers.
- `docker ps -a` — lists **all** containers, including stopped/exited ones.
- `docker ps -q` — quiet mode, just IDs (useful for scripting, e.g. `docker rm $(docker ps -aq)`).

**Q26: How do you view logs from a container?**
```bash
docker logs mycontainer
docker logs -f mycontainer      # follow (like `tail -f`)
docker logs --tail 100 mycontainer   # last 100 lines
docker logs --since 10m mycontainer  # logs from the last 10 minutes
```
This is almost always your **first stop** when troubleshooting a misbehaving container.

**Q27: What is `docker inspect` used for?**
It dumps a huge JSON blob of low-level metadata about a container or image: its IP address, mounted volumes, environment variables, restart count, exit code, resource limits, and more. Interviewers love this question because it shows you know how to dig deeper than surface-level commands.
```bash
docker inspect mycontainer
docker inspect --format='{{.State.ExitCode}}' mycontainer   # extract just one field
```

**Q28: What are restart policies, and when do you use each?**
- `no` (default) — never restart automatically.
- `on-failure[:max-retries]` — restart only if the container exits with a non-zero (error) status, optionally capped at N retries.
- `always` — always restart, no matter how it stopped, even if you rebooted the host (unless you manually stopped it).
- `unless-stopped` — like `always`, but if *you* manually stopped it, it won't restart automatically on daemon/host restart.
**Interview tip**: `unless-stopped` is the most commonly recommended default for production services, because it recovers from crashes but respects an intentional manual stop.

**Q29: How do you update the resource limits or restart policy of a container without recreating it?**
```bash
docker update --restart=always --memory=512m mycontainer
```
`docker update` changes some settings live; however, some settings (e.g., port mappings, volumes) **cannot** be changed on an existing container — you must remove and recreate it with new flags. This is worth knowing: containers are largely **immutable** by design for many settings.

**Q30: What's the difference between `docker rm` and `docker rmi`?**
- `docker rm` removes a **container**.
- `docker rmi` removes an **image**.
You cannot remove an image that a container (even a stopped one) still depends on, unless you force it or remove the dependent containers first — because the image's layers might still be in use.

**Q31: How do you clean up unused Docker resources?**
```bash
docker system prune            # removes stopped containers, unused networks, dangling images, build cache
docker system prune -a         # also removes all unused images (not just dangling ones)
docker volume prune            # removes unused volumes
docker container prune         # removes stopped containers only
```
**Caution to mention in an interview**: `prune -a` can delete images you intended to keep around for quick reuse — always double check what's "unused" before running this on a shared/production host.

---

# PART 4: Networking

### The mental model

By default, Docker creates a private virtual network for your containers. Think of it like a **power strip**: each container plugs into a virtual "network switch" (a Docker network), gets its own private IP address, and can talk to other containers plugged into the same switch. To let the outside world in, you explicitly "punch a hole" via **port publishing** (`-p`).

**Q32: What are the default Docker network drivers?**
- **bridge** (default) — a private internal network on the host; containers get their own IP, can talk to each other if on the same bridge network, and reach the outside world via NAT through the host. This is what you get automatically unless you specify otherwise.
- **host** — the container shares the host's network namespace directly — no isolation, no port mapping needed, but no isolation either (less safe, occasionally used for performance).
- **none** — no networking at all; total isolation.
- **overlay** — used in multi-host setups (Docker Swarm) to let containers on *different physical machines* talk to each other as if on the same network.
- **macvlan** — gives a container its own MAC address, making it appear as a physical device on the network (advanced/rare use case).

**Q33: Why can't two containers on the default bridge network reach each other by container name (only by IP), while containers on a custom bridge network can use names?**
Docker's default bridge network does **not** provide automatic DNS resolution between containers by name — only by IP (which is inconvenient because IPs can change). A **user-defined bridge network** (`docker network create mynet`) *does* provide automatic DNS: containers on it can resolve each other simply by container name. **This is why best practice is: always create a custom network for multi-container apps**, rather than relying on the default bridge.

```bash
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet myapp
# Inside "app" container, you can now reach the DB simply via hostname "db"
```

**Q34: Explain port publishing (`-p`) in detail.**
`-p 8080:80` means: "traffic arriving at the host's port 8080 gets forwarded to port 80 inside the container." Without this flag, the container's port is only reachable *from inside the Docker network* (i.e., from other containers), not from outside (your browser on the host, or the internet).
- `-p 80` (no host port specified) — Docker picks a random free host port. Find it with `docker port mycontainer`.
- `-p 127.0.0.1:8080:80` — binds only to localhost, not all network interfaces — good for security when you don't want it exposed externally.

**Q35: What's the difference between `EXPOSE` in a Dockerfile and `-p` at runtime?**
`EXPOSE` is just **documentation/metadata** — it tells humans and tools "this container listens on this port," but does **not** actually publish/open the port to the host. Only `-p` (or `-P` to auto-publish all exposed ports to random host ports) actually makes the port reachable from outside.

**Q36: How do containers resolve DNS / access the internet?**
By default, Docker containers use the host's DNS settings or Docker's embedded DNS server (127.0.0.11 inside the container, used to resolve other container names). Internet access happens via NAT through the host, similar to how devices on your home WiFi router share one public IP.

**Q37: How do you inspect a container's network settings?**
```bash
docker network ls                     # list all networks
docker network inspect mynet          # see which containers are attached, their IPs
docker inspect -f '{{.NetworkSettings.IPAddress}}' mycontainer
```

---

# PART 5: Storage & Volumes

### The mental model

Remember: a container's own filesystem is **temporary** — tied to the container's life. If you need data to survive container restarts/removals, or to be shared between containers, or between the host and a container, you need one of these:

| Type | What it is | Analogy |
|---|---|---|
| **Volume** | Storage managed entirely by Docker, stored in Docker's own area on disk | A storage unit you rent — Docker manages the address and organization |
| **Bind mount** | You map a specific folder from the *host* directly into the container | Plugging in an external hard drive at an exact path |
| **tmpfs mount** | Data stored in host RAM only, never written to disk | A whiteboard — erased when the container stops |

**Q38: Why are volumes generally preferred over bind mounts?**
- Volumes are managed by Docker (via `docker volume` commands), portable across host OSes, and easier to back up/migrate.
- Bind mounts depend on a specific host directory structure existing — less portable, and give the container direct access to potentially sensitive host paths (a security consideration).
- Volumes also work better with Docker's permission handling and are the recommended default for persistent app data (like databases).
Bind mounts are still great for **development** — e.g., mounting your local source code folder into a container so code changes reflect immediately without rebuilding the image.

**Q39: Show the commands for volumes.**
```bash
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker run -d -v mydata:/var/lib/postgresql/data postgres     # named volume
docker run -d -v /home/user/app:/app myapp                     # bind mount (host path : container path)
docker run -d --mount type=volume,source=mydata,target=/data myapp   # --mount is the more explicit/modern syntax
docker volume rm mydata
```

**Q40: What happens if you mount a volume to a directory that already has files in the image (e.g., mounting an empty volume onto `/var/lib/postgresql/data` which had default files)?**
The **first time** a named volume is used and it's empty, Docker copies the existing content from the image's directory into the volume. On subsequent runs, the volume's content (not the image's) takes precedence — the volume "wins" once it has data. This is a subtle but commonly-asked point.

**Q41: How do you back up and restore a volume?**
Common approach — run a temporary container that mounts the volume and a host directory, then use `tar`:
```bash
# Backup
docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar czf /backup/mydata-backup.tar.gz -C /data .

# Restore
docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar xzf /backup/mydata-backup.tar.gz -C /data
```
**Why this pattern**: it uses a disposable, minimal container just as a "worker" to move data — you don't need `tar` installed on your host, and it works identically on any OS running Docker.

---

# PART 6: Docker Compose

### The mental model

If a single `docker run` command is like manually setting up one Lego piece, Docker Compose is the instruction booklet for building the *whole* Lego set in one go — defining several containers (services), their networks, volumes, and how they relate, in one YAML file, and bringing them all up/down together.

**Q42: Why use Docker Compose instead of running several `docker run` commands manually?**
- One command (`docker compose up`) starts your entire multi-container application in the correct order.
- Configuration is version-controlled, human-readable YAML instead of long, error-prone CLI commands.
- Automatically creates a shared network for all services so they can resolve each other by name.
- Easy to tear down and rebuild an identical environment (`docker compose down` / `up`), great for reproducibility across developers/environments.

**Q43: Example `docker-compose.yml`, explained.**
```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - DB_HOST=db
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15
    volumes:
      - dbdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret

volumes:
  dbdata:
```
- `services:` — each key (`web`, `db`) becomes a container.
- `build: .` — build from the Dockerfile in the current directory (vs. `image:` which pulls a pre-built image).
- `depends_on` — controls **startup order** (web waits for db's container process to start) — but note the caveat in Q44.
- Compose automatically creates a network so `web` can reach `db` simply via the hostname `db`.
- `volumes:` at the bottom declares named volumes usable by services.

**Q44: Does `depends_on` guarantee the dependency is actually *ready* (e.g., Postgres accepting connections), not just that its container has started?**
No — this is a very commonly-asked "gotcha" question. `depends_on` only waits for the *container process* to start, not for the *application inside it* to be ready to accept requests (Postgres might take a few seconds to initialize even after its process starts). To truly wait for readiness, use:
- A `healthcheck` in the Dockerfile/Compose file plus `depends_on: condition: service_healthy`, or
- An application-level retry/wait-for-it script that retries the connection until it succeeds.

**Q45: Common Compose commands.**
```bash
docker compose up -d          # start all services in the background
docker compose down           # stop and remove containers, networks (add -v to also remove volumes)
docker compose logs -f web    # follow logs for one service
docker compose ps             # list running services
docker compose build          # rebuild images defined with "build:"
docker compose exec web bash  # shell into a running service
```

---

# PART 7: Resource Management & Monitoring

**Q46: How do you limit CPU and memory for a container, and why would you?**
```bash
docker run -d --memory="512m" --cpus="1.5" myapp
```
**Why**: without limits, one misbehaving container (e.g., a memory leak) can consume all the host's resources and starve every other container/process on the same machine ("noisy neighbor" problem). Limits enforce fair sharing and predictability — this is done under the hood using **cgroups** (control groups), a Linux kernel feature that Docker configures for you.

**Q47: What happens when a container exceeds its memory limit?**
The Linux kernel's **OOM (Out Of Memory) killer** steps in and kills the process, because it violated its cgroup memory limit. Docker reports this as the container being **OOMKilled** (visible in `docker inspect` under `State.OOMKilled: true`). This is a very common real-world troubleshooting scenario — covered in depth in Part 10.

**Q48: How do you monitor live resource usage of running containers?**
```bash
docker stats                  # live streaming CPU%, memory usage/limit, network I/O, block I/O for all containers
docker stats mycontainer      # for just one
```
This is your equivalent of `top`/`htop`, but for containers.

**Q49: How do you see what processes are running inside a container?**
```bash
docker top mycontainer
```
Useful to check if the expected application process is actually running, or if something unexpected (or nothing) is running inside.

---

# PART 8: Security

**Q50: Why is it bad practice to run a container's main process as `root`?**
By default, if you don't specify otherwise, processes inside a container run as `root` (UID 0). If an attacker exploits a vulnerability in your app to break out of the container's isolation, running as root **inside** the container makes it far more dangerous (any container-escape vulnerability grants root-level access on the host too, in the worst case). Best practice: create and switch to a non-root user in the Dockerfile:
```dockerfile
RUN adduser -D appuser
USER appuser
```

**Q51: What is the principle behind not storing secrets (passwords, API keys) in a Dockerfile or image layer?**
Anything baked into an image layer (via `ENV`, `ARG`, or a `COPY`'d file) is visible to **anyone who can pull or inspect that image** — even if you delete the file in a later layer, it still exists in the earlier layer's history (`docker history` can reveal it). Secrets should be injected at **runtime** instead — via environment variables passed at `docker run`, Docker secrets (in Swarm), or external secret managers (Vault, AWS Secrets Manager, Kubernetes Secrets) — never committed into the image itself.

**Q52: What is image scanning, and why do it?**
Tools like `docker scan`, Trivy, or Snyk scan an image's layers against known vulnerability databases (CVEs) in the OS packages and libraries it contains. This is part of a secure CI/CD pipeline — catching known vulnerable dependencies before deployment.

**Q53: What are Linux capabilities, and how do they relate to container security?**
Normally, `root` inside a container has broad system privileges (via Linux "capabilities" like the ability to change file ownership, bind to low ports, manipulate network interfaces, etc). You can **drop unnecessary capabilities** to reduce the blast radius of a compromise:
```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```
This follows the **principle of least privilege** — give a container only the exact permissions it needs, nothing more.

---

# PART 9: Logging & Debugging Tools Recap

**Q54: What are Docker logging drivers?**
Docker can send container logs to different destinations besides the default local JSON file — e.g., `syslog`, `journald`, `fluentd`, `awslogs`, `gelf`. Configured via the `--log-driver` flag or the daemon config, this matters at scale because the default `json-file` driver, left unmanaged, can grow unbounded and fill up disk (see troubleshooting Part 10).

**Q55: How do you limit log file size to prevent disk from filling up?**
```bash
docker run --log-opt max-size=10m --log-opt max-file=3 myapp
```
This caps each log file at 10MB and keeps at most 3 rotated files — a real production concern, and a very common troubleshooting root cause (see Scenario 6 in Part 10).

---

# PART 10: SCENARIO-BASED TROUBLESHOOTING (the core skill)

For every scenario below, I give: the situation, **the mental model/approach** (how to *think*, not just what to type), the diagnostic commands in order, likely root causes, and the fix. Practice narrating your thought process out loud in these terms during interviews — that's what separates "knows commands" from "can actually debug."

---

### Scenario 1: "My container exits immediately after starting"

**Approach:** A container only stays alive as long as its main (PID 1) process keeps running in the foreground. If that process finishes (or crashes) instantly, the container stops — this is by design, not a bug in Docker.

**Diagnostic steps:**
```bash
docker ps -a                       # confirm it's "Exited", check the exit code shown
docker logs mycontainer            # see what the app printed right before dying — usually shows the real error
docker inspect --format='{{.State.ExitCode}}' mycontainer
```

**Reading the exit code:**
- `0` — clean, intentional exit (e.g., a script that ran and finished — maybe you expected it to keep running as a server, but it's a one-shot script).
- `1` (or other non-zero) — an application error; check logs for a stack trace.
- `137` — the process was killed with SIGKILL, often because of an **OOM kill** (see Scenario 5) or `docker stop` timing out.
- `139` — segmentation fault.

**Common root causes:**
- The `CMD`/`ENTRYPOINT` runs a foreground-incompatible command (e.g., you ran a script that daemonizes itself in the background and then exits, leaving nothing for PID 1 to do).
- A missing config/env var causes the app to crash on startup — check `docker logs` for a clear stack trace.
- You meant to get an interactive shell but forgot `-it` (e.g., `docker run ubuntu bash` without `-it` gets no terminal to attach to, so bash exits immediately).

**Mental model to keep**: "A container is a wrapper around one process. If that process ends, the container ends. Find out *why that process ended* — the logs almost always tell you."

---

### Scenario 2: "The container is running, but I can't access my app in the browser"

**Approach:** Walk outward from the app, layer by layer: app itself → container network → port publishing → host firewall → your browser.

**Diagnostic steps:**
```bash
docker ps                                     # is the container actually "Up"? check the PORTS column
docker logs mycontainer                       # is the app actually listening / did it start correctly?
docker exec -it mycontainer sh                # get inside
   curl localhost:PORT                        # from *inside* the container — does the app respond to itself?
docker port mycontainer                       # confirm the actual host port mapping
curl localhost:HOSTPORT                       # from the host — does it respond?
```

**Common root causes, in order of likelihood:**
1. **The app inside the container is listening on `127.0.0.1` / `localhost` instead of `0.0.0.0`.** This is the single most common cause. `127.0.0.1` inside a container only accepts connections from *within that same container's network namespace* — Docker's port forwarding comes from outside that namespace, so it looks like nothing is listening. **Fix: configure the app to bind to `0.0.0.0`.**
2. Forgot the `-p` flag entirely, or mapped the wrong container-side port (e.g., app actually listens on 3000, but you mapped `-p 8080:80`).
3. A firewall on the host or cloud security group is blocking the host port.
4. The app crashed after starting (check `docker logs` and `docker ps -a` for restart loops).

**Mental model**: "Prove the app works, one hop closer to you at a time: inside the container → the container's own network → the host → your browser. The moment it stops working is exactly where the problem is."

---

### Scenario 3: "Cannot connect to the Docker daemon" / "Is the docker daemon running?"

**Approach:** This means the Docker *client* (the CLI) can't talk to the Docker *daemon* (the background service actually doing the work) — it's a connectivity/service issue, not a container issue.

**Diagnostic steps:**
```bash
sudo systemctl status docker      # (Linux) is the daemon service running at all?
sudo systemctl start docker       # start it if not
docker info                       # once running, confirms client-daemon communication works
groups $USER                      # check if your user is in the "docker" group (permission issue, see below)
```

**Common root causes:**
- The Docker service simply isn't running (service crashed, not started on boot, or Docker Desktop isn't open on Mac/Windows).
- **Permission issue**: on Linux, the Docker socket (`/var/run/docker.sock`) is typically owned by `root`/`docker` group. If your user isn't in the `docker` group, you'll get a "permission denied" trying to talk to the daemon — fixed with `sudo usermod -aG docker $USER` (then log out/in).
- Docker Desktop's internal VM (on Mac/Windows) failed to start — usually fixed by restarting Docker Desktop.

**Mental model**: "Client vs daemon: the CLI is just a messenger. If the messenger can't find the daemon, nothing about containers matters yet — fix the connection first."

---

### Scenario 4: "My image build is failing"

**Approach:** Docker builds top-to-bottom, one instruction at a time. The build output tells you **exactly which line/instruction failed** — read the last ~20 lines of output carefully rather than panicking at the wall of text.

**Diagnostic steps:**
```bash
docker build -t myapp . --progress=plain --no-cache   # verbose output, ignore cache to see the real error fresh
```
- `--no-cache` matters because sometimes a stale cached layer hides a problem that would occur on a truly fresh build (e.g., in someone else's CI environment).

**Common root causes:**
- A `RUN` command fails (e.g., a package name typo, a package that no longer exists in the repo, network issue during `apt-get`/`npm install`).
- A `COPY`/`ADD` references a file path that doesn't exist in the build context (often because it's excluded by `.dockerignore`, or the file simply doesn't exist where expected relative to the Dockerfile).
- Build context issues: forgetting that `COPY` paths are relative to the **build context** (the directory you passed to `docker build`), not the Dockerfile's own location.
- Base image tag doesn't exist / typo in `FROM` line.

**Mental model**: "The build log is sequential and literal — find the *first* failing instruction (not necessarily the last line printed), because one failure can cascade into misleading follow-up errors."

---

### Scenario 5: "A container keeps getting killed / restarting on its own" (OOMKilled)

**Approach:** Distinguish between the app crashing on its own (bug) vs. being killed by an external force (resource limits). This is where `docker inspect` earns its keep — `docker logs` alone won't show an OOM kill, because the kernel kills the process abruptly, often without giving the app a chance to log anything.

**Diagnostic steps:**
```bash
docker inspect mycontainer | grep -A5 '"State"'
# Look specifically for:
#   "OOMKilled": true
#   "ExitCode": 137
docker stats mycontainer          # watch memory climb toward its limit in real time, if you can reproduce it
```

**Common root causes:**
- The container has a memory limit (`--memory`) too low for what the app actually needs under load.
- A genuine memory leak in the application.
- No limit was set, but the *host itself* ran out of memory, and the kernel's OOM killer picked this container's process as the victim.

**Fix / approach:**
- Increase the memory limit if the app legitimately needs more (`docker update --memory=1g mycontainer` or redeploy with a higher limit).
- If it's a leak, that's an application-level bug — the container just exposed it; scaling resources is a band-aid, not a fix.
- Check `restart_policy` — an `always`/`unless-stopped` policy will make Docker keep restarting the container after every OOM kill, which *looks* like a "random restart loop" if you don't check the exit code/OOMKilled flag first.

**Mental model**: "Restarting on its own has two very different explanations — the app is buggy and exiting itself, or something external is killing it. `docker inspect`'s exit code and OOMKilled field tell you which one you're dealing with; don't guess."

---

### Scenario 6: "The host is running out of disk space"

**Approach:** Docker accumulates disk usage silently over time from four main sources: images, stopped containers, unused volumes, and build cache. Find out *which* of these is the actual offender before blindly deleting things.

**Diagnostic steps:**
```bash
docker system df               # shows disk usage broken down by images / containers / volumes / build cache
docker system df -v            # verbose, per-item breakdown
du -sh /var/lib/docker/containers/*/*-json.log   # check if individual container LOG FILES have grown huge
```

**Common root causes:**
- Old/unused images piling up over time (especially in CI environments building frequently).
- Log files growing unbounded (a chatty app with no log rotation configured — see Part 9, Q55).
- Dangling/orphaned volumes from removed containers that used anonymous volumes.
- Build cache accumulating from many `docker build` runs.

**Fix:**
```bash
docker system prune -a --volumes    # aggressive cleanup — confirm nothing important is "unused" first!
docker run --log-opt max-size=10m --log-opt max-file=3 ...   # prevent future log bloat
```

**Mental model**: "Don't `prune -a` reflexively — first measure *where* the space actually went with `docker system df`, then target the real cause, because logs vs. images vs. volumes each need a different fix."

---

### Scenario 7: "A container can't reach the internet / DNS resolution is failing inside the container"

**Approach:** Separate "can it resolve names" from "can it reach IPs at all" — these point to different layers of the problem (DNS vs. raw connectivity/routing).

**Diagnostic steps:**
```bash
docker exec -it mycontainer sh
   ping 8.8.8.8            # tests raw IP connectivity (bypasses DNS entirely)
   nslookup google.com     # or: cat /etc/resolv.conf   — tests DNS resolution specifically
```
- If `ping 8.8.8.8` works but `nslookup` fails → it's a **DNS** problem specifically.
- If `ping 8.8.8.8` also fails → it's a broader **network/routing** problem (or the host itself has no internet).

**Common root causes:**
- Corporate VPN or restrictive host DNS settings not being passed through to the container correctly.
- Custom Docker network misconfigured without proper DNS settings.
- The host machine itself has no internet access (test with `ping 8.8.8.8` from the host directly, outside Docker).
- Overly restrictive `--dns` flag set incorrectly at container run time.

**Fix**: often as simple as explicitly specifying a working DNS server: `docker run --dns=8.8.8.8 myapp`, or fixing the Docker daemon's DNS config in `/etc/docker/daemon.json`.

**Mental model**: "Peel connectivity into two separate questions — 'can I reach an IP at all' and 'can I turn a name into an IP' — and test them independently instead of testing them together."

---

### Scenario 8: "Two containers can't talk to each other"

**Approach:** Containers can only resolve each other **by name** if they're on the same **user-defined** network. If they're on different networks (or one is on the unnamed default bridge), name resolution — and sometimes connectivity entirely — will fail.

**Diagnostic steps:**
```bash
docker network ls
docker inspect mycontainer --format='{{json .NetworkSettings.Networks}}'   # which network(s) is it actually on?
docker network inspect mynet     # confirms which containers are actually attached
docker exec -it containerA ping containerB    # test directly
```

**Common root causes:**
- The two containers were started separately (e.g., different `docker run` commands or different Compose files) and ended up on different default networks.
- Using the default `bridge` network, which doesn't support name-based DNS resolution (see Part 4, Q33).
- A typo in the hostname used to reach the other container (should match the container/service name exactly).

**Fix**: put both containers on the same explicitly-created network:
```bash
docker network create sharednet
docker network connect sharednet containerA
docker network connect sharednet containerB
```
Or, in Compose, simply put them in the same `docker-compose.yml` — Compose automatically creates one shared network for all its services.

**Mental model**: "Containers aren't magically aware of each other. They can only find each other if they're plugged into the same virtual 'switch' (network) — check that first before assuming it's an app-level bug."

---

### Scenario 9: "Data isn't persisting" / "Permission denied" errors on a mounted volume

**Approach:** Two very different problems get lumped under "volume issues" — (a) you didn't actually persist the data outside the container in the first place, or (b) you did, but there's a UID/GID mismatch causing permission errors.

**Diagnostic steps:**
```bash
docker inspect mycontainer --format='{{json .Mounts}}'   # confirm a volume/bind mount is actually attached where you think
docker exec -it mycontainer ls -la /path/to/data          # check ownership/permissions from inside
id                                                          # (inside container) what user/UID is the process running as?
```

**Common root causes:**
- **No volume was actually mounted** — data was written straight into the container's writable layer, and it "disappeared" simply because the container was removed and recreated (very common beginner mistake — always double-check the `-v`/`--mount` flag is actually present and pointed at the right path).
- **UID/GID mismatch**: the process inside the container runs as a specific user (say UID 1000), but the host directory (for a bind mount) is owned by a different UID, so the container process gets "permission denied" trying to write to it.
- Using an **anonymous volume** (`-v /data` with no name) instead of a named one — anonymous volumes are easy to lose track of and get orphaned when the container is removed.

**Fix:**
- Always use named volumes or explicit bind mounts for anything that must survive: `-v mydata:/app/data`.
- For permission mismatches, either run the container as a matching UID (`--user 1000:1000`), or `chown` the host directory to match the container's expected UID.

**Mental model**: "First confirm the volume is even attached where you think it is (`docker inspect`) before assuming it's a permissions bug — a huge fraction of 'my data disappeared' issues are simply 'there was never a volume mounted at all.'"

---

### Scenario 10: "It works on my machine, but not on the server / a colleague's machine"

**Approach:** This is ironically the exact problem Docker exists to prevent — so when it *still* happens with Docker in play, the cause is almost always that something isn't as identical as you assumed between environments.

**Diagnostic steps / checklist:**
```bash
docker images                       # confirm you're running the exact same image tag on both machines, not just "latest"
docker inspect myimage --format='{{.Id}}'    # compare image IDs/digests across machines — are they byte-for-byte identical?
docker inspect mycontainer          # compare env vars, volumes, and mapped ports between environments
```

**Common root causes:**
- Different image tags/versions deployed (`latest` silently pointed to different builds at different times — see Part 2, Q18).
- Environment variables or config files differ between environments (e.g., a `.env` file present locally but missing on the server).
- A bind-mounted host path exists locally but not identically on the server (paths, permissions, or missing files).
- Platform architecture mismatch (e.g., built on an Apple Silicon Mac (`arm64`) but deploying to an `amd64` server) — leads to subtle runtime failures or an explicit "exec format error."
- The "server" has different resource limits (memory/CPU) causing an OOM kill that doesn't happen locally.

**Mental model**: "Docker guarantees the *image* is identical — it does NOT guarantee everything *around* the container (env vars, mounted files, host resources, CPU architecture) is identical. Diff those explicitly rather than assuming Docker handles it all."

---

### Scenario 11: "`docker-compose up` fails with 'port is already allocated' / 'address already in use'"

**Approach:** This means something on the host is already bound to the port you're trying to map — could be another container, or a completely different (non-Docker) process.

**Diagnostic steps:**
```bash
docker ps                          # is another container already using that host port?
sudo lsof -i :8080                 # (Linux/Mac) what process on the HOST is using this port?
sudo netstat -tulpn | grep 8080    # alternative
```

**Common root causes:**
- A previous container from an earlier `docker run`/`compose up` was never stopped and is still holding the port.
- A non-Docker service on the host (e.g., a locally-installed Nginx or database) is already using that port.

**Fix:** stop the conflicting container/process, or simply change your host-side port mapping (`-p 8081:80` instead of `8080:80`) — remember, only the left side (host port) needs to be unique; the container-side port can stay the same.

**Mental model**: "Port conflicts are always about the HOST port, never the container port — containers each have their own private network namespace, so container-side ports never collide with each other."

---

### Scenario 12: "A container is stuck in a restart loop"

**Approach:** Combine Scenario 1 (why does the process exit) with checking the restart policy — a restart loop is just "Scenario 1, happening repeatedly because a restart policy keeps bringing it back."

**Diagnostic steps:**
```bash
docker ps -a                                    # note the "Restarting" status and rising restart count
docker inspect --format='{{.RestartCount}}' mycontainer
docker logs mycontainer                         # check what happens right before each crash
docker inspect --format='{{.HostConfig.RestartPolicy}}' mycontainer
```

**Approach tip**: temporarily override the restart policy to stop the loop so you can actually investigate calmly:
```bash
docker update --restart=no mycontainer
docker start mycontainer     # start once, manually, and watch logs live without it looping
docker logs -f mycontainer
```

**Common root causes:** same as Scenario 1 (bad config, missing dependency like a database not yet reachable, crashing on a bad environment variable) — but now compounded by `depends_on` timing issues in Compose setups (see Part 6, Q44): the app container keeps crashing because it starts before its database is actually ready to accept connections.

**Mental model**: "A restart loop is not a new category of problem — it's Scenario 1's root cause, just repeating. Freeze the loop first (disable restart policy temporarily), then debug calmly instead of chasing a moving target."

---

### Scenario 13: "The Docker build is really slow" / "cache isn't being used"

**Approach:** Go back to the layer-caching mental model from Part 2 — figure out which instruction is invalidating the cache earlier than it should.

**Diagnostic steps:**
```bash
docker build --progress=plain -t myapp .
# Watch for lines like "CACHED" vs. actually executing — find the FIRST instruction that isn't cached
```

**Common root causes:**
- `COPY . .` placed too early, before dependency-install steps, so any source change reruns everything after it (see Q10).
- A `RUN apt-get update` without a pinned version, where remote package indexes change, subtly busting cache on unrelated rebuilds.
- Large, unnecessary files in the build context (missing `.dockerignore`) slowing down the "sending build context" step itself, before any instruction even runs.

**Fix**: reorder instructions (least-frequently-changing first), add a proper `.dockerignore`, and consider using `--cache-from` in CI pipelines to reuse cache across builds on ephemeral CI runners.

**Mental model**: "Caching is strictly sequential and top-down — one invalidated layer poisons every layer below it. Find the earliest broken link in the chain."

---

### Scenario 14: "I get 'permission denied' errors when running Docker commands themselves (not inside a container)"

**Approach:** This is a *host-level* permissions issue talking to the Docker socket, distinct from Scenario 9 (which is about permissions on mounted data *inside* a container).

**Diagnostic steps:**
```bash
ls -l /var/run/docker.sock         # check ownership — usually root:docker
groups $USER                        # is your user in the "docker" group?
```
**Fix:**
```bash
sudo usermod -aG docker $USER
# then fully log out and back in (group membership doesn't apply to already-open sessions)
```

**Mental model**: "If the error mentions the Docker *socket* or happens on plain `docker` commands (not something happening inside a running container), it's a host-user-permissions issue, not an application issue."

---

### Scenario 15: "Environment variables aren't being picked up by my app inside the container"

**Approach:** Environment variables can be set in several places, and later ones can silently override earlier ones. Trace the precedence order.

**Diagnostic steps:**
```bash
docker exec -it mycontainer env         # what env vars does the container ACTUALLY have, right now?
docker inspect --format='{{.Config.Env}}' mycontainer
```

**Common root causes / precedence to check (roughly highest to lowest priority):**
1. `-e VAR=value` passed directly on `docker run` / `environment:` in Compose.
2. `--env-file` file passed at runtime.
3. `ENV` instructions baked into the Dockerfile at build time.
4. A `.env` file Compose reads automatically for variable *substitution* in the YAML file itself (different from actually injecting env vars into the container — a very commonly confused distinction!).

**Common mistake**: assuming a `.env` file in your project root automatically becomes environment variables *inside* the container — in plain Compose, a `.env` file is primarily used to substitute `${VARS}` inside `docker-compose.yml` itself; you still typically need an `environment:` or `env_file:` section to actually pass values into the container.

**Mental model**: "There are multiple layers where env vars can be defined, and each layer can silently shadow the one before it — always confirm with `docker exec ... env` what the container *actually* has, rather than trusting what you think you configured."

---

# PART 11: Command Cheat Sheet (quick reference)

```bash
# Images
docker build -t name:tag .
docker images
docker rmi image_id
docker pull image:tag
docker push myrepo/image:tag
docker history image

# Containers - lifecycle
docker run -d --name x -p 8080:80 -v vol:/data --restart unless-stopped image
docker ps / docker ps -a
docker start / stop / restart / kill / pause / unpause NAME
docker rm NAME
docker exec -it NAME bash
docker logs -f --tail 100 NAME
docker inspect NAME
docker top NAME
docker stats

# Networking
docker network ls / create / inspect / connect / disconnect

# Volumes
docker volume ls / create / inspect / rm

# Compose
docker compose up -d
docker compose down -v
docker compose logs -f service
docker compose exec service bash

# Cleanup
docker system df
docker system prune -a --volumes
docker container prune
docker image prune -a
docker volume prune

# Debugging deep-dive
docker inspect --format='{{.State.ExitCode}}' NAME
docker inspect --format='{{.State.OOMKilled}}' NAME
docker events                     # live stream of daemon events (starts, stops, kills) — great for catching intermittent issues
```

---

# PART 12: Rapid-Fire Q&A (last-minute review)

- **Container vs VM?** → Containers share the host kernel (namespaces + cgroups); VMs virtualize hardware and run a full separate OS each.
- **Image vs container?** → Image = read-only template; container = running instance of an image plus a writable layer.
- **Why do containers start faster than VMs?** → No OS boot required; they're just isolated processes on the existing kernel.
- **CMD vs ENTRYPOINT?** → ENTRYPOINT is fixed; CMD supplies default (overridable) arguments.
- **How is data lost when a container is removed?** → Anything written outside a mounted volume/bind mount lives only in the container's writable layer, which is deleted with the container.
- **Default bridge vs custom bridge network?** → Custom bridge networks support automatic DNS resolution by container name; the default bridge does not.
- **What causes exit code 137?** → SIGKILL — very often an OOM kill, sometimes a `docker stop` grace-period timeout escalating to `docker kill`.
- **How do you make a container's data persist?** → Use a named volume or bind mount.
- **How do you check why a container died?** → `docker logs`, then `docker inspect` for exit code / OOMKilled flag.
- **How do you shrink an image?** → Multi-stage builds, smaller base images, combine RUN layers with cleanup in the same layer, use `.dockerignore`.
- **Why put COPY package.json before COPY . .?** → To preserve Docker's build cache for the dependency-install layer.
- **What's `docker system prune -a` risk?** → Deletes all unused images, not just dangling ones — can remove images you still wanted cached.
- **Why avoid running as root in a container?** → Reduces the impact of a container-escape vulnerability.
- **Why not bake secrets into an image?** → They persist in layer history and are extractable by anyone with access to the image, even if later "deleted."
- **depends_on gotcha?** → Only waits for the container process to start, not for the app inside to be ready — use healthchecks for true readiness.

---

**Final tip for the interview**: whenever you're asked a troubleshooting question you haven't seen before, say the mental model out loud — "First I'd check if the daemon is up, then check container state, then check logs, then decide if it's a build-time or run-time issue, then narrow down inside vs. outside the container." Interviewers are usually testing your *process*, not whether you've memorized this exact scenario.
