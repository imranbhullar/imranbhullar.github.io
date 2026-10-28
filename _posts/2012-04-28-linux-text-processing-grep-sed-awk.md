---
layout: post
title: "Linux Text Processing: grep, sed, awk, cut, sort, and xargs"
date: Apr-28-2012
categories: [Linux, SysAdmin]
tags: [linux, grep, sed, awk, text-processing, bash]
image: /assets/img/Linux_unix_bsd.png
---

The Unix philosophy is small tools that do one thing well, composed with pipes. The text-processing trio — `grep`, `sed`, and `awk` — is the heart of that philosophy, and mastering them turns hours of manual log reading into a one-liner.

## grep: Finding Lines

```bash
# Basic search
grep "error" /var/log/syslog

# Case-insensitive
grep -i "error" /var/log/syslog

# Invert match
grep -v "DEBUG" app.log

# Recursive search
grep -r "TODO" src/

# Show line numbers
grep -n "error" app.log

# Show only the matching part
grep -o "req_id=[a-z0-9]*" app.log

# Count matches
grep -c "error" app.log

# Extended regex
grep -E "error|warn|critical" app.log

# Fixed string, no regex
grep -F "1.2.3.4" access.log
```

*Tip:* `grep -F` is dramatically faster than `grep` when you are searching for a literal string, because it skips regex compilation entirely.

## sed: Stream Editing

```bash
# Replace the first occurrence per line
sed 's/foo/bar/' file.txt

# Replace all occurrences per line
sed 's/foo/bar/g' file.txt

# Edit in place
sed -i 's/foo/bar/g' file.txt

# Edit in place with a backup
sed -i.bak 's/foo/bar/g' file.txt

# Delete lines matching a pattern
sed '/DEBUG/d' app.log

# Print only matching lines
sed -n '/error/p' app.log

# Print a line range
sed -n '10,20p' file.txt

# Insert a line before a match
sed '/^server {/i # managed by ansible' nginx.conf
```

*Tip:* `sed -i` on macOS requires an argument (`sed -i ''`). On Linux it does not. If you write scripts for both, use `sed -i.bak` and delete the backup.

## awk: Field Processing

```bash
# Print the first column
awk '{print $1}' file.txt

# Print specific columns
awk '{print $1, $3}' file.txt

# Custom field separator
awk -F: '{print $1, $7}' /etc/passwd

# Filter by condition
awk '$3 > 100 {print $1, $3}' file.txt

# Sum a column
awk '{sum += $2} END {print sum}' file.txt

# Average a column
awk '{sum += $2; n++} END {print sum/n}' file.txt

# Group and sum
awk '{sum[$1] += $2} END {for (k in sum) print k, sum[k]}' file.txt
```

*Tip:* `awk` is a full programming language. If you find yourself writing a loop in bash to process columns, `awk` is almost always shorter and faster.

## cut, sort, uniq

```bash
# Extract a field by delimiter
cut -d: -f1 /etc/passwd

# Extract a character range
cut -c1-10 file.txt

# Sort lines
sort file.txt

# Sort numerically
sort -n file.txt

# Sort by a specific field
sort -t: -k3 -n /etc/passwd

# Unique lines
uniq file.txt

# Count occurrences
sort file.txt | uniq -c

# Top 10 most frequent
sort file.txt | uniq -c | sort -rn | head
```

*Tip:* `uniq` only removes adjacent duplicates. Always `sort` first, or you will get wrong counts.

## xargs: Building Command Lines

```bash
# Run a command for each line
cat files.txt | xargs rm

# Handle filenames with spaces
find . -name "*.log" -print0 | xargs -0 rm

# Run in parallel
find . -name "*.jpg" -print0 | xargs -0 -P 4 -I {} convert {} {}.png

# Limit arguments per invocation
cat urls.txt | xargs -n 1 curl -O
```

*Tip:* Always use `-print0` with `find` and `-0` with `xargs` when filenames may contain spaces. It is the single most common source of "xargs deleted the wrong file" incidents.

## Composing the Tools

The real power is composition.

```bash
# Top 10 IPs in an access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Count errors per hour
grep "ERROR" app.log | awk '{print $1, $2}' | cut -d: -f1 | sort | uniq -c

# Find the largest files in a directory tree
find . -type f -printf '%s %p\n' | sort -rn | head -20

# Replace a value across many files
grep -rl "old_value" . | xargs sed -i 's/old_value/new_value/g'
```

*Tip:* The last one is the pattern you will use most often: `grep -rl` to find the files, `xargs sed -i` to change them. It is faster and safer than a `for` loop.

## Quick Reference

```bash
# Find lines
grep -i "error" app.log

# Replace text
sed -i 's/old/new/g' file.txt

# Process columns
awk -F: '{print $1, $7}' /etc/passwd

# Top 10 most frequent
sort file.txt | uniq -c | sort -rn | head

# Safe xargs
find . -name "*.log" -print0 | xargs -0 rm
```

*Tip:* Work in order: `grep` to find, `sed` to change, `awk` to compute. Most text-processing tasks are a pipeline of those three, and most bugs are a missing `sort` before `uniq`.
