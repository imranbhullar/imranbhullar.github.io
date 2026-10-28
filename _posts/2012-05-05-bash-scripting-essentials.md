---
layout: post
title: "Bash Scripting Essentials: Variables, Loops, and Safe Defaults"
date: May-05-2012
categories: [Linux, DevOps]
tags: [linux, bash, scripting, automation, devops]
image: /assets/img/Linux_unix_bsd.png
---

Bash is the glue of Linux administration. A well-written script is a small, reliable program; a badly-written one is a footgun that deletes the wrong directory at 3 a.m. This post covers the patterns that make scripts safe, readable, and maintainable.

## The Shebang and Safe Defaults

Every script should start with a shebang and a set of defensive options.

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

* `-e` — exit on error.
* `-u` — error on undefined variables.
* `-o pipefail` — a pipeline fails if any command in it fails.
* `IFS` — restrict word splitting to newlines and tabs.

*Tip:* `set -euo pipefail` catches the vast majority of silent failures. It is the single most important line in any script.

## Variables and Quoting

```bash
# Assign
name="alice"

# Use
echo "Hello, ${name}"

# Default value
echo "${name:-unknown}"

# Error if unset
echo "${name:?name is required}"

# Command substitution
today=$(date +%F)

# Arithmetic
count=$((count + 1))
```

*Tip:* Always quote variables: `"${name}"`, not `$name`. Unquoted variables split on whitespace and expand globs, which is the source of most "my script broke on a filename with a space" bugs.

## Conditionals

```bash
# String comparison
if [[ "${name}" == "alice" ]]; then
  echo "hello alice"
fi

# Numeric comparison
if (( count > 10 )); then
  echo "many"
fi

# File tests
if [[ -f /etc/nginx/nginx.conf ]]; then
  echo "config exists"
fi

if [[ -d /var/log ]]; then
  echo "directory exists"
fi

# Command success
if command -v jq >/dev/null 2>&1; then
  echo "jq is installed"
fi
```

*Tip:* Use `[[ ]]` (bash) rather than `[ ]` (POSIX). It handles empty variables and word splitting correctly without extra quoting.

## Loops

```bash
# Iterate over a list
for user in alice bob carol; do
  echo "user: ${user}"
done

# Iterate over files
for file in /var/log/*.log; do
  echo "processing ${file}"
done

# While loop
while read -r line; do
  echo "${line}"
done < input.txt

# C-style loop
for ((i = 0; i < 10; i++)); do
  echo "${i}"
done
```

*Tip:* `while read -r line` is the correct way to read a file line by line. `for line in $(cat file)` breaks on whitespace and is a common bug.

## Functions

```bash
log() {
  local level="$1"
  shift
  echo "[$(date +%F' '%T)] [${level}] $*" >&2
}

log INFO "starting backup"
log ERROR "backup failed"
```

*Tip:* Use `local` for function variables. Without it, every variable is global, and a function can silently overwrite a caller's variable.

## Error Handling and Traps

```bash
cleanup() {
  rm -f /tmp/workfile
}

trap cleanup EXIT

# Trap specific signals
trap 'echo "interrupted"; exit 130' INT TERM
```

*Tip:* `trap cleanup EXIT` runs on every exit path, including errors and signals. It is the cleanest way to guarantee temporary files are removed.

## Argument Parsing

```bash
while [[ $# -gt 0 ]]; do
  case "$1" in
    -f|--file)
      file="$2"
      shift 2
      ;;
    -v|--verbose)
      verbose=1
      shift
      ;;
    *)
      echo "unknown option: $1" >&2
      exit 1
      ;;
  esac
done
```

*Tip:* For anything more than a couple of options, use `getopts` or a dedicated parser. Hand-rolled argument parsing is where most scripts grow bugs.

## Logging and Output

```bash
# Log to stderr, keep stdout clean for data
echo "error: something failed" >&2

# Redirect both
command >out.log 2>&1

# Discard output
command >/dev/null 2>&1

# Tee to a file and stdout
command | tee -a run.log
```

*Tip:* Send logs to stderr and data to stdout. It lets callers pipe your script's output without mixing in log lines.

## Arrays

```bash
# Indexed array
users=(alice bob carol)
echo "${users[0]}"
echo "${users[@]}"
echo "${#users[@]}"

# Append
users+=(dave)

# Iterate
for u in "${users[@]}"; do
  echo "${u}"
done

# Associative array (bash 4+)
declare -A ports
ports[http]=80
ports[https]=443
for k in "${!ports[@]}"; do
  echo "${k}=${ports[$k]}"
done
```

*Tip:* Always quote `"${array[@]}"`. Unquoted, it splits on whitespace and expands globs, which is the same bug as unquoted variables.

## getopts: Proper Option Parsing

```bash
while getopts ":f:v" opt; do
  case "${opt}" in
    f) file="${OPTARG}" ;;
    v) verbose=1 ;;
    \?) echo "invalid option: -${OPTARG}" >&2; exit 1 ;;
    :) echo "option -${OPTARG} requires an argument" >&2; exit 1 ;;
  esac
done
shift $((OPTIND - 1))
```

*Tip:* `getopts` handles short options only. For long options, use `getopt` (the external command) or a dedicated parser.

## trap ERR and set -E

```bash
set -Eeuo pipefail

trap 'echo "error on line ${LINENO}: ${BASH_COMMAND}" >&2' ERR

trap 'echo "interrupted" >&2; exit 130' INT TERM
```

*Tip:* `set -E` makes the `ERR` trap inherited by functions and subshells. Without it, an error inside a function does not trigger the trap.

## BASH_REMATCH and printf -v

```bash
# Capture groups with BASH_REMATCH
if [[ "v1.2.3" =~ ^v([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
  major="${BASH_REMATCH[1]}"
  minor="${BASH_REMATCH[2]}"
  patch="${BASH_REMATCH[3]}"
fi

# Assign to a variable without a subshell
printf -v today '%(%Y-%m-%d)T' -1
echo "${today}"
```

*Tip:* `printf -v` is faster than `var=$(command)` because it avoids a subshell. It is the correct way to format dates and numbers in a loop.

## Quick Reference

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

log() { echo "[$(date +%T)] $*" >&2; }

main() {
  log "starting"
  # ...
}

main "$@"
```

*Tip:* Wrap the body in a `main` function and call it at the end. It keeps variables local, makes the script testable, and prevents partial execution if the file is edited while running.
