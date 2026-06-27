# Apple Container + Docker Compose + Socktainer

> Run containers natively on Apple Silicon — no Docker Desktop required.

## What is this?

Apple released [`container`](https://github.com/apple/containerization) at WWDC 2025: a native macOS container runtime built on lightweight VMs (vfkit + kata-containers kernel). It replaces Docker Desktop on Apple Silicon with a lighter, faster alternative.

**Socktainer** = a Docker-socket-proxy container that makes third-party tools (Portainer, Watchtower, CI agents, etc.) work with Apple's container runtime by exposing a standard Docker API socket.

## Prerequisites

- macOS 15+ on Apple Silicon (M1/M2/M3/M4)
- [Homebrew](https://brew.sh)

---

## Quick start — 3 commands

```bash
# 1. Install Apple's container runtime
brew install container
container system start

# 2. Install Docker Compose with Apple container backend
brew install docker-compose
mkdir -p ~/.docker/cli-plugins
ln -sf /opt/homebrew/opt/docker-compose/bin/docker-compose ~/.docker/cli-plugins/docker-compose

# 3. Run your compose stack
docker compose up -d
```

That's it. No Docker Desktop, no VM setup, no DOCKER_HOST configuration needed.

---

## How it works

```
docker compose up
      │
      │  Mach IPC (no Unix socket required)
      ▼
Homebrew docker-compose 5.2+ ──► com.apple.container.apiserver
                                          │
                                          ▼
                                 Apple container runtime
                                 (per-container lightweight VM)
                                      │          │
                                 container1   container2
```

Docker Compose 5.2+ from Homebrew has a native Apple container backend. It talks to Apple's `container-apiserver` via Mach IPC directly — the same mechanism used by the `container` CLI itself.

---

## Full setup guide

### 1 — Apple container runtime

```bash
brew install container

# Start the system services (API server + network + BuildKit)
container system start

# Verify
container system status
# status = running
```

### 2 — Docker Compose (Apple container backend)

```bash
brew install docker-compose

# Link as Docker CLI plugin
mkdir -p ~/.docker/cli-plugins
ln -sf /opt/homebrew/opt/docker-compose/bin/docker-compose \
       ~/.docker/cli-plugins/docker-compose

docker compose version
# Docker Compose version 5.2.0
```

### 3 — (Optional) Docker CLI without Docker Desktop

If you want `docker` commands alongside `docker compose`:

```bash
brew install docker
```

> Note: `docker ps`, `docker images` etc. will **not** show containers managed by Apple's `container` CLI — they talk to different backends. Use `container ls` for the Apple runtime view.

### 4 — Build and run

```bash
# Build an image (uses Apple's BuildKit shim — native arm64)
container build -t myapp:latest .

# Or let Compose build it
docker compose up -d --build
```

---

## Socktainer — Docker socket proxy

Use socktainer when you need a classic Unix socket (`/var/run/docker.sock`) for tools that don't yet support the Apple container Mach IPC backend.

### When you need it

- Portainer, Watchtower, Diun
- CI/CD agents that mount the Docker socket
- Any tool that sets `DOCKER_HOST=unix:///var/run/docker.sock`

### docker-compose.yml with socktainer

```yaml
services:
  socktainer:
    image: alpine/socat
    container_name: socktainer
    restart: unless-stopped
    command: >
      UNIX-LISTEN:/var/run/docker.sock,fork,reuseaddr,mode=660
      UNIX-CONNECT:/run/container.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    network_mode: host

  myapp:
    image: myapp:latest
    depends_on:
      - socktainer
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock
```

### Using the socket directly

```bash
# Point Docker CLI to Apple container socket via socktainer
export DOCKER_HOST=unix:///var/run/docker.sock

# Or create a named context
docker context create apple \
  --docker "host=unix:///var/run/docker.sock"
docker context use apple

# Now standard docker commands work
docker ps
docker images
```

---

## Example: minimal docker-compose.yml

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  api:
    build: .
    environment:
      - NODE_ENV=production
    volumes:
      - app_data:/data

volumes:
  app_data:
```

```bash
docker compose up -d
docker compose logs -f
docker compose down
```

---

## Apple container vs Docker Desktop

| Feature | Docker Desktop | Apple `container` |
|---|---|---|
| Platform | macOS (x86 + ARM) | Apple Silicon native |
| Architecture | Single shared VM | Per-container lightweight VM |
| Memory overhead | ~2–4 GB | ~256 MB per container |
| Docker socket | `~/.docker/run/docker.sock` | Mach IPC |
| Compose | Built-in plugin | Homebrew docker-compose 5.2+ |
| Rosetta (x86 images) | ✅ | ✅ |
| Volumes | VM filesystem | virtio-fs (direct host mount) |
| Cost | Free / paid plans | Free, open source |
| Networking | NAT / VPNKit | vmnet (192.168.64.x) |

---

## Cheat sheet

```bash
# Runtime control
container system start
container system stop
container system status

# Container management
container ls                         # list running
container ls -a                      # all containers
container logs <name>
container exec -it <name> sh
container stop <name>
container rm <name>

# Image management
container build -t myimage:tag .
container image ls
container image rm myimage:tag

# Volumes
container volume ls
container volume rm <name>

# Networking
container system dns                 # DNS configuration
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `container system status` → stopped | Services not started | `container system start` |
| `docker compose` uses Docker Desktop | Wrong CLI plugin linked | Re-link: `ln -sf /opt/homebrew/opt/docker-compose/bin/docker-compose ~/.docker/cli-plugins/docker-compose` |
| `docker ps` shows nothing | Docker Desktop context active | `docker context use apple` (with socktainer) |
| Build fails: `buildkit not found` | Builder not running | `container system start` restarts it |
| Volume data missing after `down` | Unnamed volumes | Always use named volumes in compose |
| Container can't reach internet | vmnet DNS issue | `container system dns` to check |

---

## Resources

- [Apple Containerization — open source](https://github.com/apple/containerization)
- [Apple container CLI — Homebrew](https://formulae.brew.sh/formula/container)
- [Docker Compose spec](https://docs.docker.com/compose/compose-file/)
- [kata-containers](https://katacontainers.io/) — kernel used by Apple's runtime

---

*Tested on: macOS 25.x · Apple Silicon (arm64) · container CLI v1.0.0 · docker-compose v5.2.0*
