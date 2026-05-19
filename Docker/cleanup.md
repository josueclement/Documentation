# Cleanup and disk usage

Docker accumulates state aggressively — stopped containers, unused images, dangling layers, build cache, named volumes that no container references, untagged intermediate images from rebuilds. Without periodic cleanup, `/var/lib/docker` can easily grow to tens of gigabytes. This page covers how to see what's eating disk and how to free it back up, from light pruning to nuclear reset.

## See what's using disk

`docker system df` is the first thing to run when you think Docker is bloated. It shows totals by resource type, including how much is reclaimable.

```bash
docker system df
```

Sample output:

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          42        8         12.3GB    9.1GB (74%)
Containers      15        4         287MB     201MB (70%)
Local Volumes   23        6         4.5GB     2.1GB (46%)
Build Cache     128       0         3.2GB     3.2GB
```

For per-resource detail:

```bash
docker system df -v
```

That breaks down each image, container, and volume with its size, so you can spot a single 6 GB layer or a forgotten 8 GB Postgres volume.

## The big hammer — `docker system prune`

`docker system prune` removes unused containers, networks, dangling images, and build cache in one shot. By default it leaves named volumes alone (they often hold data you actually want) and only deletes **dangling** images — untagged ones left behind by rebuilds — not tagged images that simply aren't running.

```bash
# Interactive, asks for confirmation
docker system prune

# No prompt
docker system prune -f

# Also remove ALL unused images (not just dangling)
docker system prune -a

# Also remove unused named volumes (DESTRUCTIVE — wipes data)
docker system prune -a --volumes
```

The `-a --volumes` form is the closest thing to "reset Docker to clean" without uninstalling. It will delete:

- All stopped containers
- All networks not used by a container
- All images not used by a container (regardless of tag)
- All build cache
- All named volumes not used by a container

Anything **currently running** is preserved. If you also want to wipe what's running, `docker stop $(docker ps -q)` first.

> **Warning:** `--volumes` deletes named volumes containing real data. If you have a Postgres or other stateful container that's stopped (not removed) and its volume isn't in use anywhere else, this will wipe it. Back up first; see the backup pattern in [volumes.md](volumes.md).

## Targeted prunes

For finer-grained control, each resource type has its own prune subcommand. They take the same `-f` (no prompt) and `--filter` flags.

### Containers

Removes all stopped containers. Running containers are left alone.

```bash
docker container prune

# Filter: only containers that exited more than 24 hours ago
docker container prune --filter "until=24h"
```

### Images

By default removes only **dangling** images (no tags, no children). `-a` removes any image not currently used by a container.

```bash
docker image prune

# Remove all unused images, not just dangling
docker image prune -a

# Filter by age
docker image prune -a --filter "until=720h"   # older than 30 days

# Filter by label
docker image prune --filter "label!=keep"
```

### Volumes

Removes named volumes not attached to any container. By default it's interactive.

```bash
docker volume prune

# Remove anonymous volumes too (Docker 23+)
docker volume prune -a

# Filter by label
docker volume prune --filter "label!=keep"
```

### Networks

Removes user-defined networks with no containers attached. Built-in networks (`bridge`, `host`, `none`) are never touched.

```bash
docker network prune
```

### Build cache

The Docker build cache (especially BuildKit's) can balloon over time. `docker builder prune` clears it. By default it keeps the most recently used entries; `-a` removes everything.

```bash
docker builder prune

# Clear everything
docker builder prune -a

# Limit by size — keep cache under 5 GB
docker builder prune --keep-storage 5GB
```

## Logs and disk growth

Container `stdout`/`stderr` is captured by the default `json-file` log driver and grows without limit. A single chatty container can fill the disk over time. Two ways to control this.

### Per-container

```bash
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name web \
  nginx
```

That caps the log at 10 MB and rotates between 3 files (so 30 MB total).

### Globally via `daemon.json`

Apply the same limits to every container by default. Edit `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Then `sudo systemctl restart docker`. New containers pick up the new defaults; existing ones keep their old settings until recreated.

To find a noisy container's log file directly:

```bash
docker inspect --format '{{.LogPath}}' web
# Returns something like /var/lib/docker/containers/abc.../abc...-json.log
sudo ls -lh "$(docker inspect --format '{{.LogPath}}' web)"
```

To truncate it without restarting the container (don't `rm` — Docker keeps a file descriptor open):

```bash
sudo truncate -s 0 "$(docker inspect --format '{{.LogPath}}' web)"
```

## Removing specific resources

When you know exactly what to delete, target it directly instead of pruning:

```bash
# All stopped containers (one-liner)
docker rm $(docker ps -aq -f status=exited)

# All containers (stopped and running) — force-stops the running ones
docker rm -f $(docker ps -aq)

# All images
docker rmi $(docker images -q)

# All volumes (will skip ones in use)
docker volume rm $(docker volume ls -q)
```

These idioms are common in scripts. They fail noisily if there's nothing to delete (the subshell returns empty), which is usually fine.

## Reclaiming disk from a specific image

If you know a specific image is huge and you want it gone — but `docker rmi` says it's in use — find what's using it:

```bash
# Show every container that uses an image
docker ps -a --filter "ancestor=postgres:15"
```

Stop and remove those containers, then retry `docker rmi`.

For an even more aggressive reset of a single repo's worth of images:

```bash
# Remove every tag of `myapp`
docker images "myapp" -q | xargs -r docker rmi -f
```

## Where the disk actually goes

If you're curious about which on-disk file holds what, Docker stores everything under `/var/lib/docker`:

```
/var/lib/docker/
├── containers/    # per-container metadata, logs, configs
├── image/         # image metadata
├── overlay2/      # image layers + writable container layers (the big one)
├── volumes/       # named volumes' data
├── network/       # network state
└── buildkit/      # build cache
```

You can `du -sh /var/lib/docker/*` (with `sudo`) to confirm which subtree is largest. But never delete files there directly — always go through `docker` commands. The state files in `containers/` and `image/` will get out of sync with what's on disk and the daemon will misbehave or refuse to start.

## Scheduled cleanup

For a server or daily-driver, a cron entry or systemd timer that runs `docker system prune -af --filter "until=168h"` weekly keeps things manageable without losing recently-built images. Adjust the `until` window to your tolerance for re-pulling.

```bash
# Example cron entry (run as a user in the docker group, weekly at 4am Sunday)
0 4 * * 0 docker system prune -af --filter "until=168h" >/dev/null 2>&1
```

## Quick reference

| Command | Effect |
|---|---|
| `docker system df` | Show disk usage summary |
| `docker system df -v` | Per-resource breakdown |
| `docker system prune` | Remove stopped containers, unused networks, dangling images, build cache |
| `docker system prune -a` | Above + all unused images |
| `docker system prune -a --volumes` | Above + unused named volumes (destructive) |
| `docker container prune` | Remove stopped containers |
| `docker image prune [-a]` | Remove dangling \| all unused images |
| `docker volume prune [-a]` | Remove unused volumes |
| `docker network prune` | Remove unused networks |
| `docker builder prune [-a]` | Clear build cache |
