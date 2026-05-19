# Security and rootless Docker

The default Docker setup makes a security trade-off most users don't think about: the daemon runs as root, anyone in the `docker` group is effectively root, and containers run as root by default. This page covers why that is, what you can do about it (rootless Docker, dropping capabilities, running as non-root inside containers), and basic hardening practices.

## Why "docker group = root"

The Docker daemon runs as root because creating containers requires kernel features (namespaces, cgroups, network setup) that need privileged access. The daemon listens on a Unix socket at `/var/run/docker.sock`, and that socket is owned by `root:docker` so anyone in the `docker` group can talk to it.

Talking to the socket means **issuing API requests to a root-owned daemon**. One of those requests is "create a container that bind-mounts `/` from the host as read-write." Once you do that, you can read every file on the system, modify `/etc/sudoers`, or just `chroot` into it.

A demonstration of the escalation (don't actually do this on a shared machine):

```bash
# Anyone in the `docker` group can read /etc/shadow without sudo
docker run --rm -v /:/host alpine cat /host/etc/shadow
```

There's no Docker config that prevents this. The trade-off is unavoidable: convenient docker-group access means trusting every member of that group with root. The mitigations are:

1. **Don't add untrusted users to the `docker` group.** Even on a personal machine, be aware that any malware running as your user has root via this path.
2. **Use rootless Docker** (next section) — the daemon runs as your user, in user namespaces, with no privileged capabilities.
3. **Use `sudo docker ...` instead of group membership** — same security level as rootless w.r.t. the daemon, but you re-authenticate each call.

## Rootless Docker

Rootless mode runs the entire Docker daemon as an unprivileged user, inside a user namespace. The daemon process can't actually be root on the host; it has its own view where it appears to be UID 0 but maps to your user outside the namespace. If a container escapes, the attacker still only has your user's privileges — not root.

### Installation on Arch

The rootless components ship in a separate package or AUR depending on the moment. Currently:

```bash
# Pull from AUR (or from extra if it's been moved by the time you read this)
yay -S docker-rootless-extras

# (Optional) install Buildx and Compose plugins as usual; they work in both modes
sudo pacman -S docker-buildx docker-compose
```

### Installation on Debian / Ubuntu

If you installed from Docker's APT repository (see [installation.md](installation.md)), rootless extras come from the same source:

```bash
sudo apt install docker-ce-rootless-extras uidmap dbus-user-session
```

### One-time setup per user

Every user that wants rootless Docker runs the setup tool once. It configures subuid/subgid ranges, installs a systemd user unit for the daemon, and starts it.

```bash
# As your normal user, NOT root
dockerd-rootless-setuptool.sh install

# Make sure the daemon comes up at login (and not just on next reboot)
systemctl --user enable --now docker

# Set DOCKER_HOST so your `docker` CLI talks to the rootless daemon
echo 'export DOCKER_HOST=unix:///run/user/'$(id -u)'/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

Test it:

```bash
docker info | grep -i rootless
# Should show: "Security Options: ... rootless"

docker run --rm hello-world
```

### Limitations of rootless

Rootless gives up some capabilities for the security gain. The biggest ones:

- **Binding to ports below 1024 needs extra setup.** Either run the container with `--cap-add NET_BIND_SERVICE` and use the `slirp4netns` port forwarder, or set `net.ipv4.ip_unprivileged_port_start=0` via `sysctl` (host-wide).
- **`--network host` doesn't share the host's network stack** — it uses the user-namespaced network, which behaves slightly differently.
- **Some filesystem operations are slower** (overlay2 over fuse-overlayfs, depending on the host kernel).
- **Cgroups v1 + rootless is unsupported.** All modern distros use cgroups v2 by default; if you're on an older one, you'll need to switch.

For most personal development and many self-hosted setups, these are fine. For production-grade hosting that needs raw performance and host networking, rootful Docker (with strict access control) or alternatives like Podman remain common.

## Running unprivileged INSIDE containers

Separately from how the daemon runs, you should consider what UID the **process inside** a container runs as. By default it's root inside the container, which is fine on the surface — but it's the same UID 0 the host sees if a container escapes (in rootful mode) or if you bind-mount sensitive host paths.

### `USER` in the Dockerfile

The clean fix is to bake a non-root user into the image. Most official images do this themselves (e.g. `nginx` runs `nginx` as `www-data`, .NET aspnet images run as `app` UID 1654). Your own images should follow suit:

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 1000 appuser
WORKDIR /home/appuser/app
COPY --chown=appuser . .
USER appuser
CMD ["python", "main.py"]
```

`USER` only affects subsequent `RUN`/`CMD`/`ENTRYPOINT` — make sure you've done all your `apt install` and file ownership setup before switching.

### `--user` at run time

You can also override the user at run time, without touching the image:

```bash
# Run as your host UID/GID — useful with bind mounts to avoid root-owned files on the host
docker run --rm -v "$(pwd)":/work -w /work -u "$(id -u):$(id -g)" alpine sh

# Use a numeric UID even if it doesn't exist in /etc/passwd inside the container
docker run --rm -u 1000:1000 alpine id
```

### Read-only root filesystem

`--read-only` makes the container's root filesystem read-only. Combined with explicit `--tmpfs` mounts for paths the app needs to write to (like `/tmp`), this is a strong defense — a compromised process can't tamper with binaries.

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  nginx
```

If the image needs to write to specific directories at runtime, mount tmpfs or named volumes there.

## Capabilities

By default a rootful container has a long but not-quite-root set of Linux capabilities (the exact list is in the Docker source — `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, etc., are dropped, but many remain). For tighter sandboxing, drop everything and add back only what you need.

```bash
# Drop everything
docker run --rm --cap-drop ALL alpine ping 8.8.8.8
# ping will fail — needs CAP_NET_RAW

# Drop everything and add back exactly one
docker run --rm --cap-drop ALL --cap-add NET_RAW alpine ping 8.8.8.8

# The classic web server: only needs NET_BIND_SERVICE to bind low ports
docker run --rm --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

Pair with `--security-opt=no-new-privileges`, which prevents setuid binaries from escalating privileges inside the container:

```bash
docker run --rm \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  nginx
```

## seccomp, AppArmor, SELinux

Docker applies a default seccomp profile that blocks ~50 syscalls (full list in the Docker source). It's reasonable for most workloads but rarely enough by itself. On Ubuntu/Debian, the AppArmor profile `docker-default` is also auto-applied; on Fedora/RHEL, SELinux confines containers via the `container_t` type.

You generally leave these alone unless you have a reason to tweak them. If you do need to relax (e.g. a debugger that needs `ptrace`), pass a custom profile:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  --cap-add SYS_PTRACE \
  alpine strace -e openat ls
```

Don't run unconfined containers in production without thought — you've turned off layers of defense.

## Image trust

The image you run is code on your machine — vet it the same way you'd vet a binary you downloaded.

- **Prefer official images** (`library/postgres`, etc.) when one exists. They're maintained by upstream and built reproducibly.
- **Pin to digests** in production manifests (`postgres@sha256:abc...`). Tags are mutable; digests are not.
- **Scan images** before deploying. The built-in `docker scout` (recent Docker versions) and third-party tools like Trivy report known CVEs in image contents:
  ```bash
  docker scout cves myapp:1.0
  # or
  trivy image myapp:1.0
  ```
- **Watch for typosquatting.** `dockerr/nginx` is not `nginx`. Always check the publisher.

## Secrets

Don't bake secrets into images. They become part of the layers and are visible via `docker history`, and pushed to whatever registry you push to.

- **Build-time** secrets: use BuildKit's `--secret` mount instead of `ARG`.
  ```bash
  DOCKER_BUILDKIT=1 docker build \
    --secret id=npmrc,src=$HOME/.npmrc \
    -t myapp .
  ```
  In the Dockerfile: `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci`. The file isn't copied into any layer.
- **Runtime** secrets: pass via env vars, files, or — best — a secret manager mounted as a tmpfs. Don't `COPY` them into the image.

## Hardening checklist

A practical list to apply or skip case-by-case. Rare to need them all; reaching for "most of them" is reasonable for anything internet-facing.

- [ ] Run as a non-root user inside the container (`USER` in Dockerfile or `--user` at run time)
- [ ] `--read-only` root filesystem with explicit `tmpfs` for writable paths
- [ ] `--cap-drop ALL` with only the capabilities you need added back
- [ ] `--security-opt no-new-privileges`
- [ ] Resource limits — `--memory`, `--cpus`, `--pids-limit`
- [ ] Bind to `127.0.0.1` (not `0.0.0.0`) for ports that don't need to be reachable from outside
- [ ] Use a user-defined network to avoid the default bridge's quirks
- [ ] Pin base images to a digest (`FROM image@sha256:...`)
- [ ] Add a `HEALTHCHECK` so orchestrators can detect liveness issues
- [ ] Rotate registry credentials; use short-lived tokens in CI
- [ ] Run rootless Docker if you're not on a multi-user host that needs the rootful daemon

## Quick reference

| Goal | Mechanism |
|---|---|
| Run daemon unprivileged | Rootless Docker (`dockerd-rootless-setuptool.sh install`) |
| Container process not root | `USER` in Dockerfile, or `--user UID:GID` at run time |
| Read-only filesystem | `--read-only --tmpfs /tmp` |
| Minimal capabilities | `--cap-drop ALL --cap-add NEEDED_CAP` |
| Block setuid escalation | `--security-opt no-new-privileges` |
| Build-time secrets | BuildKit `--secret` + `RUN --mount=type=secret,...` |
| Image vulnerability scan | `docker scout cves IMAGE`, `trivy image IMAGE` |
| Pin to immutable digest | `image@sha256:...` |
