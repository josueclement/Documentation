# File Searching

## find

```bash
# Find files by name
find /etc -name "*.conf"

# Case-insensitive name search
find /home -iname "*.jpg"

# Find directories only
find /var -type d -name "log*"

# Find files only
find . -type f -name "*.txt"

# Find by size (greater than 100MB)
find / -type f -size +100M

# Find files modified in the last 7 days
find /home -type f -mtime -7

# Find files modified more than 30 days ago
find /var/log -type f -mtime +30

# Find files changed in the last 60 minutes
find . -type f -mmin -60

# Find empty files and directories
find /tmp -empty

# Find by permissions
find / -type f -perm 777
find / -type f -perm /u+s    # SUID bit set

# Find by owner
find /home -type f -user jo
find /var -type f -group wheel

# Combine conditions with AND (default)
find . -type f -name "*.log" -size +10M

# Combine conditions with OR
find . -type f \( -name "*.jpg" -o -name "*.png" \)

# Negate a condition
find . -type f ! -name "*.tmp"

# Limit search depth
find . -maxdepth 2 -name "*.conf"
find /etc -mindepth 1 -maxdepth 1 -type d

# Execute a command on each result
find . -type f -name "*.log" -exec rm {} \;

# Execute with confirmation
find . -type f -name "*.bak" -ok rm {} \;

# More efficient: batch execution with +
find . -type f -name "*.txt" -exec grep -l "TODO" {} +

# Delete matching files directly
find /tmp -type f -name "*.tmp" -mtime +7 -delete

# Print with custom format
find . -type f -printf "%T@ %p\n" | sort -n    # sort by modification time

# Find broken symlinks
find . -xtype l

# Exclude a directory
find . -path ./node_modules -prune -o -type f -name "*.js" -print

# Exclude multiple directories
find . \( -path ./.git -o -path ./vendor \) -prune -o -type f -print
```

## fd (Modern Alternative to find)

**Install:** `pacman -S fd`

```bash
# Simple filename search (recursive, smart-case by default)
fd "\.conf$" /etc

# Find by extension
fd -e jpg
fd -e txt -e md

# Search hidden files (excluded by default)
fd -H ".bashrc"

# Search ignored files (.gitignore'd)
fd -I "node_modules"

# Include hidden AND ignored
fd -HI "*.log"

# Directories only
fd -t d "src"

# Files only
fd -t f "config"

# Symlinks only
fd -t l

# Executable files only
fd -t x

# Empty files
fd -t e

# Limit depth
fd -d 2 "*.md"

# Exclude directories
fd -E node_modules -E .git "\.js$"

# Absolute paths
fd -a "\.conf$" /etc

# Execute command on each result
fd -e log -x gzip {}

# Execute command with parallel jobs
fd -e png -x convert {} {.}.webp    # {.} = path without extension

# Batch execute (all results as arguments)
fd -e txt -X wc -l

# Use with other placeholders
# {.}  path without extension
# {/}  basename
# {//} parent directory
# {/.} basename without extension
fd -e jpg -x mv {} {//}/thumbnails/{/}

# Change base directory
fd -e rs --base-directory /usr/src

# Filter by size
fd -S +1m -e log

# Filter by modification time
fd --changed-within 1d
fd --changed-before "2025-01-01"

# Custom color output
fd -e py --color always | less -R
```

## locate / plocate

**Install:** `pacman -S plocate`

```bash
# Update the database (run as root, or via timer)
sudo updatedb

# Simple search
locate bashrc

# Case-insensitive
locate -i readme

# Count matches
locate -c "*.conf"

# Limit number of results
locate -l 10 "*.log"

# Show only existing files (filter stale entries)
locate -e "*.conf"

# Use regex
locate -r "/etc/.*\.conf$"

# Ignore case with regex
locate -ri "readme\.md$"

# Show statistics about the database
locate -S
```

## grep (Searching File Contents)

```bash
# Search for a pattern in files
grep "error" /var/log/syslog

# Recursive search
grep -r "TODO" ./src

# Recursive, skip binary files (default in most setups)
grep -rI "password" /etc

# Case-insensitive
grep -ri "error" /var/log/

# Show line numbers
grep -rn "function" ./src

# Show filenames only
grep -rl "import React" ./src

# Show count of matches per file
grep -rc "TODO" ./src

# Show context lines (before/after/both)
grep -B 3 "panic" /var/log/syslog    # 3 lines before
grep -A 5 "error" app.log            # 5 lines after
grep -C 2 "fatal" app.log            # 2 lines before and after

# Extended regex
grep -E "(error|warning|fatal)" /var/log/syslog

# Perl-compatible regex
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" access.log

# Invert match (lines that DON'T match)
grep -v "^#" /etc/fstab    # non-comment lines

# Match whole words only
grep -w "error" app.log    # won't match "errors" or "terror"

# Match whole lines
grep -x "exact line content" file.txt

# Fixed string search (no regex interpretation)
grep -F "user.name[0]" config.txt

# Search multiple patterns
grep -e "error" -e "warning" /var/log/syslog

# Search from a pattern file
grep -f patterns.txt /var/log/syslog

# Include/exclude file patterns
grep -r --include="*.py" "import" ./src
grep -r --exclude="*.min.js" "function" ./src
grep -r --exclude-dir=".git" "TODO" .

# Null-separated output (for piping to xargs)
grep -rZl "TODO" ./src | xargs -0 rm

# Show only the matching part
grep -o "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}" access.log

# Quiet mode (exit code only — useful in scripts)
if grep -q "error" app.log; then echo "Errors found"; fi
```

## ripgrep (rg)

**Install:** `pacman -S ripgrep`

```bash
# Basic search (recursive, respects .gitignore by default)
rg "TODO"

# Search specific directory
rg "error" /var/log

# Search specific file types
rg -t py "import"
rg -t js -t ts "fetch"

# List available types
rg --type-list

# Define custom types
rg --type-add "web:*.{html,css,js}" -t web "color"

# Case-insensitive
rg -i "error"

# Smart case (case-insensitive unless uppercase is used)
rg -S "Error"    # case-sensitive because of capital E

# Fixed string (literal, no regex)
rg -F "user.name[0]"

# Show context
rg -C 3 "panic"
rg -B 2 -A 5 "error"

# File matches only
rg -l "TODO"

# Files without matches
rg --files-without-match "TODO"

# Count matches
rg -c "error" /var/log

# Include hidden files
rg --hidden "secret"

# Don't respect .gitignore
rg --no-ignore "debug"

# Search everything (hidden + ignored)
rg -uu "password"

# Glob filtering
rg -g "*.py" "import"
rg -g "!*.min.js" "function"
rg -g "src/**/*.ts" "interface"

# Multiline matching
rg -U "fn main.*\{[\s\S]*?\}"

# Replace text (preview only — doesn't modify files)
rg "foo" -r "bar"

# Show only the matched part
rg -o "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

# JSON output (for scripting)
rg --json "TODO"

# Sort results by path
rg --sort path "error"

# Limit max depth
rg --max-depth 2 "config"

# Max results count
rg -m 5 "error"    # stop after 5 matches per file

# Search compressed files
rg -z "error" logs.gz

# Null-byte separator (for xargs)
rg -0 -l "TODO" | xargs -0 sed -i 's/TODO/DONE/g'

# Stats
rg --stats "TODO"

# Use PCRE2 engine (for lookahead/lookbehind)
rg -P "(?<=user=)\w+" access.log
```

## Combining Tools

```bash
# Find files then search contents
find . -name "*.conf" -exec grep -l "Listen" {} +

# fd + rg pipeline
fd -e py | xargs rg "import os"

# fd + rg: search only recently modified files
fd -e log --changed-within 1h -x rg "error" {}

# locate + grep: fast two-stage search
locate -r "\.conf$" | xargs grep -l "port"

# Find large log files with errors
find /var/log -name "*.log" -size +10M -exec grep -l "ERROR" {} +

# Find duplicate filenames
fd -t f | awk -F/ '{print $NF}' | sort | uniq -d

# Find files and open in editor
rg -l "TODO" | xargs $EDITOR

# Count lines of code by extension
fd -e py -x wc -l | tail -1
fd -e py | xargs wc -l | sort -n

# Interactive search with fzf
fd -t f | fzf --preview 'bat {}'
rg --files | fzf --preview 'head -50 {}'
```
