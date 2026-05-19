# Images

An image is a versioned, read-only filesystem snapshot from which containers are created. This page covers working with images already published on a registry — pulling, listing, inspecting, retagging, and removing. Building your own images is in [dockerfile.md](dockerfile.md); pushing to and authenticating against a registry is in [registries.md](registries.md).

## Image name anatomy

Before running any commands, it helps to know what a fully-qualified image name looks like. Docker is permissive — most parts are optional — but understanding the full form prevents confusion.

```
[REGISTRY/[NAMESPACE/]]REPOSITORY[:TAG][@DIGEST]
```

| Part | Example | Default if omitted |
|---|---|---|
| Registry | `ghcr.io` | `docker.io` (Docker Hub) |
| Namespace | `myorg` | `library` (only on Docker Hub, for official images) |
| Repository | `postgres` | (required) |
| Tag | `:16` | `:latest` |
| Digest | `@sha256:abc…` | (none) |

So `postgres:16` is actually `docker.io/library/postgres:16`, and `ghcr.io/myorg/myapp:1.2` has all parts explicit. Tags are mutable pointers — `postgres:16` today and `postgres:16` in three months may be different bytes. For byte-stable references, use the digest form.

## Pulling images

`docker pull` downloads an image (all its layers) from a registry into your local image store. Pulling is implicit when you `docker run` an image you don't have yet, but pulling explicitly is useful for pre-fetching, for getting the latest version of a tag, or for testing connectivity to a private registry.

```bash
# Pull an official image (latest tag)
docker pull nginx

# Pull a specific tag
docker pull postgres:16

# Pull from a different registry
docker pull ghcr.io/myorg/myapp:1.2

# Pull a specific platform (useful on ARM machines pulling amd64 images)
docker pull --platform linux/amd64 alpine:3.20

# Pull every tag of a repository (rarely what you want)
docker pull --all-tags nginx
```

> **Tip:** Docker Hub anonymous pulls are rate-limited (100/6h per IP at time of writing). If you hit the limit, `docker login` first — authenticated pulls have a higher quota.

## Listing local images

`docker images` (alias of `docker image ls`) shows what's already in your local store. Each row is one image, with its name, tag, content-addressed ID, age, and on-disk size.

```bash
# All images
docker images

# Filter to a specific repository
docker images postgres

# Only image IDs (handy for scripting)
docker images -q

# Include intermediate / dangling images (untagged)
docker images -a

# Filter by predicate
docker images --filter "dangling=true"
docker images --filter "since=postgres:15"

# Custom output format (Go template)
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

A "dangling" image is one with no tag pointing to it — typically left behind after rebuilding an image with the same tag. Cleaning these up is covered in [cleanup.md](cleanup.md).

## Inspecting an image

`docker inspect` returns the full JSON metadata for an image: layer digests, env vars, default command, exposed ports, labels, build history. It's the source of truth when you're wondering "what does this image actually do when I run it?".

```bash
# Full JSON dump
docker inspect postgres:16

# Pull out specific fields with --format (Go template)
docker inspect --format '{{.Config.Cmd}}' postgres:16
docker inspect --format '{{.Config.Env}}' postgres:16
docker inspect --format '{{.Config.ExposedPorts}}' postgres:16

# Show the build history (one line per layer)
docker history postgres:16

# Same but with the full command, no truncation
docker history --no-trunc postgres:16
```

`docker history` is especially useful when reverse-engineering someone else's image — you can see what each layer did at build time.

## Searching Docker Hub

`docker search` queries Docker Hub from the command line. It's good for discovering whether an official image exists for some piece of software; for anything more, use the web UI.

```bash
# Search by keyword
docker search nginx

# Limit results
docker search --limit 5 redis

# Only official images
docker search --filter "is-official=true" python
```

## Retagging an image

`docker tag` creates an additional name pointing to an existing image. The image data is not duplicated — it's just a new pointer. The most common reason is preparing an image for push to a private registry, since the registry must appear in the name.

```bash
# Alias an existing image with a new name
docker tag myapp:dev myapp:1.0.0

# Tag for push to GHCR
docker tag myapp:1.0.0 ghcr.io/myorg/myapp:1.0.0

# Tag with both a version and `latest`
docker tag myapp:1.0.0 myapp:latest
```

After tagging, `docker images` will show both names — they share the same image ID. See [registries.md](registries.md) for what to do with the new tag.

## Removing images

`docker rmi` (alias `docker image rm`) removes an image from the local store. The image's layers are deleted only once no other image still references them.

```bash
# Remove by name+tag
docker rmi postgres:15

# Remove by image ID
docker rmi 9d6a8c1f0d3e

# Force-remove even if containers reference it (containers must be stopped)
docker rmi -f myapp:dev

# Remove every dangling image at once
docker image prune

# Remove every unused image (dangling AND tagged but not in use)
docker image prune -a
```

`docker rmi` fails if any container (running or stopped) is using the image. Either remove the container first (`docker rm`) or use `-f`. Bulk cleanup is covered in [cleanup.md](cleanup.md).

## Saving and loading images as tarballs

Two underused commands that come in handy when you need to move an image between hosts without a registry, or to archive a specific image.

```bash
# Save an image (or several) to a tar archive
docker save -o myapp-1.0.0.tar myapp:1.0.0

# Load an image from a tar archive
docker load -i myapp-1.0.0.tar

# Save and pipe through gzip in one go
docker save myapp:1.0.0 | gzip > myapp-1.0.0.tar.gz
```

Note the distinction from `docker export`/`docker import`, which operate on **containers** (flattened filesystem snapshots) rather than images and lose metadata like `CMD` and `ENV`. For images, always use `save`/`load`.

## Quick reference

| Command | Purpose |
|---|---|
| `docker pull NAME[:TAG]` | Download an image from a registry |
| `docker images [REPO]` | List local images |
| `docker inspect NAME` | Full JSON metadata |
| `docker history NAME` | Layer-by-layer build history |
| `docker search TERM` | Search Docker Hub |
| `docker tag SRC TARGET` | Add another name to an image |
| `docker rmi NAME` | Delete an image |
| `docker image prune [-a]` | Bulk cleanup of unused images |
| `docker save -o FILE NAME` | Export image to tar |
| `docker load -i FILE` | Import image from tar |
