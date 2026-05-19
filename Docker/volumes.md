# Volumes and bind mounts

Containers are ephemeral — every write goes to a thin writable layer that's discarded when the container is removed. For anything you need to keep across container restarts (database files, logs, uploaded user data) or share with the host (source code mounted into a dev container, config files), you have to mount external storage. This page covers the three options Docker provides and when to use each.

## The three storage types

Docker supports three kinds of mounts. They behave differently and are managed differently.

| | Bind mount | Named volume | tmpfs |
|---|---|---|---|
| Where data lives | Any host path you choose | `/var/lib/docker/volumes/<name>/_data` (managed by Docker) | RAM only |
| Managed by Docker | No — host filesystem is the source of truth | Yes — `docker volume create/ls/rm` | N/A |
| Survives container removal | Yes (you own it) | Yes (until you `docker volume rm`) | No |
| Portable across hosts | No (path is host-specific) | Yes (the volume name is hardcoded; data has to be exported separately) | N/A |
| Performance | Native filesystem speed | Native filesystem speed | RAM — fastest |
| Typical use | Dev source mounts, host config files, log dirs you want on the host | Database state, app data that doesn't need to be on a known host path | Secrets, scratch space, never-on-disk data |

A simple rule of thumb:

- **Need to edit the data from the host?** Bind mount.
- **Pure container state you never want to look at directly?** Named volume.
- **Sensitive or scratch data that should disappear with the container?** tmpfs.

## `-v` vs `--mount` syntax

Both flags do the same job. `-v` is shorter and ubiquitous; `--mount` is verbose but more explicit (separate key=value fields, harder to typo, supports more options). Pick one style and stick with it — both are shown side-by-side below.

```bash
# Named volume (short)
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Named volume (long)
docker run --mount type=volume,source=pgdata,target=/var/lib/postgresql/data postgres:16

# Bind mount (short)
docker run -v "$(pwd)/src":/app/src postgres:16

# Bind mount (long)
docker run --mount type=bind,source="$(pwd)/src",target=/app/src postgres:16

# tmpfs (long form only)
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=64m postgres:16
```

## Named volumes

A named volume is a directory on the host that Docker creates, manages, and remembers by name. It's the right choice for stateful service data — Postgres, Redis with AOF persistence, MongoDB, message queues — where you don't need to inspect the files from outside Docker.

```bash
# Create explicitly (optional — first use also creates it)
docker volume create pgdata

# List volumes
docker volume ls

# Inspect (shows mountpoint, labels, driver)
docker volume inspect pgdata

# Where the data actually lives
sudo ls /var/lib/docker/volumes/pgdata/_data

# Remove (must not be in use by any container)
docker volume rm pgdata

# Bulk remove all unused volumes
docker volume prune
```

Mount a named volume in a container with `-v name:/path/inside`:

```bash
docker run -d --name pg \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=dev \
  postgres:16
```

The first time you mount an empty named volume onto a path that already has content in the image (e.g. Postgres's data dir on first boot), Docker copies the image's contents into the volume. Subsequent runs see the volume's contents, not the image's. **This only happens for empty volumes** — once data is there, mounting "shadows" what was in the image.

## Bind mounts

A bind mount maps a host directory or file directly into a container. The host path is the source of truth — Docker doesn't manage anything. The container sees whatever's on the host at that location, and writes from the container show up immediately on the host.

```bash
# Mount a directory (must use an absolute path)
docker run -v /home/me/project:/app node:20

# Use $(pwd) for the current directory
docker run -v "$(pwd)":/app node:20

# Mount a single file
docker run -v "$(pwd)/nginx.conf":/etc/nginx/nginx.conf nginx

# Read-only (recommended for config files)
docker run -v "$(pwd)/nginx.conf":/etc/nginx/nginx.conf:ro nginx
```

The `:ro` suffix makes the mount read-only inside the container — defense in depth for config files, certificates, and anything you don't want the container to mutate.

> **Warning:** Bind mounts only accept absolute paths. `-v ./src:/app` does NOT do what you'd hope — Docker treats `./src` as a volume name and creates an anonymous volume. Always use `$(pwd)/src` or a full path.

### SELinux relabel flags (`:z`, `:Z`)

On distros with SELinux enforced (Fedora, RHEL, CentOS), the kernel will block a container from reading bind-mounted host files unless they have the right SELinux label. Append `:z` (shared label, OK for files shared between containers) or `:Z` (private label, more restrictive) to relabel them automatically:

```bash
docker run -v "$(pwd)/data":/data:Z myimage
```

On Arch and Debian/Ubuntu, SELinux is not enforced by default, so these flags are unnecessary — included here so you recognize them if you see them in someone else's docs.

## tmpfs mounts

tmpfs mounts live in RAM and disappear when the container stops. Useful for high-throughput scratch data, sensitive material you don't want hitting disk, or to make the root filesystem read-only with `--tmpfs /tmp`.

```bash
# Mount /tmp as tmpfs with a size cap
docker run --tmpfs /tmp:size=64m alpine sh

# More flexible form
docker run --mount type=tmpfs,target=/cache,tmpfs-size=128m alpine
```

## Practical examples

### 1. Persistent Postgres data

The canonical "I want my dev database to survive container restarts" setup. The volume is named `pgdata` and lives under `/var/lib/docker/volumes/pgdata/_data` on the host — but you should never touch those files directly.

```bash
docker volume create pgdata

docker run -d \
  --name pg \
  --restart unless-stopped \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=dev \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

Stop and remove the container with `docker rm -f pg` — the volume stays. Recreate the container with the same `-v pgdata:...` and it picks up where it left off.

### 2. Live-reload development with a source bind mount

Mount your source tree into the container so file changes on the host show up instantly. Combine with a watcher inside the container (`nodemon`, `dotnet watch`, etc.) for hot-reload.

```bash
docker run --rm -it \
  -v "$(pwd)":/app \
  -w /app \
  -p 3000:3000 \
  node:20 \
  npm run dev
```

`-w /app` makes `/app` the working directory; `--rm` cleans up the container on exit; the bind mount means edits in your editor are visible to the running process.

### 3. Read-only config file

```bash
docker run -d \
  --name web \
  -p 80:80 \
  -v "$(pwd)/nginx.conf":/etc/nginx/nginx.conf:ro \
  nginx:1.27
```

The `:ro` ensures the container can read but never modify your `nginx.conf`. Editing it on the host and running `docker exec web nginx -s reload` reloads the config in place.

### 4. tmpfs for sensitive scratch

```bash
docker run --rm \
  --tmpfs /scratch:size=256m,mode=1777 \
  myimage \
  /scratch/run.sh
```

Nothing in `/scratch` ever hits the disk, and it's gone when the container exits.

## Backup and restore

Named volumes are convenient until you need to back one up. The standard trick is to launch a throwaway container that mounts the volume and a host directory, and uses `tar` to move data between them.

### Backup

```bash
# Tar the contents of `pgdata` into a host file
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
```

That produces a `pgdata-YYYY-MM-DD.tar.gz` in the current directory containing everything from `/var/lib/docker/volumes/pgdata/_data`.

### Restore

```bash
# Reverse: untar into a (possibly fresh) volume
docker run --rm \
  -v pgdata:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar xzf /backup/pgdata-2025-01-15.tar.gz -C /data
```

For Postgres specifically, prefer `pg_dump` / `pg_restore` against a running container — backups taken with `tar` while Postgres is writing can be corrupt. The `tar` trick is universally safe only when the service is stopped or the data is read-only.

## When to put `VOLUME` in a Dockerfile

You can declare a volume in a Dockerfile with `VOLUME /var/lib/myapp`. Docker will then create an **anonymous volume** for that path whenever a container is created without an explicit mount. This is mostly useful as documentation — it signals "this directory holds state and should not stay in the image's writable layer".

For most apps, prefer to let users decide and mount their own named volumes at run time. Declaring `VOLUME` in the Dockerfile also disables build-time writes to that path after the declaration, which can surprise you.

## Quick reference

| Goal | Syntax |
|---|---|
| Named volume | `-v name:/path` |
| Bind mount (rw) | `-v /abs/host:/path` |
| Bind mount (ro) | `-v /abs/host:/path:ro` |
| tmpfs | `--tmpfs /path[:size=N]` |
| Explicit form | `--mount type=bind\|volume\|tmpfs,source=...,target=...,readonly` |
| List volumes | `docker volume ls` |
| Inspect volume | `docker volume inspect NAME` |
| Remove volume | `docker volume rm NAME` |
| Cleanup unused | `docker volume prune` |
