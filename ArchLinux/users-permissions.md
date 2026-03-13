# Users & Permissions

## User Management

```bash
# Add a new user
sudo useradd -m -G wheel -s /bin/bash newuser

# Add user with specific UID and home directory
sudo useradd -m -u 1500 -d /home/custom -s /bin/zsh newuser

# Create a system user (no home, no login)
sudo useradd -r -s /usr/bin/nologin serviceuser

# Set / change password
sudo passwd newuser

# Change your own password
passwd

# Modify user
sudo usermod -aG wheel,docker newuser    # add to groups (APPEND)
sudo usermod -s /bin/zsh newuser         # change shell
sudo usermod -l newname oldname          # rename user
sudo usermod -d /new/home -m newuser     # move home directory
sudo usermod -L newuser                  # lock account
sudo usermod -U newuser                  # unlock account

# Delete user
sudo userdel newuser
sudo userdel -r newuser    # also remove home directory and mail

# Show user info
id newuser
id                         # current user
whoami
who                        # who's logged in
w                          # detailed login info

# List all users
cat /etc/passwd
getent passwd

# Show groups for a user
groups newuser

# Change user's login shell
chsh -s /bin/zsh

# Set password expiry
sudo chage -l newuser                    # show expiry info
sudo chage -M 90 newuser                 # password expires after 90 days
sudo chage -E 2025-12-31 newuser         # account expires on date
sudo chage -d 0 newuser                  # force password change at next login
```

## Group Management

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

# Change primary group
sudo usermod -g developers newuser
```

> **Warning:** Always use `-aG` (append) with `usermod`. Using `-G` without `-a` **replaces** all supplementary groups.

## File Permissions (chmod)

### Symbolic Mode

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
chmod -R u=rwX,g=rX,o= /project

# Capital X: execute only for directories (and files already executable)
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +
```

### Numeric Mode

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
chmod 755 script.sh     # rwxr-xr-x
chmod 644 file.txt      # rw-r--r--
chmod 600 secret.txt    # rw-------
chmod 700 private_dir/  # rwx------
chmod 775 shared_dir/   # rwxrwxr-x
```

### Special Permissions

```bash
# SUID — run as file owner
chmod u+s /usr/bin/program
chmod 4755 /usr/bin/program

# SGID — run as file group / inherit group in directories
chmod g+s /shared/dir
chmod 2775 /shared/dir

# Sticky bit — only owner can delete in directory
chmod +t /tmp
chmod 1777 /tmp

# Find SUID/SGID files
find / -perm /4000 -type f 2>/dev/null    # SUID
find / -perm /2000 -type f 2>/dev/null    # SGID
```

## Ownership (chown)

```bash
# Change owner
sudo chown newuser file.txt

# Change owner and group
sudo chown newuser:developers file.txt

# Change group only
sudo chown :developers file.txt
chgrp developers file.txt

# Recursive
sudo chown -R newuser:developers /project

# Change only if current owner matches
sudo chown --from=olduser newuser file.txt

# Copy ownership from another file
sudo chown --reference=source.txt target.txt
```

## ACLs (Access Control Lists)

**Install:** `pacman -S acl` (usually included by default)

```bash
# View ACLs
getfacl file.txt
getfacl -R /project    # recursive

# Set ACL for a user
setfacl -m u:jo:rw file.txt

# Set ACL for a group
setfacl -m g:developers:rx /project

# Set default ACL (inherited by new files in directory)
setfacl -d -m g:developers:rwx /project

# Remove a specific ACL entry
setfacl -x u:jo file.txt

# Remove all ACLs
setfacl -b file.txt

# Recursive ACL
setfacl -R -m g:developers:rx /project

# Copy ACLs from one file to another
getfacl source.txt | setfacl --set-file=- target.txt

# Mask (effective permissions cap for named users/groups)
setfacl -m m::rx file.txt
```

**Note:** When ACLs are set, `ls -l` shows a `+` after the permission string: `-rw-rw-r--+`

## sudoers

```bash
# Edit sudoers (ALWAYS use visudo)
sudo visudo

# Edit a drop-in file (preferred)
sudo visudo -f /etc/sudoers.d/myuser
```

```conf
# /etc/sudoers.d/myuser

# Allow user full sudo with password
jo ALL=(ALL:ALL) ALL

# Allow user sudo without password
jo ALL=(ALL:ALL) NOPASSWD: ALL

# Allow specific commands only
jo ALL=(ALL) /usr/bin/pacman, /usr/bin/systemctl

# Allow a group
%wheel ALL=(ALL:ALL) ALL

# Allow NOPASSWD for specific commands
jo ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Run as a specific user
jo ALL=(postgres) /usr/bin/psql
```

```bash
# Test sudo config
sudo -l                # list allowed commands for current user
sudo -l -U otheruser   # list for another user

# Run as another user
sudo -u postgres psql

# Edit a file with sudoedit (safer than sudo vim)
sudoedit /etc/hosts
```

## PAM (Pluggable Authentication Modules)

Configuration directory: `/etc/pam.d/`

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

```conf
# Example: require wheel group for su
# /etc/pam.d/su
auth required pam_wheel.so use_uid
```

```bash
# Lock account after 5 failed attempts
# /etc/security/faillock.conf
# deny = 5
# unlock_time = 600

# Show failed login attempts
faillock --user jo

# Reset failed count
sudo faillock --user jo --reset

# Set password requirements
# /etc/security/pwquality.conf
# minlen = 12
# dcredit = -1      (at least 1 digit)
# ucredit = -1      (at least 1 uppercase)

# Set resource limits
# /etc/security/limits.conf
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
