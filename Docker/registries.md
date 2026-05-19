# Registries

A registry is the storage backend for Docker images — a server that speaks the OCI distribution protocol and holds tagged image versions you can pull from and push to. Docker Hub is the default, but you'll quickly run into others: GitHub Container Registry (GHCR), Google's GCR, Quay, AWS ECR, or your own self-hosted registry. This page covers how to authenticate to the common ones, push images to them, and the naming conventions that determine where an image actually goes.

## Image name → registry mapping

The first segment of an image name (before the first `/`) determines the registry. Docker treats it as a hostname if it contains a `.` or `:`; otherwise it falls back to Docker Hub.

| Image name | Registry | Notes |
|---|---|---|
| `nginx` | `docker.io` | Implicit `library/` namespace for official images |
| `library/nginx` | `docker.io` | Explicit form of the above |
| `myuser/myapp` | `docker.io` | User's Docker Hub repository |
| `ghcr.io/myuser/myapp` | `ghcr.io` | GitHub Container Registry |
| `gcr.io/project/myapp` | `gcr.io` | Google Container Registry |
| `quay.io/org/myapp` | `quay.io` | Red Hat Quay |
| `123456789.dkr.ecr.us-east-1.amazonaws.com/myapp` | AWS ECR | Region-specific URL |
| `registry.example.com:5000/myapp` | Your own | Hostname + port → custom registry |
| `localhost:5000/myapp` | Your own | Local self-hosted registry |

To push to a non-Docker-Hub registry, your image must be **tagged** with the full registry path. `docker push myapp:1.0` will fail because it implicitly targets Docker Hub; tag it first as `ghcr.io/me/myapp:1.0`, then push that name.

## Docker Hub

Docker Hub is the default registry and hosts most "official" images (the ones in the `library/` namespace, like `postgres`, `nginx`, `redis`). Anonymous pulls work fine but are rate-limited.

```bash
# Anonymous pulls are rate-limited per IP. Login lifts the limit:
docker login
# Username: myuser
# Password: <a Personal Access Token, NOT your account password>
```

For pushing, you need an account at hub.docker.com and a repository (free accounts get unlimited public repos, one private repo). Generate a Personal Access Token from your Hub account settings — Docker recommends using PATs over your password.

```bash
# Tag a local image for Docker Hub
docker tag myapp:1.0 myuser/myapp:1.0

# Push
docker push myuser/myapp:1.0

# Push multiple tags
docker push myuser/myapp:1.0
docker push myuser/myapp:latest
```

> **Tip:** `docker pushrm` from the `pushrm` plugin can sync your README from Git into Docker Hub. Useful if you maintain images publicly.

## GitHub Container Registry (GHCR)

GHCR (`ghcr.io`) ties image storage to GitHub repositories and accounts. It's free, has no anonymous rate limits, and integrates cleanly with GitHub Actions. Auth is via a Personal Access Token (classic, with `write:packages` scope) — your GitHub password does not work.

```bash
# Create a PAT at: github.com → Settings → Developer settings → Personal access tokens → Tokens (classic)
# Scopes: write:packages (which implies read:packages and repo)

# Login (paste the PAT when prompted, or pipe it)
echo "$GHCR_PAT" | docker login ghcr.io -u myuser --password-stdin

# Tag for GHCR (lowercase only — GHCR rejects uppercase characters in image names)
docker tag myapp:1.0 ghcr.io/myuser/myapp:1.0

# Push
docker push ghcr.io/myuser/myapp:1.0
```

By default a new GHCR package is **private**. To make it public, go to your GitHub profile → Packages → the package → Package settings → Change visibility. Linking the package to a repo (via the `org.opencontainers.image.source` label on the image) is also recommended for discoverability:

```dockerfile
LABEL org.opencontainers.image.source=https://github.com/myuser/myapp
LABEL org.opencontainers.image.description="My application"
LABEL org.opencontainers.image.licenses=MIT
```

## Private registries — generic flow

For any registry that requires authentication, the pattern is the same:

```bash
# Authenticate (credentials cached in ~/.docker/config.json)
docker login registry.example.com
docker login registry.example.com:5000  # if non-standard port

# Tag with the registry path
docker tag myapp:1.0 registry.example.com/team/myapp:1.0

# Push
docker push registry.example.com/team/myapp:1.0

# Logout when done
docker logout registry.example.com
```

Credentials are stored in plain JSON by default. On Linux you can hook in a credential helper (`pass`, `secretservice`) by setting `"credsStore": "pass"` in `~/.docker/config.json`.

## Pulling private images

Once logged in, pulls work transparently:

```bash
docker pull registry.example.com/team/myapp:1.0
```

For CI or other non-interactive contexts, mount a pre-populated `~/.docker/config.json` or pipe credentials via `--password-stdin`:

```bash
echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" --password-stdin registry.example.com
```

## What `docker login` actually stores

`docker login` writes an auth entry into `~/.docker/config.json`. By default this is the credential base64-encoded — readable to anything with filesystem access. If that's a concern, install a credential helper (one per platform: `docker-credential-pass` on Linux with `pass`, `docker-credential-osxkeychain` on macOS, `docker-credential-wincred` on Windows) and reference it via `"credsStore"`.

```json
{
  "auths": {
    "ghcr.io": {},
    "docker.io": {}
  },
  "credsStore": "pass"
}
```

## Running your own registry

The simplest registry is the official `registry:2` image. Useful as a private mirror for an offline lab, as a build-output cache, or just to understand how the protocol works.

```bash
# Run on localhost:5000 with persistent storage
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# Tag and push to it
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# Pull back
docker pull localhost:5000/myapp:1.0
```

This setup has **no authentication and no TLS** — fine for a single-host lab, not for anything reachable from elsewhere. For real use you'd add TLS via a reverse proxy (nginx, Caddy, Traefik) and basic-auth or token-based authentication; the official Docker docs have a recipe. For most "I just want a private registry" use cases, GHCR or Docker Hub's free tier is easier than running your own.

## Inspecting remote images without pulling

`docker manifest` lets you query a registry without downloading the layers — useful for "is this tag a multi-platform image?" or "what's the digest of the current `latest`?".

```bash
# Show the manifest (you may need to enable experimental features)
docker manifest inspect nginx:1.27

# For multi-arch images, see the platforms
docker manifest inspect --verbose nginx:1.27 | grep -i architecture
```

The newer `docker buildx imagetools inspect` does the same and more, and doesn't require experimental flags:

```bash
docker buildx imagetools inspect nginx:1.27
```

## Tagging conventions for registries

What you tag affects what consumers can reasonably do. A few habits that pay off:

- **Tag every release with a specific version** (`:1.2.3`), a moving minor (`:1.2`), and `:latest`.
- **Embed the git SHA** as a separate tag or label for traceability.
- **Set OCI labels** on the image — `org.opencontainers.image.source`, `.version`, `.revision`, `.licenses`, `.description`. GitHub renders these on the package page, and tools like Renovate use them.

```bash
GIT_SHA=$(git rev-parse --short HEAD)
VERSION=1.2.3

docker build \
  -t ghcr.io/me/myapp:${VERSION} \
  -t ghcr.io/me/myapp:${VERSION%.*} \
  -t ghcr.io/me/myapp:latest \
  -t ghcr.io/me/myapp:sha-${GIT_SHA} \
  --label org.opencontainers.image.revision=${GIT_SHA} \
  --label org.opencontainers.image.version=${VERSION} \
  --label org.opencontainers.image.source=https://github.com/me/myapp \
  .

docker push --all-tags ghcr.io/me/myapp
```

## Quick reference

| Goal | Command |
|---|---|
| Login (Docker Hub) | `docker login` |
| Login (other) | `docker login REGISTRY[:PORT]` |
| Logout | `docker logout [REGISTRY]` |
| Tag for push | `docker tag SRC REGISTRY/NS/REPO:TAG` |
| Push | `docker push REGISTRY/NS/REPO:TAG` |
| Push all tags of a repo | `docker push --all-tags REGISTRY/NS/REPO` |
| Inspect manifest remotely | `docker buildx imagetools inspect IMAGE` |
| Self-host | `docker run -d -p 5000:5000 registry:2` |
