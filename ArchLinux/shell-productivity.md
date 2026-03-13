# Shell Productivity

## Keyboard Shortcuts (Bash/Zsh — Emacs Mode)

### Movement

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
| `Ctrl+R` | Reverse search history |
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

```bash
# Show command history
history
history 20              # last 20 commands

# Search history
history | grep "docker"
Ctrl+R                  # interactive reverse search

# Execute from history
!!                      # repeat last command
!42                     # run command number 42
!docker                 # run last command starting with "docker"
!?keyword               # run last command containing "keyword"

# Reuse arguments from last command
echo !$                 # last argument
echo !^                 # first argument
echo !*                 # all arguments
echo !:2                # second argument (0-indexed: !:0 is the command)
echo !:2-4              # arguments 2 through 4

# Modify and rerun
^old^new                # replace first occurrence in last command
!!:gs/old/new           # replace all occurrences in last command

# History settings (add to ~/.bashrc)
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth    # ignore duplicates and space-prefixed
export HISTIGNORE="ls:cd:exit"   # ignore specific commands
export HISTTIMEFORMAT="%F %T "   # add timestamps

# Append (don't overwrite) history on shell exit
shopt -s histappend

# Save history after every command
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"

# Clear history
history -c              # clear session history
history -w              # write current history to file
```

## Aliases

```bash
# Define aliases (add to ~/.bashrc or ~/.zshrc)
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'
alias mkdir='mkdir -pv'

# Safety aliases
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

# Show all aliases
alias

# Remove an alias (current session)
unalias ll

# Bypass alias (run original command)
\rm file.txt
command rm file.txt
```

## Functions

```bash
# Define in ~/.bashrc or ~/.zshrc

# Create directory and cd into it
mkcd() { mkdir -p "$1" && cd "$1"; }

# Extract any archive
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

# Quick backup
bak() { cp "$1" "$1.bak.$(date +%F_%H%M%S)"; }
```

## Globbing (Pattern Matching)

```bash
# Basic wildcards
ls *.txt              # matches any .txt file
ls file?.txt          # matches file1.txt, fileA.txt, etc.
ls file[123].txt      # matches file1.txt, file2.txt, file3.txt
ls file[a-z].txt      # matches filea.txt through filez.txt
ls file[!0-9].txt     # matches files NOT ending in digit

# Extended globbing (bash — enable with: shopt -s extglob)
ls !(*.log)           # everything except .log files
ls +(*.jpg|*.png)     # .jpg or .png files
ls ?(a|b).txt         # optional match: .txt, a.txt, or b.txt
ls @(*.jpg|*.png)     # exactly one of the patterns
ls *(pattern)         # zero or more of the pattern

# Recursive globbing (bash — enable with: shopt -s globstar)
ls **/*.py            # all .py files in all subdirectories

# Zsh globbing (more powerful)
ls **/*.py            # recursive (enabled by default)
ls *.txt(.)           # regular files only
ls *(/)               # directories only
ls *(@)               # symlinks only
ls *(m-7)             # modified in last 7 days
ls *(Lk+100)          # larger than 100KB
ls *(om[1,5])         # 5 most recently modified

# Null glob (no error if no match)
# bash: shopt -s nullglob
# zsh:  setopt nullglob
```

## Parameter Expansion

```bash
# Variables
name="file.tar.gz"
path="/home/jo/documents/file.txt"

# String length
echo ${#name}                  # 11

# Default value
echo ${var:-default}           # use "default" if $var is unset/empty
echo ${var:=default}           # set $var to "default" if unset/empty

# Substring
echo ${name:0:4}               # "file"
echo ${name:5}                 # "tar.gz"
echo ${name: -2}               # "gz" (space before - is required)

# Remove prefix pattern
echo ${name#*.}                # "tar.gz"  (shortest match)
echo ${name##*.}               # "gz"      (longest match)
echo ${path#*/}                # "home/jo/documents/file.txt"
echo ${path##*/}               # "file.txt" (basename)

# Remove suffix pattern
echo ${name%.*}                # "file.tar" (shortest match)
echo ${name%%.*}               # "file"     (longest match)
echo ${path%/*}                # "/home/jo/documents" (dirname)

# Replace
echo ${name/tar/zip}           # "file.zip.gz"  (first match)
echo ${name//./,}              # "file,tar,gz"   (all matches)
echo ${name/#file/doc}         # "doc.tar.gz"    (prefix)
echo ${name/%.gz/.xz}          # "file.tar.xz"   (suffix)

# Case conversion (bash 4+)
echo ${name^^}                 # "FILE.TAR.GZ" (uppercase)
echo ${name,,}                 # "file.tar.gz" (lowercase)
echo ${name^}                  # "File.tar.gz" (capitalize first)

# Indirect reference
ref="name"
echo ${!ref}                   # value of $name

# Array operations
arr=(one two three)
echo ${arr[@]}                 # all elements
echo ${#arr[@]}                # array length
echo ${arr[@]:1:2}             # slice: "two three"
```

## Brace Expansion

```bash
# Sequence
echo {1..10}                   # 1 2 3 4 5 6 7 8 9 10
echo {a..z}                    # a b c ... z
echo {01..12}                  # 01 02 03 ... 12 (zero-padded)
echo {1..20..2}                # 1 3 5 7 ... 19 (step)

# Combinations
echo {a,b}{1,2}                # a1 a2 b1 b2

# Practical uses
mkdir -p project/{src,test,docs}/{main,utils}
cp file.txt{,.bak}            # copy file.txt to file.txt.bak
mv file.{old,new}             # rename file.old to file.new
touch file{1..10}.txt         # create file1.txt through file10.txt
```

## Job Control

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

# Disown (detach from shell, survives logout)
disown %1
disown -a                      # all jobs

# Wait for background jobs
wait                           # all
wait %1                        # specific job
wait $PID                      # specific PID
```

## Miscellaneous Tricks

```bash
# Run previous command with sudo
sudo !!

# Quick directory switching
cd -                           # toggle between last two dirs
pushd /new/dir                 # push to dir stack
popd                           # pop from dir stack
dirs -v                        # show dir stack

# Process substitution
diff <(sort file1) <(sort file2)
while read -r line; do echo "$line"; done < <(some-command)

# Here strings
grep "pattern" <<< "string to search"

# Command substitution
echo "Today is $(date +%F)"
files=$(ls *.txt)

# Arithmetic
echo $((2 + 3))
echo $((10 ** 2))
((count++))

# Quick file content
cat << 'EOF' > script.sh
#!/bin/bash
echo "hello"
EOF

# Redirect stderr
command 2>/dev/null            # discard errors
command 2>&1                   # stderr to stdout
command &> output.txt          # both to file
command 2>&1 | tee log.txt     # both to file and stdout

# Time a command
time tar czf archive.tar.gz dir/

# Yes/no to all prompts
yes | command
yes n | command
```
