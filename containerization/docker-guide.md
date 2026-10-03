# Docker & Containerization: The Complete SDE Guide

A comprehensive, production-grade guide to containerization fundamentals, Docker architecture, Dockerfile optimization, multi-container orchestration with Docker Compose, security best practices, and debugging workflows.

---

## Table of Contents

1. [Introduction to Containerization](#1-introduction-to-containerization)
2. [Containers vs. Virtual Machines](#2-containers-vs-virtual-machines)
3. [Under the Hood: Linux Kernel Primitives](#3-under-the-hood-linux-kernel-primitives)
4. [Docker Architecture](#4-docker-architecture)
5. [Core Concepts: Images, Containers, Volumes, & Networks](#5-core-concepts-images-containers-volumes--networks)
6. [Writing Production-Grade Dockerfiles](#6-writing-production-grade-dockerfiles)
7. [Layer Caching & Build Optimization](#7-layer-caching--build-optimization)
8. [Data Persistence: Volumes and Mounts](#8-data-persistence-volumes-and-mounts)
9. [Container Networking](#9-container-networking)
10. [Multi-Container Orchestration with Docker Compose](#10-multi-container-orchestration-with-docker-compose)
11. [Security & Production Best Practices](#11-security--production-best-practices)
12. [Troubleshooting & Debugging Guide](#12-troubleshooting--debugging-guide)
13. [CLI Command Reference & Cheat Sheet](#13-cli-command-reference--cheat-sheet)

---

## 1. Introduction to Containerization

In modern software engineering, the classic excuse *"It worked on my machine!"* stems from environment divergence: differences in OS libraries, system packages, runtimes, environment variables, and background dependencies between local developer setups, staging environments, and cloud production servers.

**Containerization** solves this problem by packaging an application together with its entire runtime environment—code, binaries, libraries, system tools, and configuration—into a single, standardized, isolated execution unit called a **container**.

```
+-------------------------------------------------------------------+
|                           Application                             |
|  - Source Code     - Dependencies (Node/Python/Go)   - Config     |
+-------------------------------------------------------------------+
|                        Container Engine                           |
|                      (Docker / containerd)                        |
+-------------------------------------------------------------------+
|                 Shared Host Operating System & Kernel             |
+-------------------------------------------------------------------+
```

### Key Advantages:
- **Portability**: Runs identically on macOS, Windows, Linux, and any cloud provider (AWS ECS/EKS, GCP GKE, Azure AKS).
- **Speed & Efficiency**: Starts in milliseconds with near-zero virtualization overhead because it shares the host kernel.
- **Predictability & Immutability**: Every deployment runs the exact same image snapshot.
- **Resource Density**: High container packing density compared to hardware-heavy virtual machines.

---

## 2. Containers vs. Virtual Machines

While both technologies provide isolation, their architectural abstraction layer differs fundamentally:

```
          Virtual Machines                                Containers
 +-------------------------------+             +-------------------------------+
 |  App A   |  App B   |  App C  |             |  App A   |  App B   |  App C  |
 +----------+----------+---------+             +----------+----------+---------+
 | Bins/Libs| Bins/Libs|Bins/Libs|             | Bins/Libs| Bins/Libs|Bins/Libs|
 +----------+----------+---------+             +----------+----------+---------+
 | Guest OS | Guest OS | Guest OS|             |       Container Engine        |
 +----------+----------+---------+             |     (Docker / containerd)     |
 |          Hypervisor           |             +-------------------------------+
 |      (Type 1 / Type 2)        |             |        Host Kernel            |
 +-------------------------------+             +-------------------------------+
 |      Host OS / Hardware       |             |         Host Hardware         |
 +-------------------------------+             +-------------------------------+
```

| Dimension | Containers (Docker) | Virtual Machines (VMware, VirtualBox, KVM) |
| :--- | :--- | :--- |
| **Virtualization Level** | Operating System level (shares host kernel) | Hardware level (virtualizes hardware via Hypervisor) |
| **Guest OS** | None (uses host kernel + lightweight root filesystem) | Full independent Guest OS per VM (Windows, Ubuntu, etc.) |
| **Startup Time** | Milliseconds to seconds | Minutes |
| **Resource Footprint** | Extremely low (megabytes of RAM/disk) | High (gigabytes of RAM/disk per VM) |
| **Isolation Strength** | Process-level isolation (cgroups, namespaces) | Hardware-level isolation (strict boundary) |
| **Performance** | Native near-bare-metal execution | Hypervisor translation overhead |

> [!NOTE]
> On macOS and Windows, Docker Desktop transparently runs a lightweight Linux VM in the background (using Apple Hypervisor / WSL2) to provide the Linux kernel that Linux containers require.

---

## 3. Under the Hood: Linux Kernel Primitives

Docker is not magic; it leverages fundamental features already present in the Linux kernel:

```
+--------------------------------------------------------------------------+
|                                Container                                 |
+---------------------+---------------------+------------------------------+
|     Namespaces      |       cgroups       |       OverlayFS (UnionFS)    |
|   (What you SEE)    |   (What you USE)    |        (How you STORE)       |
+---------------------+---------------------+------------------------------+
| - PID (Processes)   | - CPU limits        | - Copy-on-Write (CoW)        |
| - NET (Networks)    | - Memory limits     | - Layered read-only images   |
| - MNT (Filesystems) | - Disk I/O limits   | - Thin writable container    |
| - IPC (Shared mem)  | - Process counts    |   layer on top               |
| - UTS (Hostname)    |                     |                              |
| - USER (UID/GID)    |                     |                              |
+---------------------+---------------------+------------------------------+
```

1. **Namespaces (Isolation - "What you see")**:
   - `pid`: Isolates the process ID space. Inside the container, your process sees itself as PID 1, while on the host it is an unprivileged standard PID.
   - `net`: Gives the container its own virtual network interfaces, IP address, and routing table.
   - `mnt`: Isolates filesystem mount points.
   - `ipc`: Isolates Inter-Process Communication (shared memory, message queues).
   - `uts`: Allows the container to have its own hostname.
   - `user`: Maps container UIDs/GIDs to different host UIDs/GIDs.

2. **Control Groups (`cgroups`) (Resource Metering - "What you use")**:
   - Restricts and measures resource usage per container (e.g., limit to `1.5 CPUs` and `512MB RAM`).
   - Prevents a single runaway container from starving the host system (noisy neighbor problem).

3. **Union Filesystems / OverlayFS (Layering - "How you store")**:
   - Combines multiple discrete directory layers into a single cohesive unified view.
   - Employs **Copy-on-Write (CoW)**: Image layers remain read-only. When a container writes or modifies a file, OverlayFS copies the file up to the ephemeral top writable layer.

---

## 4. Docker Architecture

Docker uses a client-server architecture:

```
 +-----------------+                +------------------------------------+                +-----------------------+
 |  Docker Client  |  REST / gRPC   |           Docker Host              |   HTTPS        |    Docker Registry    |
 |     (CLI)       | -------------> |            (Daemon)                | -------------> |     (Docker Hub)      |
 |                 |                |                                    |                |      (AWS ECR)        |
 | docker build    |                |  +------------------------------+  |                +-----------------------+
 | docker run      |                |  |    Docker Daemon (dockerd)   |  |                            ^
 | docker pull     |                |  +------------------------------+  |                            |
 +-----------------+                |                 |                  |                            |
                                    |                 v                  |                            |
                                    |  +------------------------------+  |                            |
                                    |  |          containerd          |  | ---------------------------+
                                    |  +------------------------------+  |         Pulls Images
                                    |                 |                  |
                                    |                 v                  |
                                    |  +------------------------------+  |
                                    |  |         runc (OCI)           |  |
                                    |  +------------------------------+  |
                                    |                 |                  |
                                    |                 v                  |
                                    |  +------------------------------+  |
                                    |  |     Running Containers       |  |
                                    |  | [Cont A]   [Cont B]   ...    |  |
                                    |  +------------------------------+  |
                                    +------------------------------------+
```

- **Docker Client (`docker`)**: The command-line tool you interact with (`docker run`, `docker build`). It translates commands into REST API calls sent over a UNIX socket (`/var/run/docker.sock`) or TCP to the daemon.
- **Docker Daemon (`dockerd`)**: A background service that manages Docker objects such as images, containers, networks, and volumes.
- **containerd**: An industry-standard, high-performance container runtime that manages the complete container lifecycle (image transfer, execution, supervision).
- **runc**: The low-level OCI-compliant runtime that directly interacts with Linux kernel namespaces and cgroups to spawn containers.
- **Registry**: A centralized repository system for sharing images (Docker Hub, GitHub Container Registry `ghcr.io`, AWS ECR, Google Artifact Registry).

---

## 5. Core Concepts: Images, Containers, Volumes, & Networks

```mermaid
graph TD
    DF[Dockerfile: Blueprint Recipe] -->|docker build| IMG[Docker Image: Immutable Snapshot]
    IMG -->|docker run| C1[Container 1: Isolated Process]
    IMG -->|docker run| C2[Container 2: Isolated Process]
    C1 --- VOL[(Volume: Persistent Data)]
    C1 --- NET([User-Defined Network])
    C2 --- NET
```

### 1. Docker Image
- An immutable, read-only template with instructions for creating a container.
- Constructed in layered stacks using a `Dockerfile`.
- Identified by a repository name and tag (e.g., `postgres:16-alpine`, `node:20-slim`).

### 2. Docker Container
- A runnable, isolated instance of an image.
- Adds a thin **read-write container layer** on top of the immutable image layers.
- When deleted, data stored in this writable layer is discarded unless mounted to a persistent volume.

### 3. Docker Volume
- The designated mechanism for persisting data generated by and used by Docker containers.
- Exists outside the union filesystem on the host storage, bypassing the write-overhead of OverlayFS.

### 4. Docker Network
- Manages communication between containers, the host, and external networks.
- Provides built-in DNS service discovery: containers attached to the same custom bridge network can resolve each other by container name.

---

## 6. Writing Production-Grade Dockerfiles

A `Dockerfile` is a text document containing all the sequential instructions a user could call on the command line to assemble an image.

### Essential Directives

| Instruction | Description | Best Practice |
| :--- | :--- | :--- |
| `FROM` | Sets base image for subsequent instructions | Pin exact versions (`node:20.11-alpine3.19`), avoid `:latest` |
| `WORKDIR` | Sets the working directory inside the container | Always use absolute paths (`WORKDIR /app`) instead of `RUN cd` |
| `COPY` | Copies files/directories from host into container | Prefer `COPY` over `ADD` unless auto-extracting `.tar` |
| `RUN` | Executes commands during image build | Chain related commands with `&&` to minimize layer count |
| `ENV` | Sets persistent environment variables | Use for runtime configuration defaults |
| `ARG` | Defines build-time variables (not persisted in runtime) | Use for build flags or dependency versions |
| `EXPOSE` | Documents the port the application listens on | Informational metadata; does not publish ports on the host |
| `USER` | Sets UID/username for subsequent instructions | Always switch to a non-root user before running the app |
| `ENTRYPOINT` | Configures the default executable command | Use JSON exec form: `ENTRYPOINT ["node", "server.js"]` |
| `CMD` | Provides default arguments for `ENTRYPOINT` | Can be easily overridden by `docker run` arguments |

### `CMD` vs `ENTRYPOINT`: Understanding the Exec Form vs Shell Form

Always prefer the **Exec form** (`["executable", "param1"]`) over the **Shell form** (`executable param1`):

```dockerfile
# ❌ Shell Form: Runs inside /bin/sh -c. Does NOT receive OS signals like SIGTERM!
ENTRYPOINT npm start

# ✅ Exec Form: Spawns directly as PID 1. Graceful shutdown signals are received!
ENTRYPOINT ["npm", "start"]
```

#### Combining `ENTRYPOINT` and `CMD`:
```dockerfile
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver", "0.0.0.0:8000"]
```
- Running `docker run my-app` executes: `python manage.py runserver 0.0.0.0:8000`
- Running `docker run my-app migrate` executes: `python manage.py migrate`

---

### Production Multi-Stage Build Examples

Multi-stage builds allow you to use separate images for building vs running your application. You leave compilers, build tools, package managers, and source code out of the final runtime image, resulting in dramatically smaller and more secure images.

#### Example A: Production Multi-Stage Node.js / React Application

```dockerfile
# ==========================================
# Stage 1: Build & Compile
# ==========================================
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependency manifests first to leverage Docker layer caching
COPY package.json package-lock.json ./
RUN npm ci --prefer-offline --no-audit

# Copy source code and compile
COPY . .
RUN npm run build && npm prune --production

# ==========================================
# Stage 2: Minimal Production Runtime
# ==========================================
FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

# Security: Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only built artifacts and production dependencies from builder
COPY --from=builder --chown=appuser:appgroup /app/package.json ./
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist

# Switch away from root
USER appuser

EXPOSE 3000

# Health check to ensure service vitality
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

ENTRYPOINT ["node", "dist/server.js"]
```

#### Example B: Production Multi-Stage Python (FastAPI / Poetry)

```dockerfile
# ==========================================
# Stage 1: Build Virtualenv
# ==========================================
FROM python:3.12-slim AS builder

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    POETRY_VERSION=1.8.2 \
    POETRY_VIRTUALENVS_IN_PROJECT=1 \
    POETRY_NO_INTERACTION=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN pip install --no-cache-dir "poetry==$POETRY_VERSION"
COPY pyproject.toml poetry.lock ./
RUN poetry install --only main --no-root

# ==========================================
# Stage 2: Lean Runtime
# ==========================================
FROM python:3.12-slim AS runner

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"

WORKDIR /app

# Security: Create and switch to non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Copy virtualenv and app code
COPY --from=builder --chown=appuser:appgroup /app/.venv /app/.venv
COPY --chown=appuser:appgroup . /app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### The Importance of `.dockerignore`

Never build without a `.dockerignore` file in your repository root. It prevents secret leaks, decreases build context upload time, and preserves cache validity:

```gitignore
# Git & Documentation
.git
.github
*.md

# Dependencies & Environments
node_modules
.venv
venv
__pycache__
*.pyc

# Local secrets & configs
.env
.env.*
*.pem
*.key
id_rsa

# Build caches & logs
dist
build
.coverage
*.log
.DS_Store
.idea
.vscode
```

---

## 7. Layer Caching & Build Optimization

Docker caches intermediate images for each instruction in a `Dockerfile`. If an instruction and the files it touches have not changed since the last build, Docker reuses the cached layer instantly.

### The Golden Rule: Order Instructions by Change Frequency

```
Least Frequently Changed (Top)
 ├── Base OS & System Packages (FROM, apt-get)
 ├── Dependency Lockfiles (package.json, poetry.lock)
 ├── Dependency Installation (npm ci, pip install)
 ├── Application Source Code (COPY . .)
 └── Compile / Build Command (npm run build)
Most Frequently Changed (Bottom)
```

```dockerfile
# ❌ Anti-pattern: Invalidates dependencies cache on every code edit!
COPY . .
RUN npm install

# ✅ Best practice: Cache dependencies independently from application code!
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

### Layer Optimization Techniques
1. **Combine RUN statements**:
   ```dockerfile
   # Combines into a single layer and cleans up cache in the same layer!
   RUN apt-get update && apt-get install -y --no-install-recommends \
       curl \
       ca-certificates \
    && rm -rf /var/lib/apt/lists/*
   ```
2. **Choose the Right Base Image**:
   - `alpine`: Microscopic (~5MB), uses `musl libc` (watch out for C-extension compatibility in Python/Node).
   - `slim`: Debian-based (~50MB), uses standard `glibc`. Excellent balance of stability, compatibility, and size.
   - `distroless`: Contains strictly your application and runtime dependencies—no package manager, shell, or utilities.

---

## 8. Data Persistence: Volumes and Mounts

Containers are ephemeral by default. If a container crashes or is replaced, changes written to its local filesystem are wiped out. Docker provides three ways to mount data:

```
        +-----------------------------------------------------------+
        |                        Docker Host                        |
        |                                                           |
        |  +---------------------+        +----------------------+  |
        |  |    Named Volume     |        |      Bind Mount      |  |
        |  | /var/lib/docker/... |        |  /home/user/project  |  |
        |  +---------------------+        +----------------------+  |
        |             \                              /              |
        |              \                            /               |
        |     +----------------------------------------------+      |
        |     |                  Container                   |      |
        |     |                 /app/storage                 |      |
        |     |           (tmpfs: in-memory only)            |      |
        |     +----------------------------------------------+      |
        +-----------------------------------------------------------+
```

| Type | Managed By | Host Location | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Named Volumes** | Docker Engine | `/var/lib/docker/volumes/<name>` | Production databases, persistent application state |
| **Bind Mounts** | User / Host OS | Any arbitrary host path (e.g., `./src`) | Local dev hot-reloading, mounting config files |
| **tmpfs Mounts** | Linux Kernel | Host memory (RAM) | High-throughput sensitive keys, temporary tokens |

### Practical Volume Commands

```bash
# Create a named volume
docker volume create pg_data

# Inspect volume details
docker volume inspect pg_data

# Run PostgreSQL with a persistent named volume
docker run -d \
  --name local-postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pg_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine

# Local development with bind mount (live reload)
docker run -d \
  --name web-dev \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  my-dev-image
```

---

## 9. Container Networking

Docker allows containers to communicate with each other, the host machine, and external web APIs.

### Network Drivers

1. **`bridge` (Default)**: Private virtual network on the host. Containers get private IPs (e.g., `172.17.0.x`).
2. **`host`**: Removes network isolation; container shares the host network stack directly (maximum performance, Linux only).
3. **`none`**: Disables all networking for total isolation.
4. **`overlay`**: Enables multi-host networking across Docker Swarm / Kubernetes clusters.

### The Power of Custom User-Defined Bridge Networks

> [!IMPORTANT]
> The **default bridge** does **not** provide automatic DNS resolution between containers. You must create a **user-defined bridge network** to resolve services by container name!

```bash
# 1. Create a user-defined network
docker network create backend-net

# 2. Run Redis on the network
docker run -d --name redis-cache --network backend-net redis:7-alpine

# 3. Run application on the same network - it can connect to Redis via 'redis-cache:6379'
docker run -d --name api-service --network backend-net -p 8080:8080 my-api-image
```

---

## 10. Multi-Container Orchestration with Docker Compose

**Docker Compose** is a declarative tool for defining and running multi-container Docker applications via a `compose.yaml` (or `docker-compose.yml`) file.

### Production-Ready Example: Web API + PostgreSQL + Redis

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: backend-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://app_user:app_pass@postgres-db:5432/production_db
      - REDIS_URL=redis://redis-cache:6379/0
      - NODE_ENV=production
    depends_on:
      postgres-db:
        condition: service_healthy
      redis-cache:
        condition: service_started
    networks:
      - app-network

  postgres-db:
    image: postgres:16-alpine
    container_name: postgres-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_pass
      POSTGRES_DB: production_db
    volumes:
      - pg-data:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432" # Bind to localhost only for security
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d production_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-cache:
    image: redis:7-alpine
    container_name: redis-cache
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  pg-data:
  redis-data:
```

### Essential Compose Commands

```bash
# Start all containers in the background (builds if not built yet)
docker compose up -d

# View live streaming logs across all services
docker compose logs -f

# View live streaming logs for a specific service
docker compose logs -f api

# Execute an interactive shell inside a running service
docker compose exec api sh

# Stop all containers (preserves volumes)
docker compose stop

# Destroy containers and networks (preserves volumes)
docker compose down

# Destroy containers, networks, and persistent volumes (⚠️ DATA LOSS)
docker compose down -v
```

---

## 11. Security & Production Best Practices

### 1. Never Run as Root
By default, Docker containers run as `root` (UID 0). If an attacker breaks out through a container runtime vulnerability, they have root access to your host machine.
```dockerfile
# Create unprivileged group & user
RUN groupadd -g 10001 appuser && useradd -u 10001 -g appuser appuser
USER appuser
```

### 2. Never Hardcode Secrets or Credentials
- **Never** put `.env` files, SSH keys, or DB passwords into your `Dockerfile` or image layers.
- Even if deleted in a later `RUN` step, secrets remain accessible in prior image layers!
- Use runtime environment injection (`-e` or Docker Compose `env_file`), Docker Build Secrets (`RUN --mount=type=secret`), or cloud secret managers (AWS Secrets Manager, HashiCorp Vault).

### 3. Enforce Resource Constraints
Prevent Denial of Service (DoS) and runaway memory leaks using cgroup limits:
```bash
docker run -d \
  --memory="512m" \
  --memory-swap="1g" \
  --cpus="1.5" \
  my-app
```

### 4. Read-Only Root Filesystem
Make the container root filesystem read-only to prevent attackers from downloading tools or injecting malicious scripts:
```bash
docker run --read-only -v tmp-data:/tmp my-app
```

### 5. Scan Images for Vulnerabilities
Regularly scan images during CI/CD:
```bash
# Docker native scan
docker scout cves <image-name>

# Using Trivy
trivy image <image-name>
```

---

## 12. Troubleshooting & Debugging Guide

### Common Pitfalls & Solutions

#### 1. Container Exits Immediately (`Exit Code 0` or `1`)
- **Cause**: Containers terminate as soon as the main process (`PID 1`) completes.
- **Fix**: Check logs with `docker logs <container-id>`. Ensure your process runs in the foreground (e.g., `nginx -g 'daemon off;'` instead of backgrounding).

#### 2. Out of Memory (`OOMKilled` / `Exit Code 137`)
- **Diagnosis**: Run `docker inspect <container-id> | grep -i oom`
- **Fix**: Increase container memory limit or profile application memory usage.

#### 3. "Port is already allocated" Error
- **Cause**: Host port is already in use by another service.
- **Fix**:
  - Linux/macOS: `lsof -i :<port>` or `netstat -tulnp | grep <port>`
  - Windows: `netstat -ano | findstr :<port>`
  - Change the host port mapping: `-p 8081:8080`.

#### 4. Permission Denied on Mounted Volumes
- **Cause**: Container non-root user UID does not match host directory ownership.
- **Fix**: Align UID/GID in Dockerfile or pre-configure host directory permissions (`chown -R 10001:10001 ./data`).

#### 5. Graceful Shutdown & Zombie Processes
- If a container takes exactly 10 seconds to stop upon `docker stop`, your app isn't handling `SIGTERM` and Docker is resorting to `SIGKILL`.
- Use the `--init` flag (`docker run --init ...`) to automatically inject `tini` as PID 1 to reap zombie processes and forward signals.

---

## 13. CLI Command Reference & Cheat Sheet

### Container Lifecycle
```bash
# Run container detached with port mapping and name
docker run -d --name my-web -p 8080:80 nginx:alpine

# List active containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop / Start / Restart container
docker stop my-web
docker start my-web
docker restart my-web

# Remove stopped container
docker rm my-web

# Force remove running container
docker rm -f my-web
```

### Inspection & Debugging
```bash
# View stdout/stderr logs
docker logs -f --tail 100 my-web

# Open interactive shell in running container
docker exec -it my-web /bin/sh

# Show live resource utilization (CPU, Memory, I/O)
docker stats

# Inspect full JSON configuration
docker inspect my-web

# Display running processes inside container
docker top my-web
```

### Image Management
```bash
# Build image from local Dockerfile with tag
docker build -t my-app:v1.0 .

# Build with BuildKit (faster caching)
DOCKER_BUILDKIT=1 docker build -t my-app:v1.0 .

# List local images
docker images

# Remove image
docker rmi my-app:v1.0

# Tag and push image to registry
docker tag my-app:v1.0 registry.example.com/team/my-app:v1.0
docker push registry.example.com/team/my-app:v1.0
```

### System Cleanup & Housekeeping
```bash
# Remove all stopped containers, unused networks, and dangling images
docker system prune

# Deep cleanup: removes unused volumes and all unused images
docker system prune -a --volumes -f

# Check Docker disk usage breakdown
docker system df
```
