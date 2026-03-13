# Compression & Archives

Creating and extracting archives with `tar`, `gzip`, `xz`, `zstd`, `zip`, and `7z`. Includes a speed/ratio comparison table and parallel compression tips.

## tar

`tar` (tape archive) bundles multiple files into a single archive file. By itself it doesn't compress — compression is added via flags (`z` for gzip, `J` for xz, `j` for bzip2) or `--zstd`. Modern `tar` auto-detects compression format when extracting.

### Creating Archives

```bash
# Create uncompressed archive
tar cf archive.tar file1 file2 dir/

# Create with gzip compression (.tar.gz or .tgz)
tar czf archive.tar.gz dir/

# Create with xz compression (.tar.xz — best ratio)
tar cJf archive.tar.xz dir/

# Create with zstd compression (.tar.zst — best speed/ratio tradeoff)
tar --zstd -cf archive.tar.zst dir/

# Create with bzip2 compression (.tar.bz2)
tar cjf archive.tar.bz2 dir/

# Verbose output (show files as they're archived)
tar czvf archive.tar.gz dir/

# Preserve permissions and ownership (for backups)
tar czpf archive.tar.gz dir/
```

### Extracting Archives

```bash
# Extract archive (auto-detects compression)
tar xf archive.tar.gz

# Extract to a specific directory
tar xf archive.tar.gz -C /target/dir/

# Extract with verbose output
tar xvf archive.tar.gz

# Extract specific files only
tar xf archive.tar.gz path/to/file.txt
```

### Inspecting Archives

```bash
# List contents without extracting
tar tf archive.tar.gz

# Diff archive against filesystem (show what's changed)
tar df archive.tar
```

### Excluding and Filtering

```bash
# Exclude patterns
tar czf archive.tar.gz --exclude='*.log' --exclude='.git' dir/

# Exclude from file
tar czf archive.tar.gz --exclude-from=exclude.txt dir/
```

### Advanced

```bash
# Append files to existing uncompressed archive
tar rf archive.tar newfile.txt

# Show progress with pv (pipe viewer)
tar cf - dir/ | pv | gzip > archive.tar.gz

# Incremental backup (full first, then incremental)
tar czf backup-full.tar.gz --listed-incremental=snapshot.snar dir/
tar czf backup-incr.tar.gz --listed-incremental=snapshot.snar dir/
```

### Flag Reference

| Flag | Meaning |
|------|---------|
| `c` | Create archive |
| `x` | Extract archive |
| `t` | List contents |
| `z` | gzip compression |
| `J` | xz compression |
| `j` | bzip2 compression |
| `f` | Specify filename |
| `v` | Verbose output |
| `p` | Preserve permissions |
| `C` | Change to directory before operating |
| `r` | Append to archive |
| `--exclude` | Skip matching files |

## gzip / gunzip

`gzip` is the most widely compatible compression format. It's fast with decent compression, and virtually every Unix system has it. It compresses single files — use `tar` to bundle directories.

```bash
# Compress (replaces original file with .gz)
gzip file.txt

# Keep original file
gzip -k file.txt

# Decompress
gunzip file.txt.gz
gzip -d file.txt.gz

# Compression level (1=fastest, 9=best compression, default=6)
gzip -9 file.txt

# Compress to stdout (useful for piping)
gzip -c file.txt > file.txt.gz

# View compressed file without extracting
zcat file.txt.gz
zless file.txt.gz
zgrep "pattern" file.txt.gz

# Show compression ratio
gzip -l file.txt.gz

# Test integrity
gzip -t file.txt.gz

# Compress all .txt files in current directory
gzip *.txt
```

## xz

`xz` provides the best compression ratio of the common formats, at the cost of slower speed. It's what pacman uses for Arch packages. Supports multi-threading.

```bash
# Compress (replaces original with .xz)
xz file.txt

# Keep original
xz -k file.txt

# Decompress
xz -d file.txt.xz
unxz file.txt.xz

# Compression level (0-9, default 6)
xz -9 file.txt

# Extreme mode (even slower, slightly better ratio)
xz -9e file.txt

# Multi-threaded compression (0 = all cores)
xz -T0 file.txt

# Compress to stdout
xz -c file.txt > file.txt.xz

# View compressed file
xzcat file.txt.xz
xzless file.txt.xz
xzgrep "pattern" file.txt.xz

# Show compression info
xz -l file.txt.xz

# Test integrity
xz -t file.txt.xz
```

## zstd (Zstandard)

`zstd` offers the best speed-to-ratio tradeoff of any compressor. At default settings it's much faster than gzip with similar or better compression. Supports training dictionaries for compressing many similar small files.

**Install:** `pacman -S zstd`

```bash
# Compress (creates .zst file)
zstd file.txt

# Keep original
zstd -k file.txt

# Decompress
zstd -d file.txt.zst
unzstd file.txt.zst

# Compression level (1-19, default 3; ultra levels 20-22 for max ratio)
zstd -19 file.txt
zstd --ultra -22 file.txt

# Multi-threaded (0 = all cores)
zstd -T0 file.txt

# Compress to stdout
zstd -c file.txt > file.txt.zst

# Adaptive mode (auto-adjusts level based on I/O speed)
zstd --adapt file.txt

# Show compression info
zstd -l file.txt.zst

# Test integrity
zstd -t file.txt.zst

# Train a dictionary (for many similar small files — e.g., JSON logs)
zstd --train /path/to/samples/* -o dict.zstd
zstd -D dict.zstd file.txt
```

## zip / unzip

`zip` is the standard cross-platform archive format. Not the best compression, but universally supported — especially important when sharing files with Windows users.

### Creating Archives

```bash
# Create zip archive
zip archive.zip file1.txt file2.txt

# Include directories recursively
zip -r archive.zip dir/

# Add password protection
zip -e archive.zip files

# Exclude patterns
zip -r archive.zip dir/ -x "*.log" "*.tmp"

# Update existing archive (add new/changed files only)
zip -u archive.zip newfile.txt

# Best compression
zip -9 -r archive.zip dir/
```

### Extracting

```bash
# Extract
unzip archive.zip

# Extract to directory
unzip archive.zip -d /target/dir/

# Extract specific files
unzip archive.zip path/to/file.txt

# Overwrite without prompting
unzip -o archive.zip
```

### Inspecting

```bash
# List contents
unzip -l archive.zip

# Test integrity
unzip -t archive.zip

# Show verbose info
unzip -v archive.zip
```

## 7z

`7z` provides the best compression ratios, supports many formats, and has strong AES-256 encryption. The `-mhe=on` flag also encrypts filenames (not just contents).

**Install:** `pacman -S p7zip`

```bash
# Create 7z archive
7z a archive.7z dir/

# Create with maximum compression
7z a -mx=9 archive.7z dir/

# Create with password and filename encryption
7z a -p -mhe=on archive.7z dir/

# Create zip format (7z can create various formats)
7z a -tzip archive.zip dir/

# Extract (preserves directory structure)
7z x archive.7z

# Extract to directory
7z x archive.7z -o/target/dir/

# Extract flat (no directory structure)
7z e archive.7z

# List contents
7z l archive.7z

# Test integrity
7z t archive.7z

# Update archive
7z u archive.7z newfile.txt

# Delete from archive
7z d archive.7z file.txt

# Split archive into volumes (e.g., 100MB parts)
7z a -v100m archive.7z dir/

# Multi-threaded
7z a -mmt=on archive.7z dir/
```

## Compression Comparison

Typical characteristics at default settings. Actual results vary by data type.

| Format | Speed | Ratio | Multi-thread | Notes |
|--------|-------|-------|-------------|-------|
| **gzip** | Fast | Good | No (use `pigz`) | Most compatible, universal |
| **bzip2** | Slow | Better | No (use `pbzip2`) | Legacy, largely superseded |
| **xz** | Slow | Best | Yes (`-T`) | Great ratio, used by pacman |
| **zstd** | Very fast | Very good | Yes (`-T`) | Best speed/ratio tradeoff |
| **zip** | Fast | Good | No | Cross-platform, Windows compat |
| **7z** | Slow | Best | Yes | Best compression, strong encryption |

```bash
# Quick benchmark: compress the same file with each format
time gzip -k -9 testfile
time xz -k -9 testfile
time zstd -k -19 testfile

# Compare resulting sizes
ls -lh testfile testfile.gz testfile.xz testfile.zst
```

## Parallel Compression

Standard `gzip` and `bzip2` are single-threaded. These drop-in replacements use all cores while producing compatible output.

```bash
# pigz — parallel gzip (produces standard .gz files)
pigz file.txt
pigz -p 4 file.txt            # use 4 cores
unpigz file.txt.gz

# pbzip2 — parallel bzip2
pbzip2 file.txt
pbzip2 -d file.txt.bz2

# Use with tar (pipe through parallel compressor)
tar cf - dir/ | pigz > archive.tar.gz
tar cf - dir/ | zstd -T0 > archive.tar.zst
```
