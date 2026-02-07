---
layout: post
title: "Linux Archiving & Compression: From Tar to Cpio"
date: 2026-02-07
categories: [Linux, SysAdmin]
tags: [LFCS, bash, archiving, backup, devops]
---
![nix](/assets/img/Linux_unix_bsd.png)

Efficiently managing files is a cornerstone of Linux administration. Whether you are backing up a configuration directory like `/etc` or preparing files for a deployment, understanding the nuances of `tar`, `cpio`, and various compression algorithms is essential.

## 1. The Foundation: Tape Archives (tar)

The `tar` (Tape ArChive) command is the most common utility for collecting multiple files into a single archive. By default, `tar` archives are **not** compressed; they simply group files together. You might notice a slightly smaller size than the source due to efficient block allocation, but no actual compression algorithm is applied.

* **-c**: Create a new archive.
* **-v**: Verbose (see the files being added).
* **-f**: Specify the filename.
```bash
tar -cvf etc.tar /etc
#Decompress (Extract) -x: Extract files from an archive.

tar -xvf etc.tar
```

### Standard Compression

```bash
#-z: Filter the archive through gzip.
tar -czvf etc.tar.gz /etc
#Decompress via tar
tar -xzvf etc.tar.gz

```
### High Efficiency
```bash
#Compress via tar -j: Filter the archive through bzip2.
tar -cjvf etc.tar.bz2 /etc
# Decompress via tar
tar -xjvf etc.tar.bz2
```
### Maximum Compression
```bash
#Compress via tar -J: Filter the archive through xz.
tar -cJvf etc.tar.xz /etc
# Decompress via tar
tar -xJvf etc.tar.xz
```


## 2. gzip Command

`gzip` compresses files using the **DEFLATE** algorithm and appends a `.gz` extension.

### Compress a File

```bash
gzip file.txt

# Decompress a File
gunzip file.txt.gz
#or
gzip -d file.txt.gz
#Keep Original File While Compressing
gzip -k file.txt

#Compress Multiple Files
gzip file1.log file2.log file3.log
#Compress Recursively
gzip -r /var/log/myapp/
#View Compressed File Contents
zcat file.txt.gz
#or
zless file.txt.gz
```
gzip Compression Levels
```bash
gzip -1 file.txt   # fastest, less compression
gzip -9 file.txt   # best compression

```

## 3.xz Command
`xz` provides very high compression using the LZMA2 algorithm. It is commonly used for kernel sources, backups, and long-term storage.
```bash
#Compress a File
xz file.txt

#Decompress a File
unxz file.txt.xz
#or
xz -d file.txt.xz
# Keep Original File While Compressing
xz -k file.txt
# Compress with Maximum Compression
xz -9 file.txt
# Compress Using Multiple CPU Cores, this will automatically use all available cores
xz -T0 file.txt
#View Contents Without Extracting
xzcat file.txt.xz
#or
xzless file.txt.xz
```

| Feature           | gzip       | xz               |
| ----------------- | ---------- | ---------------- |
| Compression Speed | Fast       | Slower           |
| Compression Ratio | Medium     | Very High        |
| CPU Usage         | Low        | High             |
| Common Usage      | Logs, temp | Archives, source |
| File Extension    | `.gz`      | `.xz`            |
