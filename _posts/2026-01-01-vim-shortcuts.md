---
layout: post
title: "Vim & Sed: Efficient Text Editing and Search-Replace"
date: 2026-01-01
description: "Vim Shortcuts, Mastering Vim navigation, text replacement patterns, and non-interactive editing with sed for the LFCS exam."
categories: [Linux, DevOps]
tags: [LFCS, Vim, Sed, Text Editing, Tips & Tricks]
---
![nix](/assets/img/Linux_unix_bsd.png)

## Advanced Vim Search and Replace

Finding and replacing text efficiently within a file is a key skill for managing configuration files.

* **`/search_term`**: Search forward for a specific string. Press `n` to go to the next occurrence or `N` for the previous.
* **`:s/old/new/`**: Replace the first occurrence of "old" with "new" in the current line.
* **`:s/old/new/g`**: Replace all occurrences (globally) of "old" with "new" on the current line.
* **`:%s/old/new/g`**: Search and replace every occurrence of "old" with "new" in the entire file.
* **`:g/pattern/d`**: Search the entire file and delete all lines that match the specified pattern.

---

## Opening Files at Specific Locations

Instead of just opening a file and scrolling, use these flags to jump directly to your target.

* **`vim +$ ~/hosts`**: Open the file and jump directly to the **last line**.
* **`vim +8 ~/hosts`**: Open the file and jump to ![nix](/assets/img/Linux_unix_bsd.png)
* **`vim +/linux/ ~/hosts`**: Open the file and jump to the first occurrence of the word **"linux"**.
* **`vimtutor`**: Launch the built-in tutorial to practice Vim basics directly in the terminal.

---

## Stream Editing with `sed`

For quick, non-interactive changes, `sed` allows you to modify files without opening them.

* **`sed -i 's/google/cloudflare/' ~/hosts`**: Find the first occurrence of "google" and replace it with "cloudflare" directly in the file.
* **`sed -i '$d' ~/hosts`**: Delete the **last line** of the file.
* **`sed -i '1i "This is a test file"' ~/hosts`**: Insert a new line at the very top (Line 1) of the file.

---
### Vim Shortcuts
## Navigation
* **`gg`**: Jump to the very first line of the file.
* **`G`**: Jump to the very last line of the file.
* **`:<number>`**: Jump to a specific line (e.g., :48).
* **`H`**: Move cursor to the High (top) of the current screen.
* **`L`**: Move cursor to the Low (bottom) of the current screen.
* **`%`**: Jump between matching brackets (), [], or {}.

## Editing & Deletion
* **`i`**: Enter Insert mode at the cursor position.
* **`o`**: Open a new line below the current one and enter Insert mode.
* **`O`**: Open a new line above the current one and enter Insert mode.
* **`dd`**: Delete (cut) the current line.
* **`yy`**: Yank (copy) the current line.
* **`p`**: Put (paste) the yanked or deleted text after the cursor.
* **`u`**: Undo the last action.
* **`.`**: Repeat the last editing command (the "Dot" command).

## Power Moves
* **`ci"`**: Change Inside double quotes (clears text and enters Insert mode)
* **`di{`**: Delete Inside curly braces (perfect for cleaning out code blocks).
* **`ctrl + v`**: Enter Visual Block mode to select vertical columns of text.
* **`:%s/old/new/g`**: Search and replace "old" with "new" globally in the file.


## Saving and Exiting Shortcuts
* **`:w`**: Write (save) the changes to the file.
* **`:q`**: Quit (fails if there are unsaved changes).
* **`:wq!`**: Force write (save) and quit, even for files with restricted permissions.
* **`:x`**: Save and exit (only writes to disk if changes were actually made).
* **`:q!`**: Quit immediately without saving any changes.

---