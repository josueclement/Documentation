# File Searching

Finding files by name, path, size, and date with `find` and `fd`. Searching file contents with `grep` and `ripgrep`. Combining tools for complex lookups.

## find

`find` is the traditional Unix tool for locating files and directories by name, type, size, timestamps, permissions, and ownership. It walks the directory tree and evaluates expressions against each entry. Results can be piped or acted on directly with `-exec`.

### Basic Name Searches

The `-name` flag matches against the filename (not the full path). Use `-iname` for case-insensitive matching. Wrap patterns in quotes to prevent shell glob expansion.

```bash
# Find files by name
find /etc -name "*.conf"

# Case-insensitive name search
find /home -iname "*.jpg"
```

### Filtering by Type

The `-type` flag restricts results. Use `f` for regular files, `d` for directories, `l` for symlinks.

```bash
# Find directories only
find /var -type d -name "log*"

# Find files only
find . -type f -name "*.txt"
```

### Filtering by Size

`-size` accepts suffixes: `c` (bytes), `k` (kilobytes), `M` (megabytes), `G` (gigabytes). Prefix with `+` for "greater than" or `-` for "less than".

```bash
# Find files greater than 100MB
find / -type f -size +100M
```

### Filtering by Time

`-mtime` works in days, `-mmin` in minutes. Prefix with `-` for "less than N ago" and `+` for "more than N ago". There are also `-atime` (access) and `-ctime` (change) variants.

```bash
# Find files modified in the last 7 days
find /home -type f -mtime -7

# Find files modified more than 30 days ago
find /var/log -type f -mtime +30

# Find files changed in the last 60 minutes
find . -type f -mmin -60
```

### Other Filters

```bash
# Find empty files and directories
find /tmp -empty

# Find by permissions (exact match)
find / -type f -perm 777

# Find files with SUID bit set
find / -type f -perm /u+s

# Find by owner
find /home -type f -user jo
find /var -type f -group wheel
```

### Combining Conditions

Multiple conditions are ANDed by default. Use `\( -o \)` for OR and `!` for negation. Parentheses must be escaped or quoted.

```bash
# AND (default) — both conditions must match
find . -type f -name "*.log" -size +10M

# OR — either condition matches
find . -type f \( -name "*.jpg" -o -name "*.png" \)

# Negate a condition
find . -type f ! -name "*.tmp"
```

### Controlling Depth

`-maxdepth` and `-mindepth` control how deep `find` recurses. Depth 1 means only direct children of the starting path.

```bash
# Only search 2 levels deep
find . -maxdepth 2 -name "*.conf"

# Only top-level directories in /etc
find /etc -mindepth 1 -maxdepth 1 -type d
```

### Executing Commands on Results

`-exec` runs a command for each match. `{}` is replaced by the file path. End with `\;` for one invocation per file, or `+` to batch them (more efficient).

```bash
# Run command on each result
find . -type f -name "*.log" -exec rm {} \;

# Execute with confirmation prompt
find . -type f -name "*.bak" -ok rm {} \;

# Batch execution (pass all results at once — more efficient)
find . -type f -name "*.txt" -exec grep -l "TODO" {} +

# Delete matching files directly (faster than -exec rm)
find /tmp -type f -name "*.tmp" -mtime +7 -delete
```

### Advanced Tricks

```bash
# Print with custom format and sort by modification time
find . -type f -printf "%T@ %p\n" | sort -n

# Find broken symlinks
find . -xtype l

# Exclude a directory from search
find . -path ./node_modules -prune -o -type f -name "*.js" -print

# Exclude multiple directories
find . \( -path ./.git -o -path ./vendor \) -prune -o -type f -print
```

## fd (Modern Alternative to find)

`fd` is a fast, user-friendly alternative to `find`. It uses smart-case matching by default, respects `.gitignore`, and has a simpler syntax. Output is colorized and recursive by default.

**Install:** `pacman -S fd`

### Basic Usage

By default, `fd` performs a regex search across all filenames recursively. It ignores hidden files and `.gitignore` entries unless told otherwise.

```bash
# Simple filename search (recursive, smart-case by default)
fd "\.conf$" /etc

# Find by extension
fd -e jpg
fd -e txt -e md
```

### Searching Hidden and Ignored Files

By default `fd` skips hidden files (dotfiles) and files matched by `.gitignore`. These flags override that.

```bash
# Search hidden files (excluded by default)
fd -H ".bashrc"

# Search ignored files (.gitignore'd)
fd -I "node_modules"

# Include hidden AND ignored
fd -HI "*.log"
```

### Filtering by Type

Similar to `find -type` but with shorter flags: `f` (file), `d` (directory), `l` (symlink), `x` (executable), `e` (empty).

```bash
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
```

### Depth, Exclusion, and Paths

```bash
# Limit depth
fd -d 2 "*.md"

# Exclude directories
fd -E node_modules -E .git "\.js$"

# Absolute paths
fd -a "\.conf$" /etc

# Change base directory
fd -e rs --base-directory /usr/src
```

### Executing Commands

`fd` supports `-x` (run per result) and `-X` (batch). Placeholders: `{}` (full path), `{.}` (without extension), `{/}` (basename), `{//}` (parent), `{/.}` (basename without extension).

```bash
# Execute command on each result
fd -e log -x gzip {}

# Execute command with parallel jobs
fd -e png -x convert {} {.}.webp

# Batch execute (all results as arguments)
fd -e txt -X wc -l

# Move files using placeholders
fd -e jpg -x mv {} {//}/thumbnails/{/}
```

### Filtering by Size and Time

```bash
# Filter by size (greater than 1MB)
fd -S +1m -e log

# Filter by modification time
fd --changed-within 1d
fd --changed-before "2025-01-01"
```

## locate / plocate

`locate` finds files by searching a pre-built database rather than scanning the filesystem in real time. This makes it extremely fast for simple filename lookups, but the database must be updated periodically to reflect recent changes.

**Install:** `pacman -S plocate`

```bash
# Update the database (run as root, or via timer)
sudo updatedb

# Simple search — matches anywhere in the path
locate bashrc

# Case-insensitive
locate -i readme

# Count matches instead of listing them
locate -c "*.conf"

# Limit number of results
locate -l 10 "*.log"

# Show only files that still exist (filter stale entries)
locate -e "*.conf"

# Use regex instead of glob matching
locate -r "/etc/.*\.conf$"

# Ignore case with regex
locate -ri "readme\.md$"

# Show database statistics
locate -S
```

## grep (Searching File Contents)

`grep` searches text by matching lines against a pattern. It's the standard tool for searching inside files. Use `-r` for recursive directory searches, `-E` for extended regex, and `-P` for Perl-compatible regex.

### Basic Searching

```bash
# Search for a pattern in a file
grep "error" /var/log/syslog

# Recursive search through a directory
grep -r "TODO" ./src

# Recursive, skip binary files (default in most setups)
grep -rI "password" /etc

# Case-insensitive
grep -ri "error" /var/log/
```

### Output Control

These flags control what `grep` shows — line numbers, filenames only, counts, or surrounding context lines.

```bash
# Show line numbers
grep -rn "function" ./src

# Show filenames only (not matching lines)
grep -rl "import React" ./src

# Show count of matches per file
grep -rc "TODO" ./src

# Show context lines (before/after/both)
grep -B 3 "panic" /var/log/syslog    # 3 lines before
grep -A 5 "error" app.log            # 5 lines after
grep -C 2 "fatal" app.log            # 2 lines before and after
```

### Regex Modes

`grep` supports basic regex by default. `-E` enables extended regex (ERE) with `|`, `+`, `?` without escaping. `-P` enables Perl-compatible regex (PCRE) with lookahead/lookbehind.

```bash
# Extended regex (ERE)
grep -E "(error|warning|fatal)" /var/log/syslog

# Perl-compatible regex (PCRE)
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" access.log
```

### Inverting, Matching Whole Words, and Fixed Strings

```bash
# Invert match (lines that DON'T match)
grep -v "^#" /etc/fstab    # non-comment lines

# Match whole words only (won't match "errors" or "terror")
grep -w "error" app.log

# Match whole lines
grep -x "exact line content" file.txt

# Fixed string search (no regex interpretation — faster, safe for special chars)
grep -F "user.name[0]" config.txt
```

### Multiple Patterns

```bash
# Search multiple patterns with -e
grep -e "error" -e "warning" /var/log/syslog

# Search patterns from a file
grep -f patterns.txt /var/log/syslog
```

### Include/Exclude File Patterns

When searching recursively, these flags let you limit which files are searched by name or directory.

```bash
# Include only Python files
grep -r --include="*.py" "import" ./src

# Exclude minified JavaScript
grep -r --exclude="*.min.js" "function" ./src

# Exclude directories
grep -r --exclude-dir=".git" "TODO" .
```

### Extracting and Scripting

```bash
# Null-separated output (safe for filenames with spaces, for piping to xargs)
grep -rZl "TODO" ./src | xargs -0 rm

# Show only the matching part (not the whole line)
grep -o "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}" access.log

# Quiet mode (exit code only — useful in if-statements)
if grep -q "error" app.log; then echo "Errors found"; fi
```

## ripgrep (rg)

`ripgrep` is a fast, modern alternative to `grep -r`. It respects `.gitignore` by default, searches recursively, supports file-type filters, and has sensible defaults. Generally 2-5x faster than `grep` on large codebases.

**Install:** `pacman -S ripgrep`

### Basic Usage

```bash
# Basic search (recursive, respects .gitignore by default)
rg "TODO"

# Search specific directory
rg "error" /var/log

# Fixed string (literal, no regex)
rg -F "user.name[0]"
```

### File Type Filtering

`rg` has built-in knowledge of file types. Use `-t` to include or `-T` to exclude types. Use `--type-list` to see all recognized types.

```bash
# Search only Python files
rg -t py "import"

# Search JavaScript and TypeScript
rg -t js -t ts "fetch"

# List available types
rg --type-list

# Define custom types
rg --type-add "web:*.{html,css,js}" -t web "color"
```

### Case Sensitivity

By default, `rg` is case-sensitive. `-i` makes it case-insensitive. `-S` (smart-case) is case-insensitive unless your pattern contains uppercase characters.

```bash
# Case-insensitive
rg -i "error"

# Smart case (case-insensitive unless uppercase is used)
rg -S "Error"    # case-sensitive because of capital E
```

### Context and Output

```bash
# Show context lines (before, after, or both)
rg -C 3 "panic"
rg -B 2 -A 5 "error"

# File matches only (like grep -l)
rg -l "TODO"

# Files without matches
rg --files-without-match "TODO"

# Count matches per file
rg -c "error" /var/log
```

### Hidden and Ignored Files

By default `rg` skips hidden files and respects `.gitignore`. Each `-u` flag removes one layer of filtering.

```bash
# Include hidden files
rg --hidden "secret"

# Don't respect .gitignore
rg --no-ignore "debug"

# Search everything (hidden + ignored)
rg -uu "password"
```

### Glob Filtering

`-g` filters files by glob pattern. Prefix with `!` to exclude. This is more flexible than `-t` for custom patterns.

```bash
# Include only Python files
rg -g "*.py" "import"

# Exclude minified files
rg -g "!*.min.js" "function"

# Path-based glob
rg -g "src/**/*.ts" "interface"
```

### Advanced Features

```bash
# Multiline matching
rg -U "fn main.*\{[\s\S]*?\}"

# Replace text (preview only — doesn't modify files)
rg "foo" -r "bar"

# Show only the matched part (like grep -o)
rg -o "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

# JSON output (for scripting)
rg --json "TODO"

# Sort results by path
rg --sort path "error"

# Limit max depth
rg --max-depth 2 "config"

# Max results count (stop after 5 matches per file)
rg -m 5 "error"

# Search compressed files
rg -z "error" logs.gz

# Null-byte separator (for xargs)
rg -0 -l "TODO" | xargs -0 sed -i 's/TODO/DONE/g'

# Print statistics
rg --stats "TODO"

# Use PCRE2 engine (for lookahead/lookbehind)
rg -P "(?<=user=)\w+" access.log
```

## Combining Tools

Piping search tools together lets you build powerful multi-stage lookups — find files by name, then search their contents, or filter results through additional commands.

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
