---
layout: post
title: "tmux: The Terminal Multiplexer Every Linux Admin Should Master"
date: Jan-07-2012
categories: [Linux, DevOps]
tags: [linux, tmux, terminal, productivity, ssh, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

A dropped SSH session should never cost you a running build, a half-finished migration, or a long `rsync`. `tmux` keeps your work alive on the server no matter what happens to your connection, and along the way it turns one terminal into as many panes and windows as you need.

## Why tmux and Not Just More Terminal Tabs

Local terminal tabs die with your SSH connection, and `nohup` only saves a single command. `tmux` runs a server process on the remote host that owns your shells, so you can detach, close your laptop, reconnect from another machine, and pick up exactly where you left off. It also gives you split panes, named windows, scrollback, and copy mode without needing a GUI.

```bash
# Install on Debian/Ubuntu
sudo apt install tmux

# Install on RHEL/CentOS/Fedora
sudo dnf install tmux

# Check the version
tmux -V
```

*Tip:* Everything in tmux is driven by a prefix key, `Ctrl+b` by default. Every shortcut below means "press `Ctrl+b`, release, then press the next key".

## Starting, Naming, and Reattaching Sessions

A session is the top-level container. Name your sessions so you can tell them apart when you have several running.

```bash
# Start a new named session
tmux new -s deploy

# Start detached, useful in scripts
tmux new -d -s backup

# List all sessions
tmux ls

# Reattach to a session by name
tmux attach -t deploy

# Attach to the most recent session
tmux attach
```

Inside a session, `Ctrl+b d` detaches and leaves everything running. `Ctrl+b $` renames the current session, and `Ctrl+b s` opens an interactive session picker.

*Tip:* `tmux attach -t deploy || tmux new -s deploy` is a handy one-liner for your shell profile. It attaches if the session exists and creates it otherwise.

## Managing Windows Inside a Session

Windows are like tabs inside a session, each holding its own shell or program.

```bash
# Create a new window
Ctrl+b c

# Rename the current window
Ctrl+b ,

# Move to the next or previous window
Ctrl+b n
Ctrl+b p

# Jump directly to window 3
Ctrl+b 3

# List and pick a window interactively
Ctrl+b w

# Close the current window
Ctrl+b &
```

*Tip:* Set a base index of 1 so window numbers match your keyboard row with `set -g base-index 1` in `~/.tmux.conf`.

## Splitting Panes for Side-by-Side Work

Panes split a single window, which is where tmux really shines for troubleshooting: tail a log in one pane while running commands in another.

```bash
# Split vertically (side by side)
Ctrl+b %

# Split horizontally (stacked)
Ctrl+b "

# Move between panes with the arrow keys
Ctrl+b <arrow>

# Cycle through panes
Ctrl+b o

# Resize the current pane
Ctrl+b Ctrl+<arrow>

# Zoom the current pane to full window, press again to unzoom
Ctrl+b z

# Close the current pane
Ctrl+b x
```

*Tip:* `Ctrl+b z` is the single most useful shortcut when a pane gets cramped. It toggles a full-screen zoom without destroying your layout.

## Scrolling Back and Copying Text

tmux has its own scrollback buffer, so your terminal's scroll wheel may not behave as expected until you enter copy mode.

```bash
# Enter copy mode
Ctrl+b [

# Scroll up and down inside copy mode
Up / Down or PageUp / PageDown

# Start a selection, move, then copy
Space, then Enter

# Paste the copied buffer
Ctrl+b ]

# Search inside copy mode
Ctrl+b [ then Ctrl+s
```

*Tip:* Add `set -g mouse on` to `~/.tmux.conf` to enable mouse scrolling, pane selection, and resizing. It makes tmux feel far more natural if you are coming from a GUI terminal.

## Detaching, Killing, and Cleaning Up Sessions

Knowing how to get rid of sessions is as important as creating them, especially on shared hosts where stale sessions pile up.

```bash
# Detach from inside a session
Ctrl+b d

# Kill a specific session
tmux kill-session -t deploy

# Kill every session except the current one
tmux kill-session -a

# Kill the whole tmux server and all sessions
tmux kill-server
```

*Tip:* `tmux kill-session -a` is the safe cleanup command. It leaves you attached to the session you are in and removes everything else.

## Sharing a Session with Another User

tmux can attach multiple clients to the same session, which makes pair debugging over SSH genuinely practical.

```bash
# Create a shared session as the owner
tmux new -s pair

# Another user on the same host attaches to it
tmux attach -t pair

# Force a read-only attach for a viewer
tmux attach -t pair -r
```

*Tip:* For real multi-user sharing, set the socket permissions with `tmux -S /tmp/shared.sock new -s pair` and `chmod 777 /tmp/shared.sock`, then have the other user attach with `tmux -S /tmp/shared.sock attach -t pair`.

## A Practical tmux.conf to Start From

A small config file removes most of the friction. Drop this into `~/.tmux.conf` and reload with `Ctrl+b :` followed by `source-file ~/.tmux.conf`.

```bash
# Use a more comfortable prefix
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# Start windows and panes at 1
set -g base-index 1
setw -g pane-base-index 1

# Enable mouse support
set -g mouse on

# Larger scrollback buffer
set -g history-limit 10000

# Faster key repetition
set -sg escape-time 0

# Reload config with prefix + r
bind r source-file ~/.tmux.conf \; display "Config reloaded"
```

*Tip:* Keep your config in version control alongside your dotfiles. A tmux setup you can reproduce on any new server in seconds is worth far more than one you have to rebuild by hand.

## Bonus: Running tmux Non-Interactively in Scripts

Because tmux can create detached sessions and send keys, it doubles as a lightweight job runner for long tasks.

```bash
# Start a detached session running a long backup
tmux new -d -s backup 'rsync -av /data/ /mnt/backup/'

# Send a command into an existing session
tmux send-keys -t deploy 'git pull' Enter

# Capture the last 50 lines of a pane, useful for logging
tmux capture-pane -p -t deploy -S -50
```

*Tip:* `tmux capture-pane` is a neat way to snapshot what a long-running process printed without redirecting its output to a file in the first place.
