# Environment & Shell Configuration

Shell startup file order for bash and zsh, environment variables, PATH management, XDG base directories, locale settings, and example shell configs.

## Shell Startup Order

Understanding which files are loaded when is essential for knowing where to put environment variables, aliases, and other configuration. The key distinction is **login** vs **interactive** — a login shell runs once (e.g., SSH login, TTY), while interactive shells run every time you open a terminal.

### Bash

```
Login shell (ssh, tty, bash --login):
  /etc/profile
    → /etc/profile.d/*.sh
  ~/.bash_profile  (or ~/.bash_login, or ~/.profile — first found wins)
    → usually sources ~/.bashrc

Interactive non-login shell (opening a terminal emulator):
  /etc/bash.bashrc
  ~/.bashrc

Non-interactive (running a script):
  $BASH_ENV (if set)
```

### Zsh

```
Always (every zsh invocation, including scripts):
  /etc/zshenv
  ~/.zshenv

Login shell:
  /etc/zprofile
  ~/.zprofile

Interactive shell:
  /etc/zshrc
  ~/.zshrc

Login shell (after zshrc):
  /etc/zlogin
  ~/.zlogin

On logout:
  ~/.zlogout
  /etc/zlogout
```

### Quick Reference

| File | Bash | Zsh | When |
|------|------|-----|------|
| `/etc/profile` | Login | — | System-wide login setup |
| `/etc/bash.bashrc` | Interactive | — | System-wide bash interactive |
| `/etc/zshenv` | — | Always | System-wide env for all zsh |
| `/etc/zshrc` | — | Interactive | System-wide zsh interactive |
| `~/.bash_profile` | Login | — | User login setup |
| `~/.bashrc` | Interactive | — | User bash interactive |
| `~/.zshenv` | — | Always | User env for all zsh |
| `~/.zprofile` | — | Login | User zsh login setup |
| `~/.zshrc` | — | Interactive | User zsh interactive |
| `~/.profile` | Login fallback | — | Generic login (used if no `.bash_profile`) |

> **Q:** Where should I put environment variables?
>
> For **bash**: `~/.bash_profile` (login) or `~/.bashrc` (if your `.bash_profile` sources it).
> For **zsh**: `~/.zshenv` (available everywhere) or `~/.zprofile` (login only).
> For **both shells**: `~/.profile` and ensure both source it. Or use `~/.config/environment.d/` for systemd user sessions.

## Environment Variables

Environment variables are key-value pairs inherited by child processes. They configure how programs behave — locale, editor, paths, and more.

### Viewing and Setting

```bash
# View all environment variables
env
printenv

# View a specific variable
echo $HOME
printenv PATH

# Set for current session and all child processes
export MY_VAR="value"

# Set for a single command only (doesn't persist)
MY_VAR="value" command

# Unset a variable
unset MY_VAR

# Persist (add to appropriate shell config file — see startup order)
echo 'export MY_VAR="value"' >> ~/.bashrc
```

### Common Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `HOME` | User home directory | `/home/jo` |
| `USER` | Current username | `jo` |
| `SHELL` | Default login shell | `/bin/zsh` |
| `PATH` | Executable search path (colon-separated) | `/usr/local/bin:/usr/bin:/bin` |
| `EDITOR` | Default text editor (for `crontab -e`, `git commit`, etc.) | `vim` |
| `VISUAL` | Default visual editor (preferred over `EDITOR` by some tools) | `vim` |
| `PAGER` | Default pager (for `man`, `git log`, etc.) | `less` |
| `TERM` | Terminal type (affects color support, key handling) | `xterm-256color` |
| `LANG` | System locale | `en_US.UTF-8` |
| `TZ` | Timezone | `Europe/Paris` |
| `DISPLAY` | X11 display server | `:0` |
| `WAYLAND_DISPLAY` | Wayland display | `wayland-0` |
| `XDG_SESSION_TYPE` | Session type | `wayland` or `x11` |
| `XDG_CURRENT_DESKTOP` | Desktop environment | `GNOME` |
| `DBUS_SESSION_BUS_ADDRESS` | D-Bus session address | `unix:path=/run/user/1000/bus` |

### System-Wide Environment

These locations set variables for all users.

```bash
# /etc/environment — simple KEY=VALUE pairs (no shell expansion, no export needed)
EDITOR=vim
PATH="/usr/local/bin:/usr/bin:/bin"

# /etc/profile.d/custom.sh — shell scripts (supports expansion, sourced at login)
export GOPATH="$HOME/go"
export PATH="$PATH:$GOPATH/bin"
```

### systemd User Environment

For graphical sessions and systemd user services, environment variables set in shell config may not be available. Use `environment.d` for those.

```ini
# ~/.config/environment.d/50-custom.conf
# Available to all systemd user services and graphical sessions
EDITOR=vim
PATH=${HOME}/.local/bin:${PATH}
```

```bash
# Show systemd user environment
systemctl --user show-environment

# Set a variable in the systemd user manager
systemctl --user set-environment MY_VAR=value

# Import current shell environment into systemd user session
systemctl --user import-environment
```

## PATH Management

`PATH` is a colon-separated list of directories the shell searches for executables. Order matters — directories listed first take precedence.

### Viewing and Modifying

```bash
# View current PATH (one directory per line)
echo $PATH | tr ':' '\n'

# Prepend (higher priority — checked first)
export PATH="$HOME/.local/bin:$PATH"

# Append (lower priority — checked last)
export PATH="$PATH:/opt/custom/bin"

# Add conditionally (avoid duplicates)
[[ ":$PATH:" != *":$HOME/.local/bin:"* ]] && export PATH="$HOME/.local/bin:$PATH"
```

### Common Additions

```bash
export PATH="$HOME/.local/bin:$PATH"       # user-local binaries
export PATH="$HOME/.cargo/bin:$PATH"       # Rust (cargo install)
export PATH="$HOME/go/bin:$PATH"           # Go binaries
export PATH="$HOME/.npm-global/bin:$PATH"  # npm global packages
```

### Checking PATH Resolution

```bash
# Show which binary PATH resolves to
which python
type python
command -v python

# Show ALL matches in PATH (useful for finding shadowed binaries)
which -a python
type -a python
```

## XDG Base Directory Specification

The XDG spec standardizes where applications store their files. Using these variables keeps your home directory clean and makes backup/migration easier.

```bash
# Set in your shell config (these are the defaults if not set)
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_STATE_HOME="$HOME/.local/state"
export XDG_CACHE_HOME="$HOME/.cache"
# XDG_RUNTIME_DIR is set by systemd (typically /run/user/$UID)

# Verify runtime dir
echo $XDG_RUNTIME_DIR
```

### Common XDG Paths

Applications that respect XDG store their config here instead of dotfiles in `$HOME`.

| Application | Config Path |
|-------------|------------|
| Git | `~/.config/git/config` |
| SSH | `~/.ssh/` (does **not** follow XDG) |
| npm | `~/.config/npm/npmrc` |
| Docker | `~/.config/docker/` |
| systemd user units | `~/.config/systemd/user/` |
| fontconfig | `~/.config/fontconfig/` |
| GTK-3 | `~/.config/gtk-3.0/` |
| environment.d | `~/.config/environment.d/` |

```bash
# Create standard XDG directories
mkdir -p ~/.config ~/.local/share ~/.local/state ~/.cache ~/.local/bin
```

## Bash Configuration Example

A minimal but complete bash setup split between login and interactive config.

### Login Shell (~/.bash_profile)

Runs once when you log in. Set environment variables here.

```bash
# ~/.bash_profile
export EDITOR=vim
export VISUAL=vim
export PAGER=less

# Add user paths
export PATH="$HOME/.local/bin:$PATH"

# Source interactive config (so it also applies to login shells)
[[ -f ~/.bashrc ]] && source ~/.bashrc
```

### Interactive Shell (~/.bashrc)

Runs every time you open a terminal. Put aliases, prompt, and completions here.

```bash
# ~/.bashrc

# If not running interactively, don't do anything
[[ $- != *i* ]] && return

# History settings
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth
shopt -s histappend

# Shell options
shopt -s globstar         # ** recursive glob
shopt -s checkwinsize     # update LINES/COLUMNS after each command
shopt -s cdspell          # autocorrect cd typos
shopt -s dirspell         # autocorrect directory name typos

# Aliases
alias ls='ls --color=auto'
alias ll='ls -lah'
alias grep='grep --color=auto'
alias ..='cd ..'

# Prompt (green user@host, blue working dir)
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '

# Completions
[[ -f /usr/share/bash-completion/bash_completion ]] && source /usr/share/bash-completion/bash_completion
```

## Zsh Configuration Example

A minimal zsh setup split between environment and interactive config.

### Environment (~/.zshenv)

Runs for **every** zsh invocation (interactive, login, scripts). Keep it minimal — only environment variables.

```bash
# ~/.zshenv
export EDITOR=vim
export VISUAL=vim
export PAGER=less
export PATH="$HOME/.local/bin:$PATH"
```

### Interactive Shell (~/.zshrc)

Runs for every interactive shell. Put everything else here.

```bash
# ~/.zshrc

# History
HISTSIZE=10000
SAVEHIST=20000
HISTFILE=~/.zsh_history
setopt HIST_IGNORE_DUPS      # don't store duplicate consecutive commands
setopt HIST_IGNORE_SPACE     # don't store commands starting with a space
setopt SHARE_HISTORY         # share history between sessions in real time
setopt APPEND_HISTORY        # append instead of overwrite

# Shell options
setopt AUTO_CD               # cd by typing directory name alone
setopt CORRECT               # suggest corrections for typos
setopt GLOB_DOTS             # include dotfiles in globbing
setopt EXTENDED_GLOB         # advanced globbing (#, ~, ^)

# Aliases
alias ls='ls --color=auto'
alias ll='ls -lah'
alias grep='grep --color=auto'
alias ..='cd ..'

# Keybindings
bindkey -e                   # emacs mode (default)
# bindkey -v                 # vi mode

# Completions
autoload -Uz compinit && compinit
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Z}'    # case-insensitive
zstyle ':completion:*' menu select                       # menu selection

# Prompt (green user@host, blue working dir)
autoload -Uz promptinit && promptinit
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f%# '
```

## Changing Default Shell

The default shell is what you get when you log in via TTY or SSH.

```bash
# List available shells
cat /etc/shells

# Change your shell
chsh -s /bin/zsh

# Change another user's shell (requires root)
sudo chsh -s /bin/zsh otheruser

# Install zsh (if not already installed)
sudo pacman -S zsh zsh-completions

# Verify (shows login shell — you may need to log out and back in)
echo $SHELL
```

## Locale & Language

Locale settings control language, number formatting, date format, sorting, and character encoding.

```bash
# Show current locale settings
locale

# List all available locales
locale -a

# Generate locales:
# 1. Uncomment desired locales in /etc/locale.gen
# 2. Generate
sudo locale-gen

# Set system locale (persistent)
sudo localectl set-locale LANG=en_US.UTF-8

# Set console keymap
sudo localectl set-keymap us

# Show locale settings
localectl status
```

### Per-Session Overrides

Override specific categories without changing the system default.

```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8

# Override specific categories
export LC_TIME=en_GB.UTF-8      # 24-hour time, DD/MM/YYYY
export LC_NUMERIC=en_US.UTF-8   # US number formatting (1,000.00)
export LC_COLLATE=C              # sort by byte value (faster, predictable)
```
