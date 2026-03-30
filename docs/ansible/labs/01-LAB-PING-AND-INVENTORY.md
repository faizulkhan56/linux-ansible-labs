# Lab 01 — Inventory + SSH + `ping`

## Objective

- Create a working Ansible inventory
- Verify SSH connectivity
- Run Ansible `ping` module successfully

## Prerequisites

On control node:

```bash
ansible --version
ssh -V
```

## Step 1 — Create a folder for this lab

```bash
mkdir -p ~/ansible-labs/lab01
cd ~/ansible-labs/lab01
```

## Step 2 — Create an inventory file

Create `inventory.ini`:

```ini
[masters]
master ansible_host=<MASTER_IP>

[slaves]
slave1 ansible_host=<SLAVE1_IP>

[all:vars]
ansible_user=<SSH_USERNAME>     # e.g., ubuntu
ansible_become=true

# Optional for key-based auth:
# ansible_ssh_private_key_file=/path/to/key

# Optional for password-based labs:
# ansible_password=<SSH_PASSWORD>
# ansible_become_password=<SUDO_PASSWORD>

# Convenience group so 'nodes' matches both masters and slaves
[nodes:children]
masters
slaves
```

Theory — Why inventory matters

- An inventory is Ansible’s source of truth for “what hosts to manage.” Groups (like `masters`, `slaves`, `nodes`) let you target sets of machines at once.
- `ansible_user` defines the SSH login user; `ansible_become=true` elevates privileged tasks using sudo.
- Using a `[nodes:children]` aggregator keeps commands simple across different environments (AWS/VirtualBox).

Command breakdown (inventory variables)

- `ansible_user`: remote SSH user (e.g., `ubuntu`)
- `ansible_ssh_private_key_file`: path to SSH key for auth
- `ansible_password` / `ansible_become_password`: passwords for SSH/sudo (use only for labs)
- `ansible_become`: enables privilege escalation; use `-b` in ad-hoc for the same

If you need a specific key:

```ini
[nodes:vars]
ansible_user=<SSH_USERNAME>
ansible_become=true
ansible_ssh_private_key_file=/path/to/key
```

## Step 3 — Confirm you can SSH manually

```bash
ssh <SSH_USERNAME>@<NODE1_IP>
exit
```

If SSH fails, fix it before continuing.

## Step 4 — Run Ansible ping

```bash
ansible -i inventory.ini nodes -m ping
# If you did not define [nodes], target an existing group:
# ansible -i inventory.ini all -m ping
```

Theory — Why `ping` first

- `ping` validates SSH connectivity and Python availability on managed nodes without changing anything. It’s the safest first test.

Command breakdown

- `-i inventory.ini`: points to your inventory file
- `nodes`: host pattern; must match groups/hosts in inventory
- `-m ping`: use the builtin “ping” module (not ICMP; uses Python on the host)

Expected:
- `SUCCESS` with `pong`

## Step 5 — Inspect inventory output

```bash
ansible-inventory -i inventory.ini --list
```

## Common issues

- **Permission denied (publickey)**: wrong key, wrong user, key perms too open
- **UNREACHABLE**: security group/firewall, wrong IP, SSH service down

