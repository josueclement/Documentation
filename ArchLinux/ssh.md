# SSH

Key generation and management, client config, local/remote/dynamic tunnels, agent forwarding, `rsync`/`scp` file transfers, and `sshd` server hardening.

## Key Management

SSH keys provide passwordless authentication that's both more secure and more convenient than passwords. The recommended algorithm is **Ed25519** — it's fast, secure, and produces short keys.

### Generating Keys

```bash
# Generate a new key pair (Ed25519 — recommended)
ssh-keygen -t ed25519 -C "jo@hostname"

# Generate RSA key (if Ed25519 not supported by remote server)
ssh-keygen -t rsa -b 4096 -C "jo@hostname"

# Generate key with custom path
ssh-keygen -t ed25519 -f ~/.ssh/mykey

# Change passphrase on an existing key
ssh-keygen -p -f ~/.ssh/id_ed25519
```

### Inspecting Keys

```bash
# Show key fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# Show fingerprint in MD5 format
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E md5

# Show fingerprint in SHA256 format
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E sha256
```

### Deploying Public Keys

The public key must be added to `~/.ssh/authorized_keys` on the remote server.

```bash
# Copy public key to a remote server (easiest method)
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/mykey.pub user@host

# Manual copy (if ssh-copy-id unavailable)
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# List authorized keys on remote
ssh user@host "cat ~/.ssh/authorized_keys"
```

## SSH Config

`~/.ssh/config` lets you define per-host settings — hostname, user, port, key, and more. This eliminates long `ssh` commands and enables features like jump hosts and connection multiplexing.

```conf
# Default settings for all hosts
Host *
    AddKeysToAgent yes
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Shortcut for a specific host
Host myserver
    HostName 192.168.1.100
    User jo
    Port 2222
    IdentityFile ~/.ssh/mykey

# Jump through a bastion host (ProxyJump replaces ProxyCommand)
Host internal
    HostName 10.0.0.5
    User jo
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jo
    IdentityFile ~/.ssh/bastion_key

# Wildcard matching for a domain
Host *.dev.example.com
    User deployer
    IdentityFile ~/.ssh/dev_key

# Connection multiplexing (reuse TCP connection for multiple sessions)
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

```bash
# Create the sockets directory for multiplexing
mkdir -p ~/.ssh/sockets

# Test SSH config (show resolved config for a host)
ssh -G myserver

# Connect using config alias
ssh myserver
```

## SSH Tunnels

Tunnels forward traffic through an encrypted SSH connection. Useful for accessing remote services securely, bypassing firewalls, or creating SOCKS proxies.

### Local Port Forwarding

Makes a remote service available on a local port. Traffic flows: local port → SSH → remote service.

```bash
# Access remote port 80 via local port 8080
ssh -L 8080:localhost:80 user@remote

# Access a service behind the remote host (through the tunnel)
# Local:5432 → db.internal:5432 via remote
ssh -L 5432:db.internal:5432 user@remote
```

### Remote Port Forwarding

Exposes a local service on the remote host's port. Traffic flows: remote port → SSH → local service.

```bash
# Make local port 3000 available as remote port 9090
ssh -R 9090:localhost:3000 user@remote
```

### Dynamic Port Forwarding (SOCKS Proxy)

Creates a SOCKS5 proxy — route any application's traffic through the SSH connection.

```bash
# Create SOCKS proxy on local port 1080
ssh -D 1080 user@remote
# Then configure browser/app to use SOCKS5 proxy localhost:1080
```

### Background Tunnels

Run a tunnel in the background without an interactive shell.

```bash
# Tunnel in background (no shell)
ssh -fNL 8080:localhost:80 user@remote

# Flags:
# -f  background after authentication
# -N  no remote command (tunnel only)

# Kill a background tunnel
pkill -f "ssh -fN.*8080"
```

## SSH Agent

The SSH agent holds decrypted private keys in memory so you don't have to type your passphrase repeatedly. Most desktop environments start an agent automatically.

```bash
# Start agent manually (usually not needed on desktop)
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519

# Add with timeout (key expires from agent after N seconds)
ssh-add -t 3600 ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove a key
ssh-add -d ~/.ssh/id_ed25519

# Remove all keys
ssh-add -D
```

### Agent Forwarding

Agent forwarding lets you use your local key on a remote host to authenticate to a third host — without copying your private key to the remote.

```bash
# Connect with agent forwarding enabled
ssh -A user@bastion
# Then from bastion: ssh user@internal (uses your local key)
```

> **Warning:** Agent forwarding (`-A`) exposes your keys to the remote host's admin. Prefer `ProxyJump` instead when possible.

## Common Operations

Everyday SSH usage patterns.

### Connecting

```bash
# Basic connection
ssh user@host
ssh -p 2222 user@host
```

### Running Remote Commands

```bash
# Run a single remote command
ssh user@host "uname -a"

# Run multiple commands
ssh user@host "cd /var/log && tail -20 syslog"

# Run with pseudo-terminal (needed for interactive commands like sudo)
ssh -t user@host "sudo systemctl restart nginx"
```

### Piping Data

```bash
# Pipe local input to remote command
cat backup.sql | ssh user@host "psql -d mydb"
tar czf - /data | ssh user@host "cat > /backups/data.tar.gz"

# Pipe remote output to local command
ssh user@host "tar czf - /data" > data.tar.gz
```

### Debugging

```bash
# Verbose output (increasing levels of detail)
ssh -v user@host
ssh -vv user@host
ssh -vvv user@host
```

### Escape Sequences

While in an SSH session, type these after a newline:

| Sequence | Action |
|----------|--------|
| `~.` | Disconnect (kill hung session) |
| `~^Z` | Suspend SSH client |
| `~#` | List forwarded connections |
| `~?` | Show help |

## rsync over SSH

`rsync` efficiently transfers files by only sending differences. It's the preferred tool for backups and deployments over SSH.

### Basic Transfers

The trailing `/` on the source path matters: with `/` it copies the contents, without `/` it copies the directory itself.

```bash
# Copy to remote
rsync -avz /local/dir/ user@host:/remote/dir/

# Copy from remote
rsync -avz user@host:/remote/dir/ /local/dir/

# With custom SSH port
rsync -avz -e "ssh -p 2222" /local/dir/ user@host:/remote/dir/
```

### Previewing and Controlling Transfers

```bash
# Dry run (preview changes without transferring)
rsync -avzn /local/dir/ user@host:/remote/dir/

# Delete remote files not present locally (mirror mode)
rsync -avz --delete /local/dir/ user@host:/remote/dir/

# Show progress
rsync -avz --progress /local/dir/ user@host:/remote/dir/

# Limit bandwidth (KB/s)
rsync -avz --bwlimit=1000 /local/dir/ user@host:/remote/dir/

# Resume interrupted transfer
rsync -avz --partial --progress large-file user@host:/dest/
```

### Filtering

```bash
# Exclude patterns
rsync -avz --exclude='.git' --exclude='node_modules' src/ user@host:/deploy/

# Include/exclude rules from file
rsync -avz --filter='merge rsync-filter.txt' src/ user@host:/deploy/
```

### Common Flags

| Flag | Meaning |
|------|---------|
| `-a` | Archive (recursive, preserves permissions, times, symlinks, etc.) |
| `-v` | Verbose |
| `-z` | Compress during transfer |
| `-P` | `--partial --progress` combined |
| `-n` | Dry run |
| `-h` | Human-readable sizes |

## scp (Simple Copy)

`scp` copies files over SSH. It's simpler than `rsync` but doesn't support resume, delta transfers, or filtering.

```bash
# Copy file to remote
scp file.txt user@host:/remote/path/

# Copy file from remote
scp user@host:/remote/file.txt /local/path/

# Copy directory recursively
scp -r /local/dir user@host:/remote/

# Custom port
scp -P 2222 file.txt user@host:/remote/

# Preserve timestamps and permissions
scp -p file.txt user@host:/remote/
```

> **Note:** Prefer `rsync` over `scp` for most use cases — it's faster for repeated transfers, supports resume, and handles large directory trees better.

## sshd Hardening

Security best practices for the SSH server. Edit `/etc/ssh/sshd_config` or add drop-in files in `/etc/ssh/sshd_config.d/`.

### Recommended Settings

```conf
# Disable root login (force users to sudo)
PermitRootLogin no

# Disable password authentication (key-only — most important hardening step)
PasswordAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Change default port (reduces log noise from automated scanners)
Port 2222

# Allow only specific users or groups
AllowUsers jo admin
AllowGroups wheel ssh-users

# Limit authentication attempts per connection
MaxAuthTries 3

# Disable X11 forwarding (attack surface reduction)
X11Forwarding no

# Disable agent forwarding
AllowAgentForwarding no

# Set login grace time (seconds before unauthenticated connections are dropped)
LoginGraceTime 30

# Use only strong algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Verbose logging (useful for auditing)
LogLevel VERBOSE
```

### Applying Changes

```bash
# Test config syntax before restarting (catches errors)
sudo sshd -t

# Restart sshd to apply changes
sudo systemctl restart sshd

# Check sshd status
systemctl status sshd

# View auth logs live
journalctl -u sshd -f
```

## Permissions Reference

SSH refuses to work if file permissions are too permissive. These are the required permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519        # private key
chmod 644 ~/.ssh/id_ed25519.pub    # public key
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/known_hosts
```
