# Compression & Archives

## tar

```bash
# Create archive
tar cf archive.tar file1 file2 dir/

# Create with gzip compression
tar czf archive.tar.gz dir/

# Create with xz compression
tar cJf archive.tar.xz dir/

# Create with zstd compression
tar --zstd -cf archive.tar.zst dir/

# Create with bzip2 compression
tar cjf archive.tar.bz2 dir/

# Extract archive (auto-detects compression)
tar xf archive.tar.gz

# Extract to a specific directory
tar xf archive.tar.gz -C /target/dir/

# List contents without extracting
tar tf archive.tar.gz

# Verbose output
tar xvf archive.tar.gz

# Extract specific files
tar xf archive.tar.gz path/to/file.txt

# Exclude patterns
tar czf archive.tar.gz --exclude='*.log' --exclude='.git' dir/

# Exclude from file
tar czf archive.tar.gz --exclude-from=exclude.txt dir/

# Preserve permissions and ownership
tar czpf archive.tar.gz dir/

# Append files to existing (uncompressed) archive
tar rf archive.tar newfile.txt

# Diff archive against filesystem
tar df archive.tar

# Show progress with pv
tar cf - dir/ | pv | gzip > archive.tar.gz

# Create incremental backup
tar czf backup-full.tar.gz --listed-incremental=snapshot.snar dir/
tar czf backup-incr.tar.gz --listed-incremental=snapshot.snar dir/

# Common flags:
# c  create       x  extract       t  list
# z  gzip         J  xz            j  bzip2
# f  file         v  verbose       p  preserve permissions
# C  change dir   r  append        --exclude  skip pattern
```

## gzip / gunzip

```bash
# Compress (replaces original file)
gzip file.txt                  # creates file.txt.gz

# Keep original file
gzip -k file.txt

# Decompress
gunzip file.txt.gz
gzip -d file.txt.gz

# Compress with level (1=fast, 9=best compression)
gzip -9 file.txt

# Compress to stdout
gzip -c file.txt > file.txt.gz

# View compressed file contents
zcat file.txt.gz
zless file.txt.gz
zgrep "pattern" file.txt.gz

# Compress all files in current directory
gzip *.txt

# Show compression ratio
gzip -l file.txt.gz

# Test integrity
gzip -t file.txt.gz
```

## xz

```bash
# Compress
xz file.txt                   # creates file.txt.xz

# Keep original
xz -k file.txt

# Decompress
xz -d file.txt.xz
unxz file.txt.xz

# Compression level (0-9, default 6)
xz -9 file.txt

# Extreme mode (slower, slightly better)
xz -9e file.txt

# Multi-threaded compression
xz -T0 file.txt               # use all cores

# Compress to stdout
xz -c file.txt > file.txt.xz

# View compressed file
xzcat file.txt.xz
xzless file.txt.xz
xzgrep "pattern" file.txt.xz

# Show info
xz -l file.txt.xz

# Test integrity
xz -t file.txt.xz
```

## zstd (Zstandard)

**Install:** `pacman -S zstd`

```bash
# Compress
zstd file.txt                 # creates file.txt.zst

# Keep original
zstd -k file.txt

# Decompress
zstd -d file.txt.zst
unzstd file.txt.zst

# Compression level (1-19, default 3; ultra: 20-22)
zstd -19 file.txt
zstd --ultra -22 file.txt

# Multi-threaded
zstd -T0 file.txt             # use all cores

# Compress to stdout
zstd -c file.txt > file.txt.zst

# Adapt compression level based on I/O speed
zstd --adapt file.txt

# Show info
zstd -l file.txt.zst

# Test integrity
zstd -t file.txt.zst

# Train a dictionary (for many similar small files)
zstd --train /path/to/samples/* -o dict.zstd
zstd -D dict.zstd file.txt
```

## zip / unzip

```bash
# Create zip archive
zip archive.zip file1.txt file2.txt

# Create recursively (include directories)
zip -r archive.zip dir/

# Add password
zip -e archive.zip files

# Exclude patterns
zip -r archive.zip dir/ -x "*.log" "*.tmp"

# Update existing archive (add new/changed files)
zip -u archive.zip newfile.txt

# Compress with best ratio
zip -9 -r archive.zip dir/

# Extract
unzip archive.zip

# Extract to directory
unzip archive.zip -d /target/dir/

# List contents
unzip -l archive.zip

# Extract specific files
unzip archive.zip path/to/file.txt

# Test integrity
unzip -t archive.zip

# Overwrite without prompting
unzip -o archive.zip

# Show verbose info
unzip -v archive.zip
```

## 7z

**Install:** `pacman -S p7zip`

```bash
# Create 7z archive
7z a archive.7z dir/

# Create with maximum compression
7z a -mx=9 archive.7z dir/

# Create with password and encryption
7z a -p -mhe=on archive.7z dir/

# Create zip format
7z a -tzip archive.zip dir/

# Extract
7z x archive.7z

# Extract to directory
7z x archive.7z -o/target/dir/

# List contents
7z l archive.7z

# Test integrity
7z t archive.7z

# Extract (flat — no directory structure)
7z e archive.7z

# Update archive
7z u archive.7z newfile.txt

# Delete from archive
7z d archive.7z file.txt

# Split archive into volumes
7z a -v100m archive.7z dir/

# Multi-threaded
7z a -mmt=on archive.7z dir/
```

## Compression Comparison

| Format | Speed | Ratio | Multi-thread | Notes |
|--------|-------|-------|-------------|-------|
| **gzip** | Fast | Good | No (pigz does) | Most compatible, universal |
| **bzip2** | Slow | Better | No (pbzip2 does) | Legacy, largely superseded |
| **xz** | Slow | Best | Yes (-T) | Great ratio, used by pacman |
| **zstd** | Very fast | Very good | Yes (-T) | Best speed/ratio tradeoff |
| **zip** | Fast | Good | No | Cross-platform, Windows compat |
| **7z** | Slow | Best | Yes | Best compression, feature-rich |

```bash
# Quick benchmark: compress the same file with each
time gzip -k -9 testfile
time xz -k -9 testfile
time zstd -k -19 testfile

# Compare sizes
ls -lh testfile testfile.gz testfile.xz testfile.zst
```

## Parallel Compression

```bash
# pigz — parallel gzip
pigz file.txt                  # gzip-compatible, multi-threaded
pigz -p 4 file.txt            # use 4 cores
unpigz file.txt.gz

# pbzip2 — parallel bzip2
pbzip2 file.txt
pbzip2 -d file.txt.bz2

# Use with tar
tar cf - dir/ | pigz > archive.tar.gz
tar cf - dir/ | zstd -T0 > archive.tar.zst
```
