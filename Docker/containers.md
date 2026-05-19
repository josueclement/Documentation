# Containers

A container is a running (or stopped) instance of an image. This page covers the full lifecycle: creating one with `docker run`, inspecting what's running, executing commands inside, viewing logs, and removing containers when you're done. Practical end-to-end examples are at the bottom.

## `docker run` — the workhorse

`docker run` is the single most-used Docker command. It combines three steps that can also be run separately: pull the image if missing, create a container from it, and start it. By default it also attaches your terminal to the container's stdout/stderr; pass `-d` to detach.

```bash
# Run a container in the foreground (you see its output; Ctrl+C stops it)
docker run nginx

# Run detached (background) and give it a name
docker run -d --name web nginx

# Auto-remove the container when it stops (great for one-offs)
docker run --rm alpine echo "hello from a container"

# Interactive shell — allocate TTY (-t) and keep stdin open (-i)
docker run --rm -it alpine sh

# Override the default CMD with your own
docker run --rm alpine cat /etc/os-release
```

The arguments after the image name **replace** the image's default `CMD`. The default `ENTRYPOINT` still runs (if any), so for most images `docker run image foo bar` becomes `entrypoint foo bar` inside the container.

## Common `docker run` flags

You'll use these constantly. Each gets a dedicated section or example below; this table is a cheat sheet.

| Flag | Purpose | Example |
|---|---|---|
| `-d` | Detach (run in background) | `docker run -d nginx` |
| `-it` | Interactive TTY (for shells) | `docker run -it alpine sh` |
| `--name N` | Assign a name (otherwise random) | `--name web` |
| `--rm` | Remove container when it exits | `docker run --rm alpine date` |
| `-p H:C` | Publish container port `C` on host port `H` | `-p 8080:80` |
| `-e K=V` | Set environment variable | `-e POSTGRES_PASSWORD=dev` |
| `--env-file F` | Load env vars from a file | `--env-file .env` |
| `-v V:/path` | Mount a volume or bind mount | `-v pgdata:/var/lib/postgresql/data` |
| `-w /path` | Working directory inside container | `-w /app` |
| `-u UID:GID` | Run as a specific user | `-u 1000:1000` |
| `--restart P` | Restart policy (see below) | `--restart unless-stopped` |
| `--network N` | Attach to a named network | `--network appnet` |
| `--memory M` | Hard memory limit | `--memory 512m` |
| `--cpus N` | CPU limit (fractional) | `--cpus 1.5` |
| `--hostname H` | Set container hostname | `--hostname db` |
| `--platform P` | Force image platform | `--platform linux/amd64` |

Each storage-related flag is detailed in [volumes.md](volumes.md); networking flags in [networking.md](networking.md).

## Restart policies

By default a stopped container stays stopped. For long-running services, set a restart policy so the daemon brings them back after crashes or host reboots.

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart |
| `on-failure[:N]` | Restart only on non-zero exit, optionally up to N times |
| `always` | Restart whenever stopped (including manual stops on next daemon start) |
| `unless-stopped` | Like `always` but respects an explicit `docker stop` across daemon restarts |

`unless-stopped` is the right default for homelab services — `docker stop` actually stops them, but a reboot brings them back.

```bash
docker run -d --name web --restart unless-stopped -p 8080:80 nginx
```

## Listing containers

`docker ps` shows what's running. Add `-a` to include stopped/exited ones.

```bash
# Running containers
docker ps

# All containers (running + stopped)
docker ps -a

# Just the IDs (useful in scripts and pipes)
docker ps -aq

# Filter
docker ps --filter "status=exited"
docker ps --filter "name=web"

# Custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Container disk usage
docker ps -s
```

## Inspecting a container

`docker inspect` returns the full JSON for a container (state, network, mounts, env, args). `docker stats` shows live CPU/memory/IO usage. `docker top` shows the processes running inside.

```bash
# Full container metadata
docker inspect web

# Specific field
docker inspect --format '{{.State.Status}}' web
docker inspect --format '{{.NetworkSettings.IPAddress}}' web

# Live resource usage (one-shot, no streaming)
docker stats --no-stream

# Processes inside the container
docker top web
```

## Logs

`docker logs` reads from the container's stdout/stderr stream (whatever PID 1 wrote). For most images this is the application's main log output.

```bash
# Dump everything
docker logs web

# Follow live (like tail -f)
docker logs -f web

# Last 100 lines, with timestamps
docker logs --tail=100 -t web

# Last 10 minutes
docker logs --since=10m web

# Between two timestamps
docker logs --since=2024-01-01T00:00:00 --until=2024-01-02T00:00:00 web
```

> **Note:** `docker logs` only works for containers using the `json-file` or `journald` log driver. Logs grow without limit by default — cap them via `daemon.json` or per-container `--log-opt max-size=10m`. See [cleanup.md](cleanup.md).

## Running commands in a running container

`docker exec` runs a new process inside a container that's already running. The classic use is opening a shell to poke around or run a one-off admin command.

```bash
# Interactive shell inside the container
docker exec -it web sh
docker exec -it web bash    # if the image has bash (Debian/Ubuntu-based images)

# Run a single command and exit
docker exec web cat /etc/nginx/nginx.conf

# Run as a different user
docker exec -u root -it web sh

# Set working directory and env for the exec'd process
docker exec -w /var/log -e DEBUG=1 web ls -la
```

`exec` only works on running containers. To run something in a stopped one, start it first (`docker start`).

## Starting, stopping, restarting

```bash
# Stop gracefully (SIGTERM, then SIGKILL after 10s by default)
docker stop web

# Custom grace period
docker stop -t 30 web

# Start a stopped container
docker start web

# Restart (stop + start)
docker restart web

# Pause/unpause (freezes processes with cgroups, RAM stays allocated)
docker pause web
docker unpause web

# Send SIGKILL immediately
docker kill web

# Send an arbitrary signal
docker kill --signal=SIGHUP web
```

## Removing containers

A stopped container still occupies its writable filesystem layer and shows up in `docker ps -a`. Use `docker rm` to delete it.

```bash
# Remove a stopped container
docker rm web

# Force-remove (stops it first if running)
docker rm -f web

# Remove all stopped containers in one go
docker container prune
```

If you started the container with `--rm`, this happens automatically when it exits — preferred for one-off jobs.

## Copying files in and out

`docker cp` moves files between the host and a container in either direction. Works on both running and stopped containers.

```bash
# Copy from container to host
docker cp web:/etc/nginx/nginx.conf ./nginx.conf

# Copy from host to container
docker cp ./newconfig.conf web:/etc/nginx/nginx.conf

# Copy a whole directory
docker cp web:/var/log/nginx ./nginx-logs
```

This is convenient but not a substitute for volumes when you need persistent data — see [volumes.md](volumes.md).

## Practical examples

### 1. Disposable Postgres for development

A persistent Postgres on `localhost:5432` with data stored in a named volume. Stop it with `docker stop pg`, restart with `docker start pg`, blow it away entirely with `docker rm -f pg && docker volume rm pgdata`.

```bash
docker run -d \
  --name pg \
  --restart unless-stopped \
  -p 5432:5432 \
  -e POSTGRES_USER=dev \
  -e POSTGRES_PASSWORD=dev \
  -e POSTGRES_DB=devdb \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Connect with the psql CLI from inside the container
docker exec -it pg psql -U dev -d devdb
```

### 2. Disposable Redis

```bash
docker run -d \
  --name redis \
  --restart unless-stopped \
  -p 6379:6379 \
  redis:7-alpine

# Open a redis-cli session
docker exec -it redis redis-cli
```

### 3. One-off shell sandbox

Nothing on the host changes. The container is destroyed when you exit the shell.

```bash
docker run --rm -it alpine sh

# Or Debian if you prefer apt-style tooling
docker run --rm -it debian:bookworm-slim bash
```

### 4. Long-running web service with bind-mounted config

```bash
docker run -d \
  --name web \
  --restart unless-stopped \
  -p 80:80 \
  -v "$(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro" \
  -v "$(pwd)/site:/usr/share/nginx/html:ro" \
  nginx:1.27

# Reload config without restarting the container
docker exec web nginx -s reload
```

### 5. Building inside a one-shot container

Useful for trying a build without installing the toolchain on the host. The current directory is mounted so the build output lands on your filesystem.

```bash
docker run --rm \
  -v "$(pwd)":/src \
  -w /src \
  golang:1.22 \
  go build -o myapp ./cmd/myapp
```

## Quick reference

| Goal | Command |
|---|---|
| Run + detach + name | `docker run -d --name N image` |
| Run interactively | `docker run --rm -it image sh` |
| List running | `docker ps` |
| List all | `docker ps -a` |
| Stop / start / restart | `docker stop \| start \| restart N` |
| Shell into running | `docker exec -it N sh` |
| Show logs (follow) | `docker logs -f N` |
| Inspect | `docker inspect N` |
| Remove | `docker rm [-f] N` |
| Live stats | `docker stats` |
| Copy files | `docker cp N:/path ./local` |
