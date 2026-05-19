# Docker — Overview

Docker is a platform for packaging applications and their dependencies into **containers** — isolated, lightweight processes that share the host's Linux kernel but run with their own filesystem, network stack, and process tree. Unlike virtual machines, containers don't ship a full guest OS, so they start in milliseconds and consume far less memory and disk. This page covers the mental model and vocabulary; the other files in this directory are command-level references.

## Why use Docker

- **Reproducibility** — a container built on your laptop runs identically on a colleague's machine, in CI, and in production. The image bundles the application binary, libraries, system packages, and config it expects.
- **Isolation** — each container has its own filesystem and process namespace. You can run two versions of Postgres on the same host without conflicts.
- **Distribution** — images are versioned artifacts that you can push to and pull from a registry. "Install MyApp 2.3" becomes `docker pull myorg/myapp:2.3`.
- **Disposability** — containers are cheap to create and destroy. The expected workflow is to throw them away and recreate from the image, not to patch them in place.

## Containers vs virtual machines

A VM emulates a whole computer: hypervisor, guest kernel, guest OS, application. A container shares the host kernel and only packages the user-space pieces (libraries, binaries, config). This is why containers are smaller and faster, but also why a Linux container can't run on Windows directly without a Linux VM underneath (which is what Docker Desktop does on Windows/macOS).

## The client–daemon model

Docker is two pieces of software talking over a local Unix socket:

- **`dockerd`** — a long-running daemon (systemd service) that actually manages images, containers, networks, and volumes. It talks to the kernel via `runc` and `containerd`.
- **`docker`** — the CLI you type. It sends JSON requests to `dockerd` over `/var/run/docker.sock`.

This means anyone who can write to that socket can ask the daemon to start a container that mounts `/` from the host — effectively becoming root. That's why being a member of the `docker` group is equivalent to having root. See [security-rootless.md](security-rootless.md) for the implications and rootless mode.

```bash
# Where the socket lives
ls -l /var/run/docker.sock

# Verify the daemon is reachable from your CLI
docker info
```

## Core vocabulary

The same word often means slightly different things depending on context. This glossary uses the conventions you'll see in the rest of the docs.

| Term | What it is |
|------|------------|
| **Image** | A read-only filesystem snapshot plus metadata (default command, env vars, exposed ports). Built from a `Dockerfile` or pulled from a registry. Identified by name+tag, e.g. `postgres:16`. |
| **Container** | A running (or stopped) instance of an image, with a thin writable layer on top. Has its own PID, mounts, network. |
| **Layer** | An image is composed of stacked read-only layers — one per `RUN`/`COPY`/`ADD` instruction in the Dockerfile. Layers are content-addressed and cached, so unchanged layers are reused between builds. |
| **Tag** | A human-readable pointer to a specific image version, e.g. `:16`, `:16.2`, `:latest`. Tags are mutable — `:latest` today may not be the same image tomorrow. |
| **Digest** | An immutable SHA256 of the image content, e.g. `postgres@sha256:abc…`. Use digests when you need byte-for-byte reproducibility. |
| **Registry** | A server that hosts images. The default is Docker Hub (`docker.io`). Others: GHCR (`ghcr.io`), GCR (`gcr.io`), Quay (`quay.io`), or your own. See [registries.md](registries.md). |
| **Repository** | A named collection of related images on a registry, typically one per application with multiple tags. E.g. `library/postgres` on Docker Hub. |
| **Volume** | Persistent storage managed by Docker, decoupled from a container's lifecycle. See [volumes.md](volumes.md). |
| **Bind mount** | A host directory or file mounted directly into a container. Different from volumes — Docker doesn't manage the data. |
| **Network** | A virtual L2 network connecting containers. The default `bridge` and user-defined bridges are the common cases. See [networking.md](networking.md). |
| **Build context** | The directory you pass to `docker build` (e.g. `.`). Its contents — minus `.dockerignore` patterns — are sent to the daemon and accessible to `COPY`/`ADD`. |
| **Dockerfile** | A text recipe describing how to build an image. See [dockerfile.md](dockerfile.md). |

## Lifecycle of a container

A container moves through a small set of states. Most commands map cleanly to a transition.

```
       docker create                     docker start                 docker stop / exit
created  ───────────►  created  ─────────►  running  ──────────►  exited
                                              │   ▲                    │
                                  docker pause│   │docker unpause      │ docker rm
                                              ▼   │                    ▼
                                            paused                  removed
```

- `docker run` is shorthand for `pull` (if needed) + `create` + `start`, and optionally `attach`.
- `docker ps` shows running containers; `docker ps -a` shows all states (including exited).
- A stopped (`exited`) container still keeps its writable filesystem layer until you `docker rm` it.

The full command reference for these transitions is in [containers.md](containers.md).

## How an image becomes a running container

Walking through a single command to anchor the mental model:

```bash
docker run -d --name pg -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:16
```

1. **Resolve the image.** If `postgres:16` isn't in the local image store, the daemon pulls it from Docker Hub (the default registry).
2. **Create a container.** The daemon stacks the image's read-only layers and adds a thin writable layer on top. It allocates a network interface on the default bridge, sets up a port-forwarding rule (`-p 5432:5432`), and prepares the environment (`-e POSTGRES_PASSWORD=dev`).
3. **Start the container.** The daemon execs the image's default `CMD`/`ENTRYPOINT` as PID 1 inside the container's namespaces. `-d` detaches your shell; the container keeps running in the background.
4. **Cleanup later.** `docker stop pg` sends `SIGTERM` (then `SIGKILL` after a grace period). `docker rm pg` deletes the container's writable layer. The image stays in the local cache for next time.

Every other Docker command is some variation on this loop.

## What's next

| You want to… | Go to |
|---|---|
| Install Docker on Arch or Debian/Ubuntu | [installation.md](installation.md) |
| Pull and inspect images | [images.md](images.md) |
| Run, stop, and inspect containers | [containers.md](containers.md) |
| Build your own image with a Dockerfile | [dockerfile.md](dockerfile.md) |
| Persist data on the host | [volumes.md](volumes.md) |
| Connect containers and expose ports | [networking.md](networking.md) |
| Orchestrate multi-container apps | [compose.md](compose.md) |
| Push images to a registry | [registries.md](registries.md) |
| Free up disk space | [cleanup.md](cleanup.md) |
| Run Docker more securely | [security-rootless.md](security-rootless.md) |
