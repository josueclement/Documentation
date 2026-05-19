# Docker Cheat Sheets

| File | Key Topics |
|------|------------|
| [`overview.md`](overview.md) | Concepts: images vs containers vs layers, registries, client/daemon model, lifecycle states |
| [`installation.md`](installation.md) | Install on Arch + Debian/Ubuntu, docker group, enable service, verify, uninstall |
| [`images.md`](images.md) | `pull`, `images`, `inspect`, `tag`, `rmi`, `search`, `save`/`load`, image naming |
| [`containers.md`](containers.md) | `run`, `ps`, `start`/`stop`/`restart`, `exec`, `logs`, `rm`, common flags, restart policies, practical examples |
| [`dockerfile.md`](dockerfile.md) | Dockerfile instructions, `.dockerignore`, layer caching, ARG vs ENV, multi-stage builds (incl. .NET example) |
| [`volumes.md`](volumes.md) | Named volumes, bind mounts, tmpfs, backup pattern, practical examples |
| [`networking.md`](networking.md) | Default bridge, user-defined networks, host/none modes, port publishing, container DNS |
| [`compose.md`](compose.md) | `docker compose` v2, `compose.yaml`, multi-service example, `.env`, override files, profiles |
| [`registries.md`](registries.md) | Docker Hub, GHCR, private registries, `login`, `push`, image naming, self-hosted registry |
| [`cleanup.md`](cleanup.md) | `system df`, `system prune`, targeted prunes, log size control, scheduled cleanup |
| [`security-rootless.md`](security-rootless.md) | docker-group caveat, rootless setup, unprivileged containers, capabilities, image trust |
