---
layout: post
title: "Real-Time Terminal Sharing with Tee"
date: 2011-01-01 19:15
categories: [Linux, Tutorials]
tags: [Tips and Tricks, bash, terminal, productivity, devops]
description: "A quick and dirty way to share your live terminal session with others on the same system without complex tools."
---
![nix](/assets/img/Linux_unix_bsd.png)

Sometimes you may want to share what you're doing in the terminal with others in real-time. While tools like `tmux` or `screen` are powerful, there is a simpler way to broadcast your session to a file that others can watch.

## The Broadcast Command
The -f flag tells tail to "follow" the file, meaning it will update the
To start sharing, switch to the shell you want to display and use the following command structure:

```bash
shell_name -i |& tee /tmp/show
```

## How it works:
* **-i**: Runs the shell in interactive mode.
* **|&**: This is a Bash shortcut that pipes both stdout (standard output) and stderr (errors) to the next command.
* **tee**: Reads from the pipe and writes to both the screen and a file simultaneously.


## How Others Can Watch
Once you've started the broadcast, anyone else on the same system (or with access to your `/tmp` directory) can view your terminal activities by typing:
```bash
$ tail -f /tmp/show
```

The -f flag tells tail to "follow" the file, meaning it will update the