# Text Processing

## sed (Stream Editor)

```bash
# Replace first occurrence on each line
sed 's/foo/bar/' file.txt

# Replace all occurrences on each line
sed 's/foo/bar/g' file.txt

# Edit file in place
sed -i 's/foo/bar/g' file.txt

# Create backup before in-place edit
sed -i.bak 's/foo/bar/g' file.txt

# Case-insensitive replace
sed 's/foo/bar/gi' file.txt

# Replace on specific line
sed '5s/foo/bar/' file.txt

# Replace in line range
sed '10,20s/foo/bar/g' file.txt

# Replace from pattern to end of file
sed '/START/,$ s/foo/bar/g' file.txt

# Replace between two patterns
sed '/BEGIN/,/END/ s/foo/bar/g' file.txt

# Delete lines
sed '5d' file.txt             # delete line 5
sed '10,20d' file.txt         # delete lines 10-20
sed '/^#/d' file.txt          # delete comment lines
sed '/^$/d' file.txt          # delete empty lines
sed '/pattern/d' file.txt     # delete lines matching pattern

# Insert text before a line
sed '3i\New line of text' file.txt

# Append text after a line
sed '3a\New line of text' file.txt

# Insert before a pattern
sed '/pattern/i\Inserted line' file.txt

# Replace entire line matching pattern
sed '/pattern/c\Replacement line' file.txt

# Print only matching lines (like grep)
sed -n '/pattern/p' file.txt

# Print specific lines
sed -n '5p' file.txt          # line 5 only
sed -n '10,20p' file.txt      # lines 10-20

# Multiple operations
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Use different delimiter (useful with paths)
sed 's|/usr/local|/opt|g' file.txt
sed 's#http://#https://#g' file.txt

# Capture groups
sed 's/\(.*\)=\(.*\)/\2=\1/' file.txt       # swap around =
sed -E 's/(.*)=(.*)/\2=\1/' file.txt         # extended regex

# Add text to beginning/end of lines
sed 's/^/PREFIX: /' file.txt
sed 's/$/ # comment/' file.txt

# Remove leading/trailing whitespace
sed 's/^[[:space:]]*//' file.txt
sed 's/[[:space:]]*$//' file.txt
sed 's/^[[:space:]]*//;s/[[:space:]]*$//' file.txt

# Transliterate (y command)
sed 'y/abc/ABC/' file.txt
```

## awk

```bash
# Print specific columns
awk '{print $1}' file.txt              # first column
awk '{print $1, $3}' file.txt          # first and third
awk '{print $NF}' file.txt             # last column
awk '{print $(NF-1)}' file.txt         # second to last

# Custom field separator
awk -F: '{print $1, $3}' /etc/passwd
awk -F',' '{print $2}' data.csv

# Custom output separator
awk -F: -v OFS='\t' '{print $1, $3, $7}' /etc/passwd

# Filter by pattern
awk '/error/' file.txt                 # lines containing "error"
awk '!/^#/' file.txt                   # lines not starting with #
awk '$3 > 100' file.txt                # third column greater than 100
awk '$1 == "root"' /etc/passwd         # first field equals "root"
awk -F: '$3 >= 1000' /etc/passwd       # UID >= 1000

# Line numbers
awk '{print NR, $0}' file.txt

# Count lines
awk 'END {print NR}' file.txt

# Sum a column
awk '{sum += $3} END {print sum}' data.txt

# Average
awk '{sum += $1; n++} END {print sum/n}' numbers.txt

# Min/max
awk 'NR==1 || $3 > max {max=$3} END {print max}' data.txt

# Print lines longer than N characters
awk 'length > 80' file.txt

# Print between two patterns
awk '/START/,/END/' file.txt

# Replace field values
awk -F: -v OFS=':' '$3 == 0 {$1 = "SUPERUSER"} {print}' /etc/passwd

# Formatted output
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd

# Multiple rules
awk '/error/ {errors++} /warning/ {warnings++} END {print errors, warnings}' log.txt

# Associative arrays (frequency count)
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log | sort -rn

# Process specific lines
awk 'NR==1 {print "Header:", $0} NR>1 {print "Data:", $0}' file.txt

# Skip header row
awk 'NR > 1 {print $2}' data.tsv

# Built-in variables
# NR  = current line number (across all files)
# FNR = current line number (in current file)
# NF  = number of fields in current line
# FS  = input field separator
# OFS = output field separator
# RS  = input record separator
# ORS = output record separator
```

## cut

```bash
# Cut by character position
cut -c1-10 file.txt            # characters 1-10
cut -c5- file.txt              # character 5 to end
cut -c-20 file.txt             # first 20 characters

# Cut by field with delimiter
cut -d: -f1 /etc/passwd        # first field
cut -d: -f1,3,7 /etc/passwd   # fields 1, 3, and 7
cut -d',' -f2-4 data.csv      # fields 2 through 4

# Cut by bytes
cut -b1-16 file.bin

# Complement (everything except specified)
cut -d: --complement -f2 /etc/passwd

# Change output delimiter
cut -d: -f1,7 --output-delimiter=$'\t' /etc/passwd
```

## sort

```bash
# Basic alphabetical sort
sort file.txt

# Reverse sort
sort -r file.txt

# Numeric sort
sort -n numbers.txt

# Human-readable numeric sort (1K, 2M, 3G)
sort -h sizes.txt

# Sort by specific column (field)
sort -k2 file.txt              # sort by 2nd column
sort -k2,2n file.txt           # numeric sort by 2nd column only
sort -t: -k3,3n /etc/passwd   # sort by UID

# Sort by multiple keys
sort -k1,1 -k2,2n file.txt    # alphabetical by col1, then numeric by col2

# Remove duplicates while sorting
sort -u file.txt

# Case-insensitive sort
sort -f file.txt

# Check if already sorted
sort -c file.txt

# Stable sort (preserve original order for equal elements)
sort -s -k2,2 file.txt

# Sort by month name
sort -M dates.txt

# Random shuffle
sort -R file.txt

# Sort IP addresses
sort -t. -k1,1n -k2,2n -k3,3n -k4,4n ips.txt

# Sort with version numbers
sort -V versions.txt
```

## uniq

**Note:** `uniq` only detects adjacent duplicates — always `sort` first.

```bash
# Remove adjacent duplicates
sort file.txt | uniq

# Count occurrences
sort file.txt | uniq -c

# Show only duplicated lines
sort file.txt | uniq -d

# Show only unique lines (appear exactly once)
sort file.txt | uniq -u

# Ignore case
sort file.txt | uniq -i

# Skip first N fields when comparing
sort file.txt | uniq -f 1

# Skip first N characters when comparing
sort file.txt | uniq -s 5

# Top 10 most frequent lines
sort file.txt | uniq -c | sort -rn | head -10
```

## tr (Translate/Delete Characters)

```bash
# Replace characters
echo "hello" | tr 'a-z' 'A-Z'       # lowercase to uppercase
echo "HELLO" | tr 'A-Z' 'a-z'       # uppercase to lowercase

# Replace specific characters
echo "hello world" | tr ' ' '_'
cat file.txt | tr '\t' ' '           # tabs to spaces

# Squeeze repeated characters
echo "aabbcc" | tr -s 'a-z'          # abc
echo "too   many   spaces" | tr -s ' '

# Delete characters
echo "hello 123 world" | tr -d '0-9'
echo "hello world" | tr -d ' '

# Delete everything except (complement + delete)
echo "hello 123 world" | tr -cd '0-9\n'    # keep only digits + newlines

# Replace non-printable characters
cat file.bin | tr -cd '[:print:]\n'

# DOS to Unix line endings
tr -d '\r' < dosfile.txt > unixfile.txt

# Character classes
# [:alnum:]  letters and digits
# [:alpha:]  letters
# [:digit:]  digits
# [:lower:]  lowercase letters
# [:upper:]  uppercase letters
# [:space:]  whitespace
# [:print:]  printable characters
# [:punct:]  punctuation
```

## xargs

```bash
# Basic: pass stdin as arguments
find . -name "*.txt" | xargs wc -l

# Handle filenames with spaces (null delimiter)
find . -name "*.txt" -print0 | xargs -0 wc -l

# Limit arguments per command
echo "1 2 3 4 5" | xargs -n 2 echo

# Parallel execution
find . -name "*.png" -print0 | xargs -0 -P 4 -n 1 optipng

# Placeholder for argument position
ls *.tar.gz | xargs -I {} tar xzf {} -C /tmp/extracted/

# Prompt before each execution
find . -name "*.bak" | xargs -p rm

# Run command for each line (not word)
cat urls.txt | xargs -I {} curl -O {}

# Combine with grep output
grep -rl "TODO" src/ | xargs sed -i 's/TODO/DONE/g'

# Dry run (show commands without executing)
find . -name "*.log" | xargs -t echo rm
```

## column

```bash
# Format into columns
mount | column -t

# Specify delimiter for table mode
cat /etc/passwd | column -t -s ':'

# Create columns from a list
echo -e "one\ntwo\nthree\nfour\nfive\nsix" | column

# JSON pretty-print (util-linux 2.37+)
echo '{"a":1,"b":2}' | column --json

# Custom output separator
column -t -s ',' -o ' | ' data.csv
```

## paste

```bash
# Merge lines side by side
paste file1.txt file2.txt

# Custom delimiter
paste -d ',' file1.txt file2.txt

# Join all lines into one (serial mode)
paste -s file.txt

# Join all lines with a custom delimiter
paste -sd ',' file.txt

# Convert single column to multiple columns
paste - - - < file.txt              # 3 columns
cat file.txt | paste -d ',' - -     # 2 columns, comma-separated

# Combine with cut for column rearrangement
paste <(cut -f3 data.tsv) <(cut -f1 data.tsv)
```

## wc (Word Count)

```bash
# Count lines, words, characters
wc file.txt

# Lines only
wc -l file.txt

# Words only
wc -w file.txt

# Characters only
wc -m file.txt

# Bytes only
wc -c file.txt

# Longest line length
wc -L file.txt

# Count files
ls | wc -l
fd -t f | wc -l

# Count lines across multiple files
wc -l src/*.py

# Count with find
find . -name "*.py" -exec cat {} + | wc -l
```

## tee

```bash
# Write to file AND stdout
ls -la | tee listing.txt

# Append instead of overwrite
echo "new entry" | tee -a log.txt

# Write to multiple files
echo "hello" | tee file1.txt file2.txt file3.txt

# Use in pipelines (inspect intermediate output)
cat data.txt | sort | tee sorted.txt | uniq -c | tee counts.txt | sort -rn | head

# Combine with sudo (write to protected file)
echo "new line" | sudo tee -a /etc/hosts > /dev/null
```

## Combining Tools — Common Patterns

```bash
# Frequency analysis of HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Extract and sort unique IPs
awk '{print $1}' access.log | sort -u

# CSV column statistics
awk -F',' '{print $3}' data.csv | sort -n | tail -1    # max of column 3

# Find and replace across multiple files
grep -rl "oldtext" src/ | xargs sed -i 's/oldtext/newtext/g'

# Remove duplicate lines preserving order (no sort needed)
awk '!seen[$0]++' file.txt

# Extract key=value pairs
grep -oP '(?<=key=)\w+' config.txt

# Transpose rows and columns
awk '{for(i=1;i<=NF;i++) a[i]=a[i]" "$i} END {for(i in a) print a[i]}' data.txt

# Generate numbered list
cat -n file.txt
awk '{printf "%3d. %s\n", NR, $0}' file.txt

# Join lines matching a pattern with the next line
sed '/pattern/{N;s/\n/ /}' file.txt

# Print every Nth line
awk 'NR % 5 == 0' file.txt

# Side-by-side diff of sorted files
diff <(sort file1.txt) <(sort file2.txt)
```
