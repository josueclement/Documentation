# Text Processing

Transforming, filtering, and analyzing text from the command line with `sed`, `awk`, `cut`, `sort`, `uniq`, `tr`, `xargs`, `column`, `paste`, `wc`, and `tee`.

## sed (Stream Editor)

`sed` reads input line by line, applies editing commands, and writes the result to stdout. The most common operation is `s/pattern/replacement/` for substitution. By default it doesn't modify the original file — use `-i` for in-place editing.

### Substitution

The `s` command replaces text. The `g` flag at the end means "all occurrences on the line" — without it, only the first match per line is replaced.

```bash
# Replace first occurrence on each line
sed 's/foo/bar/' file.txt

# Replace all occurrences on each line
sed 's/foo/bar/g' file.txt

# Case-insensitive replace
sed 's/foo/bar/gi' file.txt
```

### In-Place Editing

`-i` modifies the file directly. Use `-i.bak` to create a backup before editing.

```bash
# Edit file in place
sed -i 's/foo/bar/g' file.txt

# Create backup before in-place edit
sed -i.bak 's/foo/bar/g' file.txt
```

### Line-Specific Operations

You can target substitutions to specific lines by number, range, or pattern match.

```bash
# Replace on specific line only
sed '5s/foo/bar/' file.txt

# Replace in line range
sed '10,20s/foo/bar/g' file.txt

# Replace from pattern to end of file
sed '/START/,$ s/foo/bar/g' file.txt

# Replace between two patterns
sed '/BEGIN/,/END/ s/foo/bar/g' file.txt
```

### Deleting Lines

The `d` command removes lines. Can target by line number, range, or pattern.

```bash
sed '5d' file.txt             # delete line 5
sed '10,20d' file.txt         # delete lines 10-20
sed '/^#/d' file.txt          # delete comment lines
sed '/^$/d' file.txt          # delete empty lines
sed '/pattern/d' file.txt     # delete lines matching pattern
```

### Inserting, Appending, and Replacing Lines

`i\` inserts before a line, `a\` appends after, and `c\` replaces the entire line.

```bash
# Insert text before line 3
sed '3i\New line of text' file.txt

# Append text after line 3
sed '3a\New line of text' file.txt

# Insert before a pattern
sed '/pattern/i\Inserted line' file.txt

# Replace entire line matching pattern
sed '/pattern/c\Replacement line' file.txt
```

### Printing Specific Lines

The `-n` flag suppresses default output, so only lines explicitly printed with `p` are shown — useful for extracting ranges.

```bash
# Print only matching lines (like grep)
sed -n '/pattern/p' file.txt

# Print line 5 only
sed -n '5p' file.txt

# Print lines 10-20
sed -n '10,20p' file.txt
```

### Multiple Operations and Alternate Delimiters

Use `-e` to chain multiple operations. When patterns contain `/` (like file paths), use an alternate delimiter to avoid escaping.

```bash
# Multiple operations
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Use | as delimiter (useful with paths)
sed 's|/usr/local|/opt|g' file.txt

# Use # as delimiter
sed 's#http://#https://#g' file.txt
```

### Capture Groups and Back-references

Capture groups let you rearrange matched text. Use `\(` and `\)` in basic regex, or `(` and `)` with `-E` (extended regex). Refer to groups with `\1`, `\2`, etc.

```bash
# Swap values around = sign
sed 's/\(.*\)=\(.*\)/\2=\1/' file.txt

# Same with extended regex (cleaner syntax)
sed -E 's/(.*)=(.*)/\2=\1/' file.txt
```

### Modifying Line Beginnings and Endings

```bash
# Add text to beginning of every line
sed 's/^/PREFIX: /' file.txt

# Add text to end of every line
sed 's/$/ # comment/' file.txt

# Remove leading whitespace
sed 's/^[[:space:]]*//' file.txt

# Remove trailing whitespace
sed 's/[[:space:]]*$//' file.txt

# Remove both leading and trailing whitespace
sed 's/^[[:space:]]*//;s/[[:space:]]*$//' file.txt
```

### Character Transliteration

The `y` command maps characters one-to-one, like `tr`. Each character in the first set is replaced by the corresponding character in the second.

```bash
sed 'y/abc/ABC/' file.txt
```

## awk

`awk` is a pattern-scanning and text-processing language. It automatically splits each line into fields (`$1`, `$2`, ...) by whitespace. It excels at column-based operations, filtering, and aggregation — essentially a mini programming language for structured text.

### Printing Fields

Fields are numbered starting at `$1`. `$0` is the entire line. `$NF` is the last field, `$(NF-1)` is second-to-last.

```bash
# Print first column
awk '{print $1}' file.txt

# Print first and third columns
awk '{print $1, $3}' file.txt

# Print last column
awk '{print $NF}' file.txt

# Print second to last column
awk '{print $(NF-1)}' file.txt
```

### Custom Field Separators

`-F` sets the input field separator. `-v OFS=` sets the output field separator used when printing multiple fields.

```bash
# Colon-separated input (like /etc/passwd)
awk -F: '{print $1, $3}' /etc/passwd

# Comma-separated input (CSV)
awk -F',' '{print $2}' data.csv

# Set output separator to tab
awk -F: -v OFS='\t' '{print $1, $3, $7}' /etc/passwd
```

### Filtering with Patterns

A pattern before the action block `{}` limits which lines are processed. Patterns can be regexes, comparisons, or field tests.

```bash
# Lines containing "error"
awk '/error/' file.txt

# Lines NOT starting with #
awk '!/^#/' file.txt

# Third column greater than 100
awk '$3 > 100' file.txt

# First field equals "root"
awk '$1 == "root"' /etc/passwd

# UID >= 1000 (colon-separated)
awk -F: '$3 >= 1000' /etc/passwd
```

### Built-in Variables

`NR` is the current line number across all files, `FNR` is per-file, `NF` is the number of fields on the current line.

```bash
# Print line numbers
awk '{print NR, $0}' file.txt

# Count lines
awk 'END {print NR}' file.txt

# Print lines longer than 80 characters
awk 'length > 80' file.txt

# Print between two patterns (inclusive)
awk '/START/,/END/' file.txt

# Skip header row
awk 'NR > 1 {print $2}' data.tsv

# Process first line differently
awk 'NR==1 {print "Header:", $0} NR>1 {print "Data:", $0}' file.txt
```

### Aggregation

`awk` keeps state between lines, making it ideal for sums, counts, averages, and min/max calculations.

```bash
# Sum a column
awk '{sum += $3} END {print sum}' data.txt

# Average
awk '{sum += $1; n++} END {print sum/n}' numbers.txt

# Min/max
awk 'NR==1 || $3 > max {max=$3} END {print max}' data.txt
```

### Modifying and Formatting Output

```bash
# Replace field values
awk -F: -v OFS=':' '$3 == 0 {$1 = "SUPERUSER"} {print}' /etc/passwd

# Formatted output with printf
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd

# Multiple rules (count errors and warnings separately)
awk '/error/ {errors++} /warning/ {warnings++} END {print errors, warnings}' log.txt
```

### Associative Arrays

`awk` supports associative arrays (hash maps), useful for frequency counting and grouping.

```bash
# Frequency count (e.g., count requests per IP)
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log | sort -rn
```

### Built-in Variables Reference

| Variable | Meaning |
|----------|---------|
| `NR` | Current line number (across all files) |
| `FNR` | Current line number (in current file) |
| `NF` | Number of fields in current line |
| `FS` | Input field separator |
| `OFS` | Output field separator |
| `RS` | Input record separator |
| `ORS` | Output record separator |

## cut

`cut` extracts sections from each line — by character position, byte position, or delimited field. It's simpler than `awk` for quick column extraction but doesn't support regex delimiters.

### Cutting by Character Position

```bash
# Characters 1-10
cut -c1-10 file.txt

# Character 5 to end of line
cut -c5- file.txt

# First 20 characters
cut -c-20 file.txt
```

### Cutting by Delimited Field

`-d` sets the delimiter (single character), `-f` selects fields. Default delimiter is tab.

```bash
# First field of colon-delimited file
cut -d: -f1 /etc/passwd

# Multiple fields
cut -d: -f1,3,7 /etc/passwd

# Field range
cut -d',' -f2-4 data.csv
```

### Other Options

```bash
# Cut by bytes (different from characters for multi-byte encodings)
cut -b1-16 file.bin

# Complement — everything EXCEPT specified fields
cut -d: --complement -f2 /etc/passwd

# Change output delimiter
cut -d: -f1,7 --output-delimiter=$'\t' /etc/passwd
```

## sort

`sort` orders lines alphabetically by default. Supports numeric, human-readable, version, and month sorting. Can sort by specific columns (keys) and remove duplicates.

### Basic Sorting

```bash
# Alphabetical sort
sort file.txt

# Reverse sort
sort -r file.txt

# Numeric sort (treats lines as numbers)
sort -n numbers.txt

# Human-readable numeric sort (understands 1K, 2M, 3G)
sort -h sizes.txt
```

### Sorting by Column

`-k` specifies the sort key. Format is `-k START,END` where START and END are field numbers. Append `n` for numeric, `r` for reverse.

```bash
# Sort by 2nd column
sort -k2 file.txt

# Numeric sort by 2nd column only
sort -k2,2n file.txt

# Sort /etc/passwd by UID (3rd colon-separated field)
sort -t: -k3,3n /etc/passwd

# Sort by multiple keys (alphabetical by col1, then numeric by col2)
sort -k1,1 -k2,2n file.txt
```

### Other Options

```bash
# Remove duplicates while sorting
sort -u file.txt

# Case-insensitive sort
sort -f file.txt

# Check if already sorted (exit code 0 if sorted)
sort -c file.txt

# Stable sort (preserve original order for equal elements)
sort -s -k2,2 file.txt

# Sort by month name (Jan, Feb, ...)
sort -M dates.txt

# Random shuffle
sort -R file.txt

# Sort IP addresses correctly
sort -t. -k1,1n -k2,2n -k3,3n -k4,4n ips.txt

# Sort version numbers (1.2.3 < 1.10.1)
sort -V versions.txt
```

## uniq

`uniq` filters out **adjacent** duplicate lines. Since it only compares consecutive lines, you almost always need to `sort` the input first.

```bash
# Remove adjacent duplicates
sort file.txt | uniq

# Count occurrences of each unique line
sort file.txt | uniq -c

# Show only lines that appear more than once
sort file.txt | uniq -d

# Show only lines that appear exactly once
sort file.txt | uniq -u

# Ignore case when comparing
sort file.txt | uniq -i

# Skip first N fields when comparing
sort file.txt | uniq -f 1

# Skip first N characters when comparing
sort file.txt | uniq -s 5

# Top 10 most frequent lines
sort file.txt | uniq -c | sort -rn | head -10
```

## tr (Translate/Delete Characters)

`tr` translates, squeezes, or deletes characters. It works on individual characters — not strings or words. It reads from stdin only (no filename argument).

### Translating Characters

The first set is mapped character-by-character to the second set.

```bash
# Lowercase to uppercase
echo "hello" | tr 'a-z' 'A-Z'

# Uppercase to lowercase
echo "HELLO" | tr 'A-Z' 'a-z'

# Replace spaces with underscores
echo "hello world" | tr ' ' '_'

# Tabs to spaces
cat file.txt | tr '\t' ' '
```

### Squeezing Repeated Characters

`-s` collapses runs of repeated characters down to a single one.

```bash
# Squeeze repeated letters
echo "aabbcc" | tr -s 'a-z'          # abc

# Squeeze multiple spaces into one
echo "too   many   spaces" | tr -s ' '
```

### Deleting Characters

`-d` removes all occurrences of the specified characters.

```bash
# Delete all digits
echo "hello 123 world" | tr -d '0-9'

# Delete spaces
echo "hello world" | tr -d ' '
```

### Complement (Inverting the Set)

`-c` inverts the character set — useful for keeping only certain characters.

```bash
# Keep only digits and newlines (delete everything else)
echo "hello 123 world" | tr -cd '0-9\n'

# Remove non-printable characters
cat file.bin | tr -cd '[:print:]\n'

# DOS to Unix line endings (remove carriage returns)
tr -d '\r' < dosfile.txt > unixfile.txt
```

### Character Classes

| Class | Matches |
|-------|---------|
| `[:alnum:]` | Letters and digits |
| `[:alpha:]` | Letters |
| `[:digit:]` | Digits |
| `[:lower:]` | Lowercase letters |
| `[:upper:]` | Uppercase letters |
| `[:space:]` | Whitespace |
| `[:print:]` | Printable characters |
| `[:punct:]` | Punctuation |

## xargs

`xargs` reads items from stdin and passes them as arguments to a command. Essential for connecting tools that produce lists (like `find`, `grep -l`) with tools that operate on arguments.

### Basic Usage

```bash
# Pass stdin as arguments to wc
find . -name "*.txt" | xargs wc -l

# Handle filenames with spaces (null delimiter — pair with -print0 or -0)
find . -name "*.txt" -print0 | xargs -0 wc -l
```

### Controlling Argument Batching

`-n` limits how many arguments are passed per invocation. `-I {}` uses a placeholder for each item.

```bash
# Limit to 2 arguments per command invocation
echo "1 2 3 4 5" | xargs -n 2 echo

# Placeholder for argument position
ls *.tar.gz | xargs -I {} tar xzf {} -C /tmp/extracted/

# Run command for each line (not word)
cat urls.txt | xargs -I {} curl -O {}
```

### Parallel Execution

`-P` runs multiple commands in parallel — useful for CPU-bound or I/O-bound tasks.

```bash
# Run 4 compressions in parallel
find . -name "*.png" -print0 | xargs -0 -P 4 -n 1 optipng
```

### Safety and Dry Runs

```bash
# Prompt before each execution
find . -name "*.bak" | xargs -p rm

# Dry run (show commands without executing — use echo)
find . -name "*.log" | xargs -t echo rm
```

### Common Patterns

```bash
# Combine with grep output to do bulk find-and-replace
grep -rl "TODO" src/ | xargs sed -i 's/TODO/DONE/g'
```

## column

`column` formats stdin into aligned columns or tables. Useful for making output readable.

```bash
# Auto-format into aligned columns
mount | column -t

# Specify input delimiter for table mode
cat /etc/passwd | column -t -s ':'

# Create columns from a vertical list
echo -e "one\ntwo\nthree\nfour\nfive\nsix" | column

# Custom output separator
column -t -s ',' -o ' | ' data.csv

# JSON pretty-print (util-linux 2.37+)
echo '{"a":1,"b":2}' | column --json
```

## paste

`paste` merges lines from multiple files side by side, or joins all lines of a single file into one. The opposite of `cut` in a sense — it combines rather than splits.

```bash
# Merge two files side by side (tab-separated by default)
paste file1.txt file2.txt

# Custom delimiter
paste -d ',' file1.txt file2.txt

# Join all lines into one (serial mode)
paste -s file.txt

# Join all lines with a custom delimiter
paste -sd ',' file.txt

# Convert single column to multiple columns (- reads from stdin)
paste - - - < file.txt              # 3 columns
cat file.txt | paste -d ',' - -     # 2 columns, comma-separated

# Combine with cut for column rearrangement
paste <(cut -f3 data.tsv) <(cut -f1 data.tsv)
```

## wc (Word Count)

`wc` counts lines, words, and characters. Frequently used to count files, lines of code, or to verify expected output length.

```bash
# Count lines, words, characters (all three)
wc file.txt

# Lines only (most common)
wc -l file.txt

# Words only
wc -w file.txt

# Characters only
wc -m file.txt

# Bytes only (different from chars for multi-byte encodings)
wc -c file.txt

# Longest line length
wc -L file.txt

# Count files in a directory
ls | wc -l
fd -t f | wc -l

# Count lines across multiple files
wc -l src/*.py

# Count total lines found by find
find . -name "*.py" -exec cat {} + | wc -l
```

## tee

`tee` reads from stdin and writes to both stdout and one or more files simultaneously. Useful for logging pipeline output while still passing it through.

```bash
# Write to file AND stdout
ls -la | tee listing.txt

# Append instead of overwrite
echo "new entry" | tee -a log.txt

# Write to multiple files at once
echo "hello" | tee file1.txt file2.txt file3.txt

# Inspect intermediate pipeline output (debugging)
cat data.txt | sort | tee sorted.txt | uniq -c | tee counts.txt | sort -rn | head

# Write to a protected file via sudo (sudo can't redirect, but tee can)
echo "new line" | sudo tee -a /etc/hosts > /dev/null
```

## Combining Tools — Common Patterns

These are real-world recipes that chain multiple text-processing tools together.

```bash
# Frequency analysis of HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Extract and sort unique IPs
awk '{print $1}' access.log | sort -u

# CSV column statistics — find the max value in column 3
awk -F',' '{print $3}' data.csv | sort -n | tail -1

# Find and replace across multiple files
grep -rl "oldtext" src/ | xargs sed -i 's/oldtext/newtext/g'

# Remove duplicate lines while preserving original order (no sort needed)
awk '!seen[$0]++' file.txt

# Extract values from key=value pairs using lookbehind
grep -oP '(?<=key=)\w+' config.txt

# Transpose rows and columns
awk '{for(i=1;i<=NF;i++) a[i]=a[i]" "$i} END {for(i in a) print a[i]}' data.txt

# Generate numbered list
cat -n file.txt
awk '{printf "%3d. %s\n", NR, $0}' file.txt

# Join a line matching a pattern with the following line
sed '/pattern/{N;s/\n/ /}' file.txt

# Print every Nth line (every 5th line here)
awk 'NR % 5 == 0' file.txt

# Side-by-side diff of sorted versions of two files
diff <(sort file1.txt) <(sort file2.txt)
```
