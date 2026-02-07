---
layout: post
title: "The Linux Survival Guide: How to Get Help from the Command Line"
date: 2025-10-12 13:40:00 +0500
categories: [Linux, Tutorial]
tags: [Command Line, SysAdmin, Troubleshooting, Documentation]
description: "Don't get stuck in the terminal. Learn the essential commands to find documentation, manuals, and command usage directly from the Linux CLI."
---
![nix](/assets/img/Linux_unix_bsd.png)

In the world of Linux and Unix, you don't need to memorize every flag and argument. You just need to know how to find them. Whether you are troubleshooting a production server or building a new Terraform module, these built-in help systems are your best friends.

## 1. The Manual Pages (`man`)
The `man` command is the gold standard for Linux documentation. Most programs install a manual page that describes their purpose, syntax, and options.

* **Usage:** `man <command>`
* **Example:** `man tar`

> **Pro Tip:** Use `/search_term` while inside a man page to find specific keywords quickly. Press `n` to jump to the next result.



## 2. The `help` Built-in
Not every command is a standalone program; some are built directly into your shell (like `cd`, `alias`, or `export`). For these, `man` might not give you what you need.

* **Usage:** `help <command>`
* **Example:** `help cd`

## 3. The `--help` Flag
If you just need a quick syntax reminder without scrolling through a full manual, most modern CLI tools support the `--help` flag.

* **Example:** `grep --help`

## 4. `whatis` and `apropos`
* **`whatis`**: Gives you a one-line description of what a command does.
* **`apropos`**: Searches the manual page descriptions for a specific keyword. This is perfect when you know *what* you want to do, but not *which* command does it.

* **Example:** `apropos "partition"` (will list all commands related to disk partitioning).

## 5. GNU Info Pages (`info`)
For more complex GNU tools (like `gcc` or `sed`), the `info` system provides a more detailed, hyperlinked manual than `man`.

* **Usage:** `info <command>`

## 6. Understanding Manual Sections (Section 5)
The manual is divided into sections. By default, `man` shows you Section 1 (Executable programs). However, as a SysAdmin, you often need **Section 5**, which covers **File Formats and Configurations**.

If you run `man passwd`, you get help for the *command* to change a password. If you run `man 5 passwd`, you get the documentation for the *actual file structure* of `/etc/passwd`.

* **Usage:** `man <section_number> <topic>`
* **Example:** `man 5 shadow` or `man 5 exports`



### Common Sections Reference:
1. **Section 1:** User Commands (e.g., `ls`, `mkdir`)
2. **Section 5:** File Formats (e.g., `/etc/hosts`, config files)
3. **Section 8:** System Administration commands (e.g., `iptables`, `fdisk`)
---



### Summary Table

| Tool | Best Used For... |
| :--- | :--- |
| **man** | Full, detailed official documentation. |
| **help** | Shell built-in commands (Bash/Zsh). |
| **--help** | Quick syntax and flag reference. |
| **apropos** | Finding a command based on a keyword. |
| **info** | Deep-dives into GNU software. |

