# Installing Docker

How to install Docker Engine on Arch Linux and on Debian/Ubuntu, set it up so your user can run `docker` without `sudo`, verify the installation, and uninstall cleanly. Docker Desktop is a separate product (a GUI plus a Linux VM, useful on Windows/macOS) and is not covered here — on a Linux host you want the engine directly.

## Arch Linux

Arch ships Docker in the `extra` repository, so the install is a one-liner. The `docker` package brings in `containerd` and `runc` as dependencies.

```bash
# Install Docker Engine
sudo pacman -S docker

# Optional: install the Buildx and Compose plugins as separate packages
sudo pacman -S docker-buildx docker-compose
```

The daemon (`docker.service`) is not started by default. Enable it so it comes up at boot and start it now in one go.

```bash
# Enable + start the daemon immediately
sudo systemctl enable --now docker.service

# Check it's running
sudo systemctl status docker.service
```

You can stop here and use Docker with `sudo docker ...` for every command, but most people add their user to the `docker` group instead. **Be aware that this is equivalent to giving the user root access** to the system — see [security-rootless.md](security-rootless.md) for the why and for rootless mode if you're not comfortable with that.

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# The new group membership only applies to new login sessions.
# Either log out and back in, or refresh the current shell with:
newgrp docker
```

## Debian / Ubuntu

The version in the default Debian/Ubuntu repositories (`docker.io`) is often months behind. For the current engine plus Buildx and Compose v2, install from Docker's official APT repository. The steps below work on Debian 12 (bookworm) and Ubuntu 22.04/24.04.

First, remove any older or distro-shipped packages that would conflict:

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

Set up Docker's APT repository (HTTPS support, GPG key, sources list):

```bash
# Prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Trust Docker's signing key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release && echo "$ID")/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repo (this expands to the right URL for Debian or Ubuntu automatically)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/$(. /etc/os-release && echo "$ID") \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install the engine plus the Buildx and Compose plugins:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin
```

Unlike on Arch, the Debian/Ubuntu packages enable and start `docker.service` automatically. Verify with `systemctl status docker`. The post-install steps for the `docker` group are identical to Arch:

```bash
sudo usermod -aG docker $USER
newgrp docker   # or log out and back in
```

> **Note:** If you prefer the distro-shipped package and don't care about the version lag, `sudo apt install docker.io docker-compose-v2` works on recent Debian/Ubuntu and avoids the third-party repo entirely.

## Verify the installation

These three commands confirm that the daemon is up, your CLI can talk to it, and a container can be created and run end-to-end. Run them as your normal user (after the group setup above) — no `sudo` should be required.

```bash
# Show the client and daemon versions
docker version

# Show daemon info: storage driver, cgroup driver, kernel version, etc.
docker info

# Run a trivial container that prints a hello message and exits
docker run --rm hello-world
```

`docker run --rm hello-world` pulls the `hello-world` image from Docker Hub on the first run, creates a container, runs it, and the `--rm` flag removes the stopped container afterwards. If you see "Hello from Docker!" you're done.

## Uninstall

### Arch Linux

`-Rns` removes the package, its unneeded dependencies, and any backup config files. Docker's data (images, containers, volumes) lives under `/var/lib/docker` and is not removed by the package manager — delete it manually if you want a clean slate.

```bash
# Stop the daemon
sudo systemctl disable --now docker.service

# Remove the package(s)
sudo pacman -Rns docker docker-buildx docker-compose

# (Optional) Wipe all images, containers, volumes, and networks
sudo rm -rf /var/lib/docker /var/lib/containerd
```

### Debian / Ubuntu

```bash
# Stop the daemon
sudo systemctl disable --now docker.service

# Remove the engine and plugins
sudo apt purge -y docker-ce docker-ce-cli containerd.io \
                  docker-buildx-plugin docker-compose-plugin

# (Optional) Wipe persisted data
sudo rm -rf /var/lib/docker /var/lib/containerd

# (Optional) Remove the APT source and key
sudo rm /etc/apt/sources.list.d/docker.list /etc/apt/keyrings/docker.gpg
```

## Common post-install tweaks

A few daemon settings are worth knowing about. They live in `/etc/docker/daemon.json` (create the file if it doesn't exist) and require `sudo systemctl restart docker` to take effect.

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "/var/lib/docker"
}
```

- **`log-opts`** caps the size of per-container log files so they don't fill the disk over time. See [cleanup.md](cleanup.md) for more.
- **`data-root`** moves Docker's storage to another disk. Useful if `/var` is on a small partition.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Daemon not running (`systemctl start docker`) or your user is not in the `docker` group (or you didn't log out/in after `usermod`). |
| `permission denied while trying to connect to the Docker daemon socket` | Same as above — group membership not active in current shell. Run `newgrp docker` or open a new terminal. |
| `Error response from daemon: pull access denied` | The image name is wrong, the tag doesn't exist, or it's a private image — see [registries.md](registries.md) for `docker login`. |
| Daemon won't start, journal shows cgroup errors | Old kernel without cgroup v2, or conflicting cgroup driver in `daemon.json`. Run `docker info | grep -i cgroup` and align with what systemd uses. |
