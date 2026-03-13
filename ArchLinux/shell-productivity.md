# Shell Productivity

Keyboard shortcuts, history tricks, aliases, functions, globbing patterns, parameter expansion, brace expansion, and job control for bash and zsh.

## Keyboard Shortcuts (Bash/Zsh — Emacs Mode)

These shortcuts work in the default Emacs keybinding mode for both bash and zsh. They use readline, so they also work in many other CLI tools.

### Movement

Move the cursor without reaching for arrow keys.

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Alt+F` | Move forward one word |
| `Alt+B` | Move backward one word |
| `Ctrl+F` | Move forward one character |
| `Ctrl+B` | Move backward one character |
| `Ctrl+XX` | Toggle between current position and beginning |

### Editing

Cut, paste, delete, and transform text on the command line. "Cut" text goes into a kill ring that you can paste back with `Ctrl+Y`.

| Shortcut | Action |
|----------|--------|
| `Ctrl+W` | Cut word before cursor |
| `Alt+D` | Cut word after cursor |
| `Ctrl+K` | Cut from cursor to end of line |
| `Ctrl+U` | Cut from cursor to beginning of line |
| `Ctrl+Y` | Paste (yank) last cut text |
| `Alt+Y` | Cycle through previous cuts (after Ctrl+Y) |
| `Ctrl+D` | Delete character under cursor (or exit shell if empty) |
| `Ctrl+H` | Delete character before cursor (backspace) |
| `Alt+T` | Swap current word with previous |
| `Ctrl+T` | Swap current character with previous |
| `Alt+U` | Uppercase from cursor to end of word |
| `Alt+L` | Lowercase from cursor to end of word |
| `Alt+C` | Capitalize word |
| `Ctrl+_` | Undo |

### History & Control

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Reverse search history (incremental) |
| `Ctrl+S` | Forward search history |
| `Ctrl+G` | Cancel search |
| `Ctrl+P` / `Up` | Previous command |
| `Ctrl+N` / `Down` | Next command |
| `Alt+.` | Insert last argument of previous command |
| `Alt+#` | Comment out current line and execute (saves to history) |
| `Ctrl+L` | Clear screen |
| `Ctrl+C` | Cancel current command |
| `Ctrl+Z` | Suspend current process |
| `Ctrl+D` | Exit shell (if line is empty) |

## History

Bash and zsh maintain a command history that you can search, recall, and manipulate. Effective history usage eliminates repetitive typing.

### Viewing and Searching

```bash
# Show command history
history
history 20              # last 20 commands

# Search history
history | grep "docker"
Ctrl+R                  # interactive reverse search (type to narrow)
```

### Re-executing Previous Commands

History expansion (`!`) lets you quickly recall and modify past commands.

```bash
# Repeat last command
!!

# Run command number 42
!42

# Run last command starting with "docker"
!docker

# Run last command containing "keyword"
!?keyword
```

### Reusing Arguments

You can grab specific arguments from the previous command without retyping them.

```bash
# Last argument of previous command
echo !$

# First argument
echo !^

# All arguments
echo !*

# Specific argument by position (0 = command, 1 = first arg)
echo !:2

# Range of arguments
echo !:2-4
```

### Modifying and Rerunning

```bash
# Quick substitution in last command (replace first occurrence)
^old^new

# Replace all occurrences in last command
!!:gs/old/new
```

### History Configuration

Add these to `~/.bashrc` to customize history behavior.

```bash
export HISTSIZE=10000            # commands kept in memory
export HISTFILESIZE=20000        # commands kept in history file
export HISTCONTROL=ignoreboth    # ignore duplicates and space-prefixed commands
export HISTIGNORE="ls:cd:exit"   # don't record these commands
export HISTTIMEFORMAT="%F %T "   # add timestamps to history

# Append (don't overwrite) history on shell exit
shopt -s histappend

# Save history after every command (not just on exit)
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
```

### Managing History

```bash
# Clear session history
history -c

# Write current history to file
history -w
```

## Aliases

Aliases replace a command name with a string. They're defined in `~/.bashrc` or `~/.zshrc` and are great for shortening frequently used commands.

### Defining Aliases

```bash
# Navigation and listing
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'

# Add color by default
alias grep='grep --color=auto'
alias diff='diff --color=auto'

# Human-readable defaults
alias df='df -h'
alias du='du -h'
alias free='free -h'

# Create parent directories automatically
alias mkdir='mkdir -pv'

# Safety aliases (prompt before overwriting)
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Git aliases
alias gs='git status'
alias gd='git diff'
alias gl='git log --oneline -20'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
```

### Managing Aliases

```bash
# Show all defined aliases
alias

# Remove an alias (current session only)
unalias ll

# Bypass alias (run the original command)
\rm file.txt
command rm file.txt
```

## Functions

Shell functions are more powerful than aliases — they accept arguments, support control flow, and can contain multiple commands. Define them in `~/.bashrc` or `~/.zshrc`.

```bash
# Create directory and cd into it
mkcd() { mkdir -p "$1" && cd "$1"; }

# Extract any archive format automatically
extract() {
    case "$1" in
        *.tar.gz|*.tgz)  tar xzf "$1" ;;
        *.tar.bz2|*.tbz) tar xjf "$1" ;;
        *.tar.xz|*.txz)  tar xJf "$1" ;;
        *.tar.zst)        tar --zstd -xf "$1" ;;
        *.tar)            tar xf "$1" ;;
        *.gz)             gunzip "$1" ;;
        *.bz2)            bunzip2 "$1" ;;
        *.xz)             unxz "$1" ;;
        *.zip)            unzip "$1" ;;
        *.7z)             7z x "$1" ;;
        *.rar)            unrar x "$1" ;;
        *)                echo "Unknown format: $1" ;;
    esac
}

# Find and cd to a directory
fcd() { cd "$(fd -t d "$1" | head -1)"; }

# Quick timestamped backup of a file
bak() { cp "$1" "$1.bak.$(date +%F_%H%M%S)"; }
```

## Globbing (Pattern Matching)

Globbing is the shell's built-in pattern matching for filenames. The shell expands glob patterns before passing them to commands.

### Basic Wildcards

These work in all POSIX shells.

```bash
ls *.txt              # matches any .txt file
ls file?.txt          # matches file1.txt, fileA.txt (? = one character)
ls file[123].txt      # matches file1.txt, file2.txt, file3.txt
ls file[a-z].txt      # matches filea.txt through filez.txt
ls file[!0-9].txt     # matches files NOT ending in digit
```

### Extended Globbing (Bash)

Extended globs add negation, alternatives, and repetition. Must be enabled first.

```bash
# Enable extended globbing
shopt -s extglob

ls !(*.log)           # everything except .log files
ls +(*.jpg|*.png)     # one or more of .jpg or .png
ls ?(a|b).txt         # optional match: .txt, a.txt, or b.txt
ls @(*.jpg|*.png)     # exactly one of the patterns
ls *(pattern)         # zero or more of the pattern
```

### Recursive Globbing

`**` matches any depth of subdirectories. Must be enabled in bash; enabled by default in zsh.

```bash
# Enable in bash
shopt -s globstar

# All .py files in all subdirectories
ls **/*.py
```

### Zsh-Specific Globbing

Zsh has more powerful globbing with qualifier suffixes in parentheses.

```bash
ls **/*.py            # recursive (enabled by default)
ls *.txt(.)           # regular files only
ls *(/)               # directories only
ls *(@)               # symlinks only
ls *(m-7)             # modified in last 7 days
ls *(Lk+100)          # larger than 100KB
ls *(om[1,5])         # 5 most recently modified
```

### Null Glob

By default, if a glob matches nothing, it's passed literally (bash) or causes an error (zsh). Null glob makes it expand to nothing instead.

```bash
# bash
shopt -s nullglob

# zsh
setopt nullglob
```

## Parameter Expansion

Parameter expansion is bash/zsh's built-in string manipulation. It's faster than calling external tools like `sed` or `cut` for simple operations.

### Defaults and Assignment

```bash
# Use default value if variable is unset or empty
echo ${var:-default}

# Set variable to default if unset or empty
echo ${var:=default}
```

### Length and Substrings

```bash
name="file.tar.gz"

# String length
echo ${#name}                  # 11

# Substring extraction (offset, length)
echo ${name:0:4}               # "file"
echo ${name:5}                 # "tar.gz"
echo ${name: -2}               # "gz" (space before - is required)
```

### Prefix and Suffix Removal

`#` removes from the beginning, `%` removes from the end. Single `#`/`%` is shortest match, double `##`/`%%` is longest (greedy) match.

```bash
path="/home/jo/documents/file.txt"
name="file.tar.gz"

# Remove shortest prefix match
echo ${name#*.}                # "tar.gz"
echo ${path#*/}                # "home/jo/documents/file.txt"

# Remove longest prefix match (greedy)
echo ${name##*.}               # "gz"
echo ${path##*/}               # "file.txt" (like basename)

# Remove shortest suffix match
echo ${name%.*}                # "file.tar"
echo ${path%/*}                # "/home/jo/documents" (like dirname)

# Remove longest suffix match (greedy)
echo ${name%%.*}               # "file"
```

### Replacement

```bash
name="file.tar.gz"

# Replace first occurrence
echo ${name/tar/zip}           # "file.zip.gz"

# Replace all occurrences
echo ${name//./,}              # "file,tar,gz"

# Replace prefix
echo ${name/#file/doc}         # "doc.tar.gz"

# Replace suffix
echo ${name/%.gz/.xz}          # "file.tar.xz"
```

### Case Conversion (Bash 4+)

```bash
name="file.tar.gz"

echo ${name^^}                 # "FILE.TAR.GZ" (all uppercase)
echo ${name,,}                 # "file.tar.gz" (all lowercase)
echo ${name^}                  # "File.tar.gz" (capitalize first)
```

### Indirect Reference and Arrays

```bash
# Indirect reference (variable name stored in another variable)
ref="name"
echo ${!ref}                   # value of $name

# Array operations
arr=(one two three)
echo ${arr[@]}                 # all elements
echo ${#arr[@]}                # array length
echo ${arr[@]:1:2}             # slice: "two three"
```

## Brace Expansion

Brace expansion generates strings from a pattern. Unlike globbing, it doesn't match against existing files — it produces all combinations.

### Sequences

```bash
echo {1..10}                   # 1 2 3 4 5 6 7 8 9 10
echo {a..z}                    # a b c ... z
echo {01..12}                  # 01 02 03 ... 12 (zero-padded)
echo {1..20..2}                # 1 3 5 7 ... 19 (step of 2)
```

### Combinations

Braces expand as a cartesian product.

```bash
echo {a,b}{1,2}                # a1 a2 b1 b2
```

### Practical Uses

```bash
# Create nested directory structure in one command
mkdir -p project/{src,test,docs}/{main,utils}

# Quick file backup (copies file.txt to file.txt.bak)
cp file.txt{,.bak}

# Rename file (file.old → file.new)
mv file.{old,new}

# Create numbered files
touch file{1..10}.txt
```

## Job Control

Job control lets you manage multiple processes from a single shell — suspend, resume, background, and detach them. See `process-management.md` for broader process control.

```bash
# Run in background
sleep 100 &

# List jobs
jobs
jobs -l                        # with PIDs

# Bring to foreground
fg %1

# Send to background
bg %1

# Suspend foreground job
Ctrl+Z

# Disown (detach from shell — survives logout)
disown %1
disown -a                      # all jobs

# Wait for background jobs
wait                           # all
wait %1                        # specific job
wait $PID                      # specific PID
```

## Miscellaneous Tricks

Useful shell patterns that don't fit neatly elsewhere.

### Sudo and Directory Tricks

```bash
# Run previous command with sudo
sudo !!

# Toggle between last two directories
cd -

# Directory stack (push, pop, list)
pushd /new/dir
popd
dirs -v
```

### Process Substitution

Process substitution (`<(...)`) lets you pass command output as if it were a file. Useful for `diff`, `paste`, and other tools that expect file arguments.

```bash
diff <(sort file1) <(sort file2)
while read -r line; do echo "$line"; done < <(some-command)
```

### Redirection Patterns

```bash
# Discard error output
command 2>/dev/null

# Redirect stderr to stdout
command 2>&1

# Redirect both stdout and stderr to file
command &> output.txt

# Redirect both to file AND stdout (logging + display)
command 2>&1 | tee log.txt
```

### Here Strings and Here Documents

```bash
# Here string (pass a string as stdin)
grep "pattern" <<< "string to search"

# Here document (multi-line input)
cat << 'EOF' > script.sh
#!/bin/bash
echo "hello"
EOF
```

### Miscellaneous

```bash
# Command substitution
echo "Today is $(date +%F)"
files=$(ls *.txt)

# Arithmetic
echo $((2 + 3))
echo $((10 ** 2))
((count++))

# Time a command
time tar czf archive.tar.gz dir/

# Answer yes/no to all prompts
yes | command
yes n | command
```
