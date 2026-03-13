# Environment & Shell Configuration

## Shell Startup Order

### Bash

```
Login shell (ssh, tty, bash --login):
  /etc/profile
    → /etc/profile.d/*.sh
  ~/.bash_profile  (or ~/.bash_login, or ~/.profile — first found)
    → usually sources ~/.bashrc

Interactive non-login shell (terminal emulator):
  /etc/bash.bashrc
  ~/.bashrc

Non-interactive (scripts):
  $BASH_ENV (if set)
```

### Zsh

```
Always:
  /etc/zshenv
  ~/.zshenv

Login shell:
  /etc/zprofile
  ~/.zprofile

Interactive:
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
| `~/.profile` | Login fallback | — | Generic login (if no .bash_profile) |

> **Q:** Where should I put environment variables?
>
> For **bash**: `~/.bash_profile` (login) or `~/.bashrc` (if your `.bash_profile` sources it).
> For **zsh**: `~/.zshenv` (available everywhere) or `~/.zprofile` (login only).
> For **both shells**: `~/.profile` and ensure both source it. Or use `~/.config/environment.d/` for systemd user sessions.

## Environment Variables

```bash
# View all environment variables
env
printenv

# View a specific variable
echo $HOME
printenv PATH

# Set for current session
export MY_VAR="value"

# Set for a single command
MY_VAR="value" command

# Unset a variable
unset MY_VAR

# Persist variables (add to appropriate file, see startup order)
echo 'export MY_VAR="value"' >> ~/.bashrc
```

### Common Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `HOME` | User home directory | `/home/jo` |
| `USER` | Current username | `jo` |
| `SHELL` | Default login shell | `/bin/zsh` |
| `PATH` | Executable search path | `/usr/local/bin:/usr/bin:/bin` |
| `EDITOR` | Default text editor | `vim` |
| `VISUAL` | Default visual editor | `vim` |
| `PAGER` | Default pager | `less` |
| `TERM` | Terminal type | `xterm-256color` |
| `LANG` | System locale | `en_US.UTF-8` |
| `TZ` | Timezone | `Europe/Paris` |
| `DISPLAY` | X11 display | `:0` |
| `WAYLAND_DISPLAY` | Wayland display | `wayland-0` |
| `XDG_SESSION_TYPE` | Session type | `wayland` or `x11` |
| `XDG_CURRENT_DESKTOP` | Desktop environment | `GNOME` |
| `DBUS_SESSION_BUS_ADDRESS` | D-Bus session address | `unix:path=/run/user/1000/bus` |

### System-Wide Environment

```bash
# /etc/environment — simple KEY=VALUE pairs (no shell expansion)
EDITOR=vim
PATH="/usr/local/bin:/usr/bin:/bin"

# /etc/profile.d/custom.sh — shell scripts (supports expansion)
export GOPATH="$HOME/go"
export PATH="$PATH:$GOPATH/bin"
```

### systemd User Environment

```ini
# ~/.config/environment.d/50-custom.conf
# Available to all systemd user services and graphical sessions
EDITOR=vim
PATH=${HOME}/.local/bin:${PATH}
```

```bash
# Show systemd user environment
systemctl --user show-environment

# Set systemd user variable
systemctl --user set-environment MY_VAR=value

# Import all environment into systemd user session
systemctl --user import-environment
```

## PATH Management

```bash
# View current PATH
echo $PATH
echo $PATH | tr ':' '\n'    # one per line

# Add to PATH (in ~/.bashrc or ~/.zshrc)
export PATH="$HOME/.local/bin:$PATH"     # prepend (higher priority)
export PATH="$PATH:/opt/custom/bin"      # append (lower priority)

# Add conditionally (avoid duplicates)
[[ ":$PATH:" != *":$HOME/.local/bin:"* ]] && export PATH="$HOME/.local/bin:$PATH"

# Common user paths to add
export PATH="$HOME/.local/bin:$PATH"
export PATH="$HOME/.cargo/bin:$PATH"       # Rust
export PATH="$HOME/go/bin:$PATH"           # Go
export PATH="$HOME/.npm-global/bin:$PATH"  # npm global

# Check which binary PATH resolves to
which python
type python
command -v python

# Show all matches in PATH
which -a python
type -a python
```

## XDG Base Directory Specification

```bash
# Set in your shell config (these are the defaults)
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_STATE_HOME="$HOME/.local/state"
export XDG_CACHE_HOME="$HOME/.cache"
# XDG_RUNTIME_DIR is set by systemd (typically /run/user/$UID)

# Verify
echo $XDG_RUNTIME_DIR
```

### Common XDG Paths

| Application | Config Path |
|-------------|------------|
| Git | `~/.config/git/config` |
| SSH | `~/.ssh/` (does not follow XDG) |
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

```bash
# ~/.bash_profile
# Login shell — set environment, then source .bashrc

export EDITOR=vim
export VISUAL=vim
export PAGER=less

# Add user paths
export PATH="$HOME/.local/bin:$PATH"

# Source interactive config
[[ -f ~/.bashrc ]] && source ~/.bashrc
```

```bash
# ~/.bashrc
# Interactive shell — aliases, prompt, completions

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

# Prompt
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '

# Completions
[[ -f /usr/share/bash-completion/bash_completion ]] && source /usr/share/bash-completion/bash_completion
```

## Zsh Configuration Example

```bash
# ~/.zshenv
# All zsh invocations — environment only
export EDITOR=vim
export VISUAL=vim
export PAGER=less
export PATH="$HOME/.local/bin:$PATH"
```

```bash
# ~/.zshrc
# Interactive shell — everything else

# History
HISTSIZE=10000
SAVEHIST=20000
HISTFILE=~/.zsh_history
setopt HIST_IGNORE_DUPS
setopt HIST_IGNORE_SPACE
setopt SHARE_HISTORY
setopt APPEND_HISTORY

# Shell options
setopt AUTO_CD               # cd by typing directory name
setopt CORRECT               # suggest corrections
setopt GLOB_DOTS             # include dotfiles in globbing
setopt EXTENDED_GLOB         # advanced globbing

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

# Prompt (simple example)
autoload -Uz promptinit && promptinit
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f%# '
```

## Changing Default Shell

```bash
# List available shells
cat /etc/shells

# Change your shell
chsh -s /bin/zsh

# Change another user's shell
sudo chsh -s /bin/zsh otheruser

# Install zsh
sudo pacman -S zsh zsh-completions

# Verify
echo $SHELL
```

## Locale & Language

```bash
# Show current locale
locale

# List available locales
locale -a

# Generate locales
# 1. Uncomment desired locales in /etc/locale.gen
# 2. Generate
sudo locale-gen

# Set system locale
sudo localectl set-locale LANG=en_US.UTF-8

# Set per-session
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8

# Override specific categories
export LC_TIME=en_GB.UTF-8      # 24-hour time, DD/MM/YYYY
export LC_NUMERIC=en_US.UTF-8   # US number formatting
export LC_COLLATE=C              # sort by byte value (faster)
```
