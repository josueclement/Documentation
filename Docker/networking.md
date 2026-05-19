# Networking

Every container gets its own network namespace — a virtual network stack isolated from the host and from other containers. Docker provides a small set of network drivers to connect them: the default `bridge`, user-defined bridges (recommended for anything with more than one container), `host` (no isolation), and `none` (no network). This page covers what you need 90% of the time; advanced overlay networks (Swarm, multi-host) are out of scope.

## Network drivers at a glance

| Driver | What it does | When to use |
|---|---|---|
| `bridge` (default) | Containers get IPs on a private virtual L2 network. Outbound traffic NATs through the host. | The default for everything; the **user-defined** flavor is recommended. |
| `host` | No network namespace — the container shares the host's network stack directly. | When you need raw network performance or to listen on a host port without `-p`. Linux only. |
| `none` | Network namespace exists but has no interfaces. | When you want zero network access (sandbox, offline batch jobs). |
| `overlay` | L2 spanning multiple Docker hosts. | Swarm clusters only — not covered here. |
| `macvlan` | Container gets its own MAC address on the physical LAN. | Rare; when you need containers to appear as first-class devices on the local network. |

## The default bridge — and why you should not use it

When you install Docker, it creates a network called `bridge` (the driver and the network share the name). Any container started without `--network` attaches to it. Containers on the default bridge can reach the internet via NAT and can be reached from the host on their published ports.

But the default bridge has one quirk that bites everyone eventually: **containers on it can't resolve each other by name**. They see each other by IP, but `docker run` doesn't update `/etc/hosts` or run a DNS server on the default bridge. You'd have to inspect each container's IP and hardcode it — fragile and ugly.

The fix is trivial: create a user-defined bridge network. Containers on the same user-defined bridge resolve each other by container name via Docker's embedded DNS. That alone makes it the right default.

## Working with user-defined networks

```bash
# Create a user-defined bridge network
docker network create appnet

# With explicit options (subnet, gateway, driver)
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  --gateway 172.28.0.1 \
  appnet

# List networks
docker network ls

# Inspect — shows containers attached, IPAM config, options
docker network inspect appnet

# Connect an already-running container
docker network connect appnet web

# Disconnect
docker network disconnect appnet web

# Remove a network (must have no containers attached)
docker network rm appnet

# Bulk cleanup of unused networks
docker network prune
```

Attach a container to a network at run time with `--network`:

```bash
docker run -d --name db --network appnet -e POSTGRES_PASSWORD=dev postgres:16
docker run -d --name web --network appnet -p 8080:80 myapp:1.0
```

Inside `web`, the database is reachable simply as `db:5432`. Docker's embedded DNS resolves `db` to the database container's IP automatically.

A container can be attached to multiple networks. This is occasionally useful — e.g. an app on both a public-facing network and a private DB-only network — but adds confusion, so prefer to keep things on one network when you can.

## Port publishing

By default, container ports are only reachable from other containers on the same network. To expose a port to the **host** (and from there, to the outside world), publish it with `-p` at run time.

```bash
# Publish container port 80 on host port 8080, all interfaces
docker run -p 8080:80 nginx

# Bind only to 127.0.0.1 (not exposed beyond the host)
docker run -p 127.0.0.1:8080:80 nginx

# Bind to a specific interface
docker run -p 192.168.1.10:8080:80 nginx

# Map a UDP port (default is TCP)
docker run -p 53:53/udp coredns/coredns

# Let Docker pick a random free host port
docker run -p 80 nginx
# Then ask what it picked
docker port <container> 80

# Publish all EXPOSE'd ports to random host ports
docker run -P nginx
```

> **Warning:** `-p 8080:80` binds to **all** host interfaces by default, which means the container is reachable from your LAN. Bind to `127.0.0.1` (`-p 127.0.0.1:8080:80`) for services you only want accessible from the local machine. Docker on Linux also bypasses the host firewall by inserting iptables rules directly — `ufw deny 8080` will NOT block a published container port.

`EXPOSE` in a Dockerfile is documentation only — it does not publish anything. You still need `-p` at run time. `-P` (capital P) does honor `EXPOSE` and publishes every documented port to a random host port.

## Container-to-container communication

The whole point of a user-defined bridge: containers on it can address each other by name. To demonstrate end-to-end, here's an app + db pair sharing a network.

```bash
# Create the network
docker network create appnet

# Database on the network — note no -p, only the app needs to reach it
docker run -d \
  --name db \
  --network appnet \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Application on the same network; uses `db` as the hostname
docker run -d \
  --name web \
  --network appnet \
  -p 8080:8080 \
  -e DATABASE_URL="postgres://app:secret@db:5432/appdb" \
  myapp:1.0
```

The `web` container's `DATABASE_URL` references `db` — that's the **container name**, which Docker's embedded DNS resolves on `appnet`. The database does not need `-p`; only `web` is reachable from the host (on `localhost:8080`).

For more than one or two services, switch to Docker Compose, which declares the network implicitly — see [compose.md](compose.md).

## Host network mode

`--network host` skips the network namespace altogether — the container uses the host's network stack directly. It's faster (no NAT, no veth pair) and the container sees the host's IPs and interfaces, but you lose isolation and `-p` has no effect (the container listens on host ports directly).

```bash
docker run -d --network host nginx
# nginx is now listening on host port 80, no -p needed
```

This only works as you'd expect on Linux. On Docker Desktop (Mac/Windows), `host` mode still goes through the Linux VM's networking. Use it sparingly — most apps don't need it, and it makes port conflicts more likely.

## None network mode

`--network none` gives the container no network interfaces except `lo`. Useful for completely offline jobs.

```bash
docker run --rm --network none alpine ip addr
# Only `lo` is present
```

## Diagnostics inside a container

When a container can't reach where you expect, drop into it and check. Most minimal images don't have networking tools installed, so use a debugging image alongside or install on the fly.

```bash
# Quick diagnostics in an alpine container on the same network
docker run --rm -it --network appnet alpine sh
# Inside:
ping db
nslookup db
nc -zv db 5432

# In an already-running container that has bash/curl/etc.
docker exec -it web sh
nslookup db
curl -v http://db:5432
```

`docker network inspect appnet` shows everything currently attached, with IPs — invaluable when DNS isn't working.

## Quick reference

| Goal | Command |
|---|---|
| Create user-defined bridge | `docker network create NAME` |
| List networks | `docker network ls` |
| Inspect | `docker network inspect NAME` |
| Attach container at start | `docker run --network NAME ...` |
| Attach later | `docker network connect NAME CONTAINER` |
| Disconnect | `docker network disconnect NAME CONTAINER` |
| Remove network | `docker network rm NAME` |
| Cleanup unused | `docker network prune` |
| Publish port | `-p [HOST_IP:]HOST:CONTAINER[/PROTO]` |
| Random host port | `-p CONTAINER` |
| Publish all EXPOSEd | `-P` |
| Show published ports | `docker port CONTAINER` |
| Host networking | `--network host` |
| No networking | `--network none` |
