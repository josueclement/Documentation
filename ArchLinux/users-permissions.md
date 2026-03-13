# Users & Permissions

User and group management, file permissions (`chmod`, `chown`), ACLs, `sudoers` configuration, and PAM authentication modules.

## User Management

Linux user accounts are defined in `/etc/passwd` (metadata) and `/etc/shadow` (password hashes). `useradd` creates accounts, `usermod` modifies them.

### Creating Users

The `-m` flag creates a home directory. `-G` adds supplementary groups. `-s` sets the login shell.

```bash
# Add a new user with home directory, wheel group, bash shell
sudo useradd -m -G wheel -s /bin/bash newuser

# Add user with specific UID and custom home directory
sudo useradd -m -u 1500 -d /home/custom -s /bin/zsh newuser

# Create a system user (no home, no login shell — for running services)
sudo useradd -r -s /usr/bin/nologin serviceuser

# Set / change password
sudo passwd newuser

# Change your own password
passwd
```

### Modifying Users

```bash
# Add to additional groups (APPEND — always use -a with -G)
sudo usermod -aG wheel,docker newuser

# Change shell
sudo usermod -s /bin/zsh newuser

# Rename user
sudo usermod -l newname oldname

# Move home directory (creates new dir and moves contents)
sudo usermod -d /new/home -m newuser

# Lock account (disable login)
sudo usermod -L newuser

# Unlock account
sudo usermod -U newuser
```

### Deleting Users

```bash
# Delete user (keeps home directory)
sudo userdel newuser

# Delete user and remove home directory and mail spool
sudo userdel -r newuser
```

### Inspecting Users

```bash
# Show user info (UID, GID, groups)
id newuser
id                         # current user

# Who am I
whoami

# Who's logged in
who
w                          # detailed login info (idle time, process)

# List all users
cat /etc/passwd
getent passwd

# Show groups for a user
groups newuser

# Change your own login shell
chsh -s /bin/zsh
```

### Password Expiry

`chage` manages password aging and account expiration policies.

```bash
# Show expiry info for a user
sudo chage -l newuser

# Password expires after 90 days
sudo chage -M 90 newuser

# Account expires on a specific date
sudo chage -E 2025-12-31 newuser

# Force password change at next login
sudo chage -d 0 newuser
```

## Group Management

Groups organize users for shared file access and privilege management.

```bash
# Create a group
sudo groupadd developers

# Create with specific GID
sudo groupadd -g 2000 developers

# Add user to a group
sudo usermod -aG developers newuser

# Remove user from a group
sudo gpasswd -d newuser developers

# Delete a group
sudo groupdel developers

# List all groups
cat /etc/group
getent group

# Show group members
getent group wheel

# Change a user's primary group
sudo usermod -g developers newuser
```

> **Warning:** Always use `-aG` (append) with `usermod`. Using `-G` without `-a` **replaces** all supplementary groups, potentially locking the user out of sudo.

## File Permissions (chmod)

Every file has three permission sets: **owner** (u), **group** (g), **others** (o). Each set can have **read** (r=4), **write** (w=2), and **execute** (x=1).

### Symbolic Mode

Symbolic mode uses letters and operators: `+` adds, `-` removes, `=` sets exactly.

```bash
# Add execute for owner
chmod u+x script.sh

# Remove write for group and others
chmod go-w file.txt

# Set read+write for owner, read for group, none for others
chmod u=rw,g=r,o= file.txt

# Add read for everyone
chmod a+r file.txt

# Set directory permissions recursively
# Capital X = execute only for directories (and files already executable)
chmod -R u=rwX,g=rX,o= /project

# Separate treatment for files and directories
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +
```

### Numeric (Octal) Mode

Each digit is the sum of read (4) + write (2) + execute (1) for owner, group, others respectively.

| # | Permission |
|---|-----------|
| 0 | `---` none |
| 1 | `--x` execute |
| 2 | `-w-` write |
| 3 | `-wx` write + execute |
| 4 | `r--` read |
| 5 | `r-x` read + execute |
| 6 | `rw-` read + write |
| 7 | `rwx` read + write + execute |

```bash
chmod 755 script.sh     # rwxr-xr-x  (owner full, others read+exec)
chmod 644 file.txt      # rw-r--r--  (owner read+write, others read)
chmod 600 secret.txt    # rw-------  (owner only)
chmod 700 private_dir/  # rwx------  (owner only)
chmod 775 shared_dir/   # rwxrwxr-x  (owner+group full, others read+exec)
```

### Special Permissions

These are the leading digit in a 4-digit octal mode. They change how executables run or how directories handle file ownership.

```bash
# SUID — executable runs as the file owner (not the caller)
chmod u+s /usr/bin/program
chmod 4755 /usr/bin/program

# SGID — executable runs as the file group; on directories, new files inherit the group
chmod g+s /shared/dir
chmod 2775 /shared/dir

# Sticky bit — on directories, only the file owner can delete their files (used on /tmp)
chmod +t /tmp
chmod 1777 /tmp

# Find SUID/SGID files on the system (security auditing)
find / -perm /4000 -type f 2>/dev/null    # SUID
find / -perm /2000 -type f 2>/dev/null    # SGID
```

## Ownership (chown)

`chown` changes file owner and group. Most operations require root.

```bash
# Change owner
sudo chown newuser file.txt

# Change owner and group
sudo chown newuser:developers file.txt

# Change group only
sudo chown :developers file.txt
chgrp developers file.txt

# Recursive (change everything in a directory tree)
sudo chown -R newuser:developers /project

# Change only if current owner matches
sudo chown --from=olduser newuser file.txt

# Copy ownership from another file
sudo chown --reference=source.txt target.txt
```

## ACLs (Access Control Lists)

ACLs extend traditional Unix permissions by allowing per-user and per-group rules on individual files. Useful when standard owner/group/others isn't granular enough.

**Install:** `pacman -S acl` (usually included by default)

When ACLs are set, `ls -l` shows a `+` after the permission string: `-rw-rw-r--+`

### Viewing and Setting ACLs

```bash
# View ACLs on a file
getfacl file.txt

# View ACLs recursively
getfacl -R /project

# Grant a specific user read+write access
setfacl -m u:jo:rw file.txt

# Grant a specific group read+execute access
setfacl -m g:developers:rx /project

# Set default ACL (new files in directory inherit this)
setfacl -d -m g:developers:rwx /project

# Recursive ACL
setfacl -R -m g:developers:rx /project
```

### Removing ACLs

```bash
# Remove a specific ACL entry
setfacl -x u:jo file.txt

# Remove all ACLs (revert to standard permissions)
setfacl -b file.txt
```

### Copying ACLs

```bash
# Copy ACLs from one file to another
getfacl source.txt | setfacl --set-file=- target.txt
```

### Mask

The mask is the effective permissions cap for named users and groups (not the owner). It's automatically recalculated when ACLs change, but can be set manually.

```bash
setfacl -m m::rx file.txt
```

## sudoers

`sudoers` controls who can run commands as root (or other users) and whether a password is required. **Always** edit via `visudo` — it validates syntax before saving, preventing lockouts.

### Editing

```bash
# Edit the main sudoers file
sudo visudo

# Edit a drop-in file (preferred — survives package upgrades)
sudo visudo -f /etc/sudoers.d/myuser
```

### Common Rules

```conf
# /etc/sudoers.d/myuser

# Allow user full sudo with password
jo ALL=(ALL:ALL) ALL

# Allow user sudo without password
jo ALL=(ALL:ALL) NOPASSWD: ALL

# Allow specific commands only
jo ALL=(ALL) /usr/bin/pacman, /usr/bin/systemctl

# Allow a group (% prefix for groups)
%wheel ALL=(ALL:ALL) ALL

# Allow NOPASSWD for specific commands only
jo ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Run commands as a specific user
jo ALL=(postgres) /usr/bin/psql
```

### Testing and Using sudo

```bash
# List allowed commands for current user
sudo -l

# List for another user
sudo -l -U otheruser

# Run as another user
sudo -u postgres psql

# Edit a file with sudoedit (safer than sudo vim — drops privileges after opening)
sudoedit /etc/hosts
```

## PAM (Pluggable Authentication Modules)

PAM is the framework that handles authentication for login, sudo, SSH, and other services. Each service has a config file in `/etc/pam.d/` that chains authentication modules together.

### Key Configuration Files

```bash
# List PAM config files
ls /etc/pam.d/

# Key files:
# /etc/pam.d/system-auth    — default authentication
# /etc/pam.d/system-login   — login authentication
# /etc/pam.d/su             — su command
# /etc/pam.d/sudo           — sudo command
# /etc/pam.d/sshd           — SSH authentication
```

### Common PAM Configurations

```conf
# Require wheel group for su (prevent non-wheel users from su'ing to root)
# /etc/pam.d/su
auth required pam_wheel.so use_uid
```

### Account Lockout

`faillock` locks accounts after too many failed authentication attempts.

```bash
# Configure in /etc/security/faillock.conf:
# deny = 5              (lock after 5 failures)
# unlock_time = 600     (unlock after 10 minutes)

# Show failed login attempts
faillock --user jo

# Reset failed count manually
sudo faillock --user jo --reset
```

### Password Quality and Resource Limits

```bash
# Set password requirements in /etc/security/pwquality.conf:
# minlen = 12        (minimum length)
# dcredit = -1       (at least 1 digit)
# ucredit = -1       (at least 1 uppercase)

# Set resource limits in /etc/security/limits.conf:
# jo  hard  nofile  65535
# @developers  soft  nproc  4096
```

## Key Files

| File | Purpose |
|------|---------|
| `/etc/passwd` | User accounts (name, UID, GID, home, shell) |
| `/etc/shadow` | Encrypted passwords and expiry info |
| `/etc/group` | Group definitions |
| `/etc/gshadow` | Encrypted group passwords |
| `/etc/sudoers` | Sudo rules (edit with `visudo`) |
| `/etc/sudoers.d/` | Drop-in sudo rules |
| `/etc/login.defs` | Default login settings (UID ranges, password aging) |
| `/etc/skel/` | Skeleton files copied to new user home directories |
| `/etc/pam.d/` | PAM configuration |
| `/etc/security/` | Security configs (limits, faillock, pwquality) |
