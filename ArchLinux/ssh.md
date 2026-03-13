# SSH

## Key Management

```bash
# Generate a new key pair (Ed25519 — recommended)
ssh-keygen -t ed25519 -C "jo@hostname"

# Generate RSA key (if Ed25519 not supported)
ssh-keygen -t rsa -b 4096 -C "jo@hostname"

# Generate key with custom path
ssh-keygen -t ed25519 -f ~/.ssh/mykey

# Change passphrase on an existing key
ssh-keygen -p -f ~/.ssh/id_ed25519

# Show key fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# Show key in different formats
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E md5
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E sha256

# Copy public key to a remote server
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/mykey.pub user@host

# Manual copy (if ssh-copy-id unavailable)
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# List authorized keys on remote
ssh user@host "cat ~/.ssh/authorized_keys"
```

## SSH Config

File: `~/.ssh/config`

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

# Jump through a bastion host
Host internal
    HostName 10.0.0.5
    User jo
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jo
    IdentityFile ~/.ssh/bastion_key

# Wildcard matching
Host *.dev.example.com
    User deployer
    IdentityFile ~/.ssh/dev_key

# Keep connection alive for multiplexing
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

```bash
# Create the sockets directory
mkdir -p ~/.ssh/sockets

# Test SSH config
ssh -G myserver    # show resolved config for a host

# Connect using config alias
ssh myserver
```

## SSH Tunnels

```bash
# Local port forwarding — access remote service locally
# Local:8080 → remote:80
ssh -L 8080:localhost:80 user@remote

# Access a service behind the remote (through the tunnel)
# Local:5432 → db.internal:5432 via remote
ssh -L 5432:db.internal:5432 user@remote

# Remote port forwarding — expose local service on remote
# Remote:9090 → local:3000
ssh -R 9090:localhost:3000 user@remote

# Dynamic port forwarding (SOCKS proxy)
ssh -D 1080 user@remote
# Then configure browser to use SOCKS5 proxy localhost:1080

# Tunnel in background (no shell)
ssh -fNL 8080:localhost:80 user@remote

# Tunnel flags:
# -f  background after authentication
# -N  no remote command (tunnel only)
# -L  local forward
# -R  remote forward
# -D  dynamic (SOCKS)

# Kill a background tunnel
pkill -f "ssh -fN.*8080"
```

## SSH Agent

```bash
# Start agent (usually auto-started by your DE/WM)
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519

# Add with timeout (seconds)
ssh-add -t 3600 ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove a key
ssh-add -d ~/.ssh/id_ed25519

# Remove all keys
ssh-add -D

# Agent forwarding (use key on remote for further hops)
ssh -A user@bastion
# Then from bastion: ssh user@internal (uses your local key)
```

> **Warning:** Agent forwarding (`-A`) exposes your keys to the remote host's admin. Prefer `ProxyJump` instead.

## Common Operations

```bash
# Basic connection
ssh user@host
ssh -p 2222 user@host

# Run a remote command
ssh user@host "uname -a"
ssh user@host "cat /etc/os-release"

# Run multiple commands
ssh user@host "cd /var/log && tail -20 syslog"

# Run with pseudo-terminal (for interactive commands)
ssh -t user@host "sudo systemctl restart nginx"

# Pipe local input to remote
cat backup.sql | ssh user@host "psql -d mydb"
tar czf - /data | ssh user@host "cat > /backups/data.tar.gz"

# Pipe remote output locally
ssh user@host "tar czf - /data" > data.tar.gz

# Verbose output (debugging)
ssh -v user@host
ssh -vv user@host     # more verbose
ssh -vvv user@host    # maximum verbosity

# Escape sequences (while in SSH session)
# ~.    disconnect
# ~^Z   suspend SSH
# ~#    list forwarded connections
# ~?    help
```

## rsync over SSH

```bash
# Copy to remote
rsync -avz /local/dir/ user@host:/remote/dir/

# Copy from remote
rsync -avz user@host:/remote/dir/ /local/dir/

# With custom SSH port
rsync -avz -e "ssh -p 2222" /local/dir/ user@host:/remote/dir/

# Dry run (preview changes)
rsync -avzn /local/dir/ user@host:/remote/dir/

# Delete remote files not present locally
rsync -avz --delete /local/dir/ user@host:/remote/dir/

# Exclude patterns
rsync -avz --exclude='.git' --exclude='node_modules' src/ user@host:/deploy/

# Include/exclude rules from file
rsync -avz --filter='merge rsync-filter.txt' src/ user@host:/deploy/

# Show progress
rsync -avz --progress /local/dir/ user@host:/remote/dir/

# Limit bandwidth (KB/s)
rsync -avz --bwlimit=1000 /local/dir/ user@host:/remote/dir/

# Resume interrupted transfer
rsync -avz --partial --progress large-file user@host:/dest/

# Common flags:
# -a  archive (recursive, preserves permissions, times, etc.)
# -v  verbose
# -z  compress during transfer
# -P  --partial --progress
# -n  dry run
# -h  human-readable sizes
```

## scp (Simple Copy)

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

> **Note:** Prefer `rsync` over `scp` for most use cases — it's faster for repeated transfers and supports resume.

## sshd Hardening

File: `/etc/ssh/sshd_config` (or drop-in: `/etc/ssh/sshd_config.d/*.conf`)

```conf
# Disable root login
PermitRootLogin no

# Disable password authentication (key-only)
PasswordAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Change default port
Port 2222

# Allow only specific users
AllowUsers jo admin

# Allow only specific groups
AllowGroups wheel ssh-users

# Limit authentication attempts
MaxAuthTries 3

# Disable X11 forwarding
X11Forwarding no

# Disable agent forwarding
AllowAgentForwarding no

# Set login grace time
LoginGraceTime 30

# Use only strong algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Log level
LogLevel VERBOSE
```

```bash
# Test config before restarting
sudo sshd -t

# Restart sshd
sudo systemctl restart sshd

# Check sshd status
systemctl status sshd

# View auth logs
journalctl -u sshd -f
```

## Permissions Reference

```bash
# Required permissions (too permissive = SSH refuses to work)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519        # private key
chmod 644 ~/.ssh/id_ed25519.pub    # public key
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/known_hosts
```
