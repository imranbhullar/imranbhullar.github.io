---
layout: post
title: "Git Essentials: A Comprehensive Guide to Basic Operations"
date: 2026-01-10 21:40:00 +0000
description: "Master the core Git workflow: from repo initialization and configuration to advanced logging, reverting, and version tagging."
categories: [DevOps, Guide]
tags: [Git, Version Control, Linux, Workflow]
---
![nix](/assets/img/Linux_unix_bsd.png)

Efficient version control is the foundation of any solid DevOps pipeline. Whether you are managing infrastructure code or software applications, mastering these basic Git operations is essential for maintaining a clean and traceable project history.

---

## 1. Understanding Git Repositories
A repository (or "repo") is the central folder where Git tracks the history of your project.

* **Initialize a new local repo:**
    ```bash
    git init
    ```
* **Clone a remote repo to your machine:**
    ```bash
    git clone [https://github.com/username/project-name.git](https://github.com/username/project-name.git)
    ```

---

## 2. Configuring Git
Before making changes, you must configure your identity so your commits are correctly attributed.

* **Set global user details:**
    ```bash
    git config --global user.name "Your Name"
    git config --global user.email "your.email@example.com"
    ```
* **Verify configuration:**
    ```bash
    git config --list
    ```

---

## 3. Local Repos and Inspection
Once your repo is set up, you need to track changes and inspect the project's state.

* **Check the status of your files:** `git status`
* **View full commit history:**
    ```bash
    git log
    ```
* **View history in a condensed, one-line format:**
    ```bash
    git log --oneline
    ```
* **Show details of a specific commit:**
    ```bash
    git show <commit-id>
    ```

---

## 4. Reverting and Undoing Changes
Git provides several ways to fix mistakes, depending on whether the changes are committed or just sitting in your working directory.

* **Discard changes in a specific file (not yet committed):**
    ```bash
    git checkout -- <filename>
    ```
* **Revert a specific commit (safe for shared repos):**
    ```bash
    git revert <commit-id>
    ```
    *Note: This creates a new commit that is the exact opposite of the one you are reverting.*

---

## 5. Creating SSH Repos
SSH is the preferred method for secure, password-less communication with remote servers like GitHub or GitLab.

* **Generate an SSH key:**
    ```bash
    ssh-keygen -t ed25519 -C "your.email@example.com"
    ```
* **Add remote via SSH:**
    ```bash
    git remote add origin git@github.com:user/repo.git
    ```

---

## 6. Branching and Merging
Branches allow you to work on new features or bug fixes without affecting the stable `main` code.

* **Create and switch to a new branch:**
    ```bash
    git checkout -b feature-update
    ```
* **Merge changes back to main:**
    ```bash
    git checkout main
    git merge feature-update
    ```

---

## 7. Versioning and Tags
Tags are used to mark specific release points (milestones) in your project’s timeline.

* **Create a Lightweight Tag:**
    ```bash
    git tag v1.0.0
    ```
* **Create an Annotated Tag (Recommended for Releases):**
    ```bash
    git tag -a v1.1.0 -m "Release version 1.1.0 with feature updates"
    ```
* **Push tags to the remote server:**
    ```bash
    git push origin --tags
    ```

---

### Summary Table of Common Commands

| Command | Description |
| :--- | :--- |
| `git log --oneline` | Displays a short version of the commit history. |
| `git show` | Shows the changes in the most recent or specific commit. |
| `git revert` | Undoes a commit by creating a new inverse commit. |
| `git tag` | Lists or creates version markers for the project. |

> **Pro Tip:** When working with Terraform or CI/CD, always use annotated tags (`-a`) to provide clear release notes for your infrastructure changes!