# Lab 02 — Ad-hoc commands

## Objective

Learn to run safe one-liners:
- connectivity checks
- gather facts
- run shell/command modules
- install a package

## Step 0 — Setup

Use your `inventory.ini` from Lab 01. Ensure your host pattern matches groups in your inventory.

```bash
cd ~/ansible-labs/lab01
```

### Choose your environment inventory

#### Option A — AWS EC2 inventory (example)

```ini
[aws_slaves]
slave1 ansible_host=<SLAVE_PRIVATE_IP>

[masters]
master ansible_host=<MASTER_PRIVATE_IP>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/ansible-lab/ansible-lab-key.pem
ansible_become=true

# Make 'nodes' pattern work by grouping both masters and slaves:
[nodes:children]
masters
aws_slaves
```

#### Option B — VirtualBox inventory (example)

```ini
[masters]
master ansible_host=192.168.56.101

[slaves]
slave1 ansible_host=192.168.56.102

[all:vars]
ansible_user=ubuntu
# For password labs:
# ansible_password=test123
# ansible_become_password=test123
ansible_become=true
ansible_become_method=sudo

# Make 'nodes' pattern work by grouping both masters and slaves:
[nodes:children]
masters
slaves
```

Tip:
- If you don’t define a `nodes` group, use `all`, `masters`, or `slaves` in the host pattern instead of `nodes`.

## Step 1 — Ping

```bash
ansible -i inventory.ini nodes -m ping
```

Theory — Ad-hoc vs playbooks

- Ad-hoc commands are quick, one-off actions (great for checks, small changes). Playbooks are versioned, repeatable automations for larger tasks.

Command breakdown

- `-i inventory.ini`: which hosts to target
- `nodes`: group defined in your inventory (or use `all`, `masters`, `slaves`)
- `-m ping`: use the builtin ping module (sanity check)

## Step 2 — Gather facts

```bash
ansible -i inventory.ini nodes -m setup | head -n 40
```

Theory — Facts

- The `setup` module collects host details (OS, network, CPU). Useful for targeting by conditions or templates.

Command breakdown

- `-m setup`: runs facts collection
- `| head -n 40`: show only top 40 lines for readability

## Step 3 — Run a command

```bash
ansible -i inventory.ini nodes -m command -a "uname -a"
```

Theory — command vs shell

- `command` is safer; it does not invoke a shell. Use it when you do not need shell features (pipes, redirects, `&&`).

Command breakdown

- `-m command -a "uname -a"`: runs binary `uname` with args on each host

## Step 4 — Run a shell pipeline (use shell module)

```bash
ansible -i inventory.ini nodes -m shell -a "df -h | tail -n +2"
```

Theory — When to use shell

- Use `shell` for complex pipelines, redirects, environment expansion. Be mindful of quoting and idempotency.

Command breakdown

- `-m shell -a "df -h | tail -n +2"`: executes via shell so `|` is interpreted

For a second contrast (**`echo hello | wc -c`** with `shell` vs `command`), see **`02-LAB-COMMAND-MODULE-BASIC.md`** — Part 1, **§11** (after the last-reboot example).

## Step 5 — Install a package (Ubuntu)

```bash
ansible -i inventory.ini nodes -b -m apt -a "name=htop state=present update_cache=yes"
```

Theory — Becoming root

- Package management requires root. `-b` (become) elevates via sudo when `ansible_become=true` is set (or configure become in inventory).

Command breakdown

- `-b`: use sudo
- `-m apt`: Ubuntu/Debian package manager module
- `name=htop state=present`: ensure package is installed (idempotent)
- `update_cache=yes`: refresh apt cache before install

Verify:

```bash
ansible -i inventory.ini nodes -m command -a "htop --version"
```

## Notes

- Next lab in depth: **`02-LAB-COMMAND-MODULE-BASIC.md`** (more `command` ad-hoc patterns + `command-demo.yml` playbook).
- Prefer `command` over `shell` unless you need shell features (pipes, redirects).
- Use `-b` (become) for tasks needing sudo.
- If you see “Could not match supplied host pattern ‘nodes’”, either:
  - Run against an existing group: `ansible -i inventory.ini all -m ping` (or `masters`, `slaves`, `aws_slaves`), or
  - Add the `[nodes:children]` section shown above to your inventory.

