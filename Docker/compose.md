# Docker Compose

Docker Compose lets you define a multi-container application in a single YAML file and bring the whole thing up or down with one command. It replaces long `docker run` invocations and ad-hoc `docker network create` setups with a versioned, source-controllable description of your stack. It's the right tool for development environments and small self-hosted deployments — anything bigger usually moves to Kubernetes.

## v1 vs v2 — use v2

There are two implementations you may encounter:

- **`docker-compose`** (with the dash, Compose v1) — the original Python tool, now deprecated.
- **`docker compose`** (with a space, Compose v2) — a Go plugin shipped with the Docker CLI.

Always use v2 (`docker compose ...`). It's installed automatically by the Docker packages on Arch and via `docker-compose-plugin` on Debian/Ubuntu (see [installation.md](installation.md)). The two are mostly compatible at the YAML level, but v2 is faster, actively maintained, and what every example below uses.

## The Compose file

Compose reads a file named `compose.yaml` (or `compose.yml`, or the older `docker-compose.yaml`) in the current directory. Pass `-f path/to/file.yaml` to use a different one.

In v2 you no longer need a top-level `version:` key — that field was a v1 thing and is now ignored. A minimal file looks like this:

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "8080:80"
```

`docker compose up` from the same directory pulls the image, creates a network (named `<dir>_default`), creates a container (named `<dir>-web-1`), starts it, and streams its logs. `Ctrl+C` stops the container; `docker compose down` removes it.

## A real example — app + Postgres

A more representative file: one application container that depends on a Postgres database, with persistent storage, a private network, and environment variables loaded from a `.env` file. This is the foundation of most homelab stacks.

```yaml
# compose.yaml
services:
  app:
    build: .                       # Build from ./Dockerfile in this directory
    # image: ghcr.io/me/app:1.0    # Or pull a prebuilt image instead
    container_name: app
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/appdb
      LOG_LEVEL: info
    ports:
      - "8080:8080"
    networks:
      - backend

  db:
    image: postgres:16
    container_name: db
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pgdata:

networks:
  backend:
```

A few things to notice:

- **No `-p` on the database.** `db` is only reachable from `app` via the shared `backend` network. The app connects to it as `db:5432` — the service name doubles as the hostname.
- **`depends_on` with `condition: service_healthy`** waits for the Postgres healthcheck to pass before starting `app`. Plain `depends_on: [db]` only waits for the container to start, not for the service to be ready.
- **`${DB_PASSWORD}` is substituted from the environment** — typically from a `.env` file in the same directory (see below).
- **Top-level `volumes:` and `networks:`** declare the named resources used by services. Compose creates them on `up` if missing.

## The `.env` file

Compose automatically loads `.env` from the project directory and uses its values for `${...}` substitution in the YAML.

```
# .env
DB_PASSWORD=correcthorsebatterystaple
```

**Keep `.env` out of git.** Add it to `.gitignore` and commit a `.env.example` with placeholder values instead.

The `.env` file affects YAML substitution only. To make those values available **inside** containers, you still need to map them with `environment:` (as in the example above) or use `env_file:`:

```yaml
services:
  app:
    image: myapp
    env_file:
      - .env.app   # All KEY=VALUE lines become env vars in the container
```

## Common commands

`docker compose` always operates on the file in the current directory (unless overridden with `-f`). All commands are scoped to that project — they don't touch unrelated containers.

```bash
# Start everything (foreground, streams logs, Ctrl+C stops)
docker compose up

# Start in the background
docker compose up -d

# Recreate containers even if config hasn't changed
docker compose up -d --force-recreate

# Build images first if your file has `build:` services
docker compose up -d --build

# Stop containers (keeps them around for inspection)
docker compose stop

# Stop and remove containers, networks
docker compose down

# Also remove named volumes (destructive — wipes data)
docker compose down -v

# Pull newer images for all services
docker compose pull

# Show container status
docker compose ps

# Tail logs for all services
docker compose logs -f

# Logs for one service
docker compose logs -f app

# Exec into a service (uses the service name, not container name)
docker compose exec app sh
docker compose exec db psql -U app -d appdb

# Run a one-off command (creates a new container, removes it after)
docker compose run --rm app npm test

# Restart a single service
docker compose restart app

# Re-evaluate and show the merged final config (with substitutions applied)
docker compose config

# Build images without starting
docker compose build
```

## Override files

The Compose convention for environment-specific tweaks is to keep a base `compose.yaml` and layer `compose.override.yaml` (or any other named file) on top. The two are merged at runtime, with later files winning conflicts.

By default Compose automatically merges `compose.yaml` + `compose.override.yaml` if the latter exists. For other names, pass them with multiple `-f` flags:

```bash
# Production: base file only
docker compose -f compose.yaml up -d

# Development: base + override
docker compose -f compose.yaml -f compose.dev.yaml up -d
```

A typical `compose.dev.yaml` mounts source code as a bind mount and exposes ports to localhost only:

```yaml
# compose.dev.yaml
services:
  app:
    build:
      context: .
      target: dev          # Multi-stage builds: stop at the dev stage
    volumes:
      - ./src:/app/src     # Live source mount
    environment:
      LOG_LEVEL: debug
    ports:
      - "127.0.0.1:8080:8080"
```

## Profiles for optional services

Long files often grow services that aren't always needed (admin UIs, monitoring). Tag them with a profile so they only start when explicitly enabled:

```yaml
services:
  app:
    image: myapp

  db:
    image: postgres:16

  pgadmin:
    image: dpage/pgadmin4
    profiles: [tools]      # Only starts when this profile is active
```

```bash
docker compose up -d                 # Just app + db
docker compose --profile tools up -d # Adds pgadmin
```

## Useful patterns

A few recipes that come up repeatedly:

### Restart a misbehaving service

```bash
docker compose restart app
# Or recreate from scratch (e.g. after an image update)
docker compose pull app
docker compose up -d --force-recreate app
```

### Wipe and rebuild a stateful service

For when you want a fresh database while leaving everything else alone.

```bash
docker compose stop db
docker compose rm -f db
docker volume rm $(basename $PWD)_pgdata   # the named volume
docker compose up -d db
```

### Validate the file before applying

`docker compose config` parses and substitutes everything without touching containers. Great for spotting typos and seeing the final shape after overrides.

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
```

## When NOT to use Compose

Compose is for a single host. The moment you need multiple hosts, rolling deploys, autoscaling, or anything resembling a production cluster, the tool you want is Kubernetes (or Swarm, if you want to stay in the Docker family). Compose remains the right answer for development and for small/medium self-hosted setups, but don't try to push it past that.

## Quick reference

| Command | Purpose |
|---|---|
| `docker compose up [-d]` | Create and start everything |
| `docker compose down [-v]` | Stop and remove (optionally also volumes) |
| `docker compose ps` | List project containers |
| `docker compose logs -f [SVC]` | Tail logs |
| `docker compose exec SVC CMD` | Run command in running service |
| `docker compose run --rm SVC CMD` | Run command in a one-off container |
| `docker compose build [SVC]` | Build images |
| `docker compose pull [SVC]` | Pull images |
| `docker compose restart [SVC]` | Restart service(s) |
| `docker compose config` | Show merged config after substitution |
| `docker compose --profile P up` | Start including profile P |
