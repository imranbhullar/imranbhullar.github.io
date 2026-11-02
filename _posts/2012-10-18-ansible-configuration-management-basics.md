---
layout: post
title: "Ansible Basics: Agentless Configuration Management for Linux Fleets"
date: Oct-18-2012
categories: [Linux, DevOps]
tags: [linux, ansible, automation, configuration-management, devops]
image: /assets/img/ansible-automation-robotic-arm.jpg
---

Hand-running the same five commands across twenty servers doesn't scale, and it doesn't leave a record of what you did. Ansible manages that fleet from a single control node over plain SSH, with no agent to install or maintain on the managed hosts. This post covers the pieces you need to go from zero to a working playbook.

## Why Agentless Wins for Most Shops

Puppet, Chef, and Salt all traditionally run an agent on every managed node, which means a daemon to patch, a certificate to rotate, and a client version to keep in sync with the master. Ansible instead pushes modules over SSH, runs them, and cleans up after itself.

```bash
# Install the control node tooling (only needed on the machine you run from)
sudo apt install ansible      # Debian/Ubuntu
sudo dnf install ansible      # RHEL/Fedora

ansible --version
```

*Tip:* Because Ansible needs nothing installed on managed nodes beyond Python and SSH, it's the easiest of the major tools to introduce into an environment that doesn't already have configuration management — there's no agent rollout to coordinate first.

## Inventory: Telling Ansible What It's Managing

An inventory file lists the hosts Ansible can act on, optionally grouped and with per-host or per-group variables.

```ini
# /etc/ansible/hosts (INI style)
[webservers]
web1.example.com
web2.example.com ansible_port=2222

[dbservers]
db1.example.com

[webservers:vars]
http_port=80
```

```yaml
# Equivalent inventory.yml (YAML style)
webservers:
  hosts:
    web1.example.com:
    web2.example.com:
      ansible_port: 2222
  vars:
    http_port: 80
dbservers:
  hosts:
    db1.example.com:
```

*Tip:* Point at a non-default inventory explicitly with `-i inventory.yml` rather than editing `/etc/ansible/hosts`; keeping the inventory next to the playbooks in version control is what makes the whole setup reproducible.

## Ad-Hoc Commands

Before writing a playbook, `ansible` itself runs a single module against a set of hosts, which is useful for quick checks or emergency changes.

```bash
# Ping every host (checks SSH + Python, not ICMP)
ansible all -i inventory.yml -m ping

# Check uptime across the webservers group
ansible webservers -i inventory.yml -m command -a "uptime"

# Install a package, as root, on one group
ansible dbservers -i inventory.yml -m apt -a "name=postgresql state=present" --become
```

*Tip:* `-m command` runs a raw command and is intentionally "unsafe" — it doesn't check idempotency. Reach for a proper module (`apt`, `copy`, `service`, etc.) whenever one exists for the job, and keep `command`/`shell` for the genuine gaps.

## Writing a Simple Playbook

A playbook is YAML describing a list of plays, each targeting a set of hosts and running an ordered list of tasks.

```yaml
# site.yml
---
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Deploy nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Reload nginx

    - name: Ensure nginx is enabled and running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

```bash
ansible-playbook -i inventory.yml site.yml
```

*Tip:* `become: true` is Ansible's equivalent of `sudo`; set it per-play or per-task rather than globally in `ansible.cfg` so it's obvious from the playbook alone which tasks actually need elevated privileges.

## Idempotency: Running the Same Playbook Twice

A well-written Ansible task describes a desired end state, not a one-time action, so running the playbook again against an already-configured host should report no changes.

```bash
# First run: changes happen
ansible-playbook -i inventory.yml site.yml
# PLAY RECAP: web1.example.com : ok=4 changed=3 ...

# Second run: nothing left to do
ansible-playbook -i inventory.yml site.yml
# PLAY RECAP: web1.example.com : ok=4 changed=0 ...
```

*Tip:* A task that shows `changed` on every single run is a sign it's using `command`/`shell` where a declarative module exists, or that the module is being driven with options that don't let it detect current state correctly.

## Variables and Facts

Ansible gathers detailed facts about each host automatically, and combines them with variables you define, letting a single playbook adapt per host.

```yaml
- name: Show the OS family and total memory
  debug:
    msg: "{{ ansible_facts['os_family'] }} with {{ ansible_facts['memtotal_mb'] }} MB RAM"

- name: Install the right package name per distro family
  apt:
    name: "{{ 'apache2' if ansible_facts['os_family'] == 'Debian' else 'httpd' }}"
    state: present
```

```bash
# Dump every fact Ansible collects about a host
ansible web1.example.com -m setup
```

*Tip:* `ansible_facts` is gathered fresh at the start of every play by default, which costs a few seconds per host; add `gather_facts: false` to plays that don't need it to speed up large runs.

## Handlers and notify

Handlers are tasks that only run when notified by another task that actually changed something, which avoids restarting a service on every run regardless of whether its config changed.

```yaml
tasks:
  - name: Deploy sshd config
    template:
      src: templates/sshd_config.j2
      dest: /etc/ssh/sshd_config
    notify: Restart sshd

handlers:
  - name: Restart sshd
    service:
      name: sshd
      state: restarted
```

*Tip:* A handler only fires once per play even if multiple tasks notify it, and it runs at the end of the play by default — use `meta: flush_handlers` mid-play if you need it to run earlier.

## Organizing Playbooks with Roles

Once a playbook grows past a handful of tasks, a role gives it a standard directory layout that's reusable across projects.

```bash
ansible-galaxy init roles/nginx

roles/nginx/
├── tasks/main.yml
├── handlers/main.yml
├── templates/
├── files/
├── defaults/main.yml
└── vars/main.yml
```

```yaml
# site.yml, using the role
- hosts: webservers
  become: true
  roles:
    - nginx
```

*Tip:* `defaults/main.yml` holds variables meant to be overridden by whoever uses the role, while `vars/main.yml` holds ones the role depends on internally — mixing the two up is a common source of confusing override behavior.

## Secrets with ansible-vault

Playbooks often need credentials that can't sit in a git repo in plain text; `ansible-vault` encrypts them in place.

```bash
# Create an encrypted file
ansible-vault create group_vars/dbservers/vault.yml

# Edit it later
ansible-vault edit group_vars/dbservers/vault.yml

# Run a playbook that references vaulted variables
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

*Tip:* Keep vaulted variables in a separate file (conventionally `vault.yml`) from the rest of `group_vars`, and reference them by name from the plain-text vars file — that way `git diff` on the plain file still shows you what variable names changed, even though their vaulted values are opaque.

## Dry Runs with --check and --diff

Before applying a playbook to production, `--check` reports what would change without actually making those changes, and `--diff` shows the specific content that would be modified.

```bash
ansible-playbook -i inventory.yml site.yml --check --diff
```

*Tip:* `--check` mode isn't perfectly reliable for every module — some can't predict their own idempotency without actually running — so treat it as a strong sanity check before a production run, not an absolute guarantee.
