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
[nodes]
node1 ansible_host=<NODE1_IP>

[nodes:vars]
ansible_user=<SSH_USERNAME>
ansible_become=true
```

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
```

Expected:
- `SUCCESS` with `pong`

## Step 5 — Inspect inventory output

```bash
ansible-inventory -i inventory.ini --list
```

## Common issues

- **Permission denied (publickey)**: wrong key, wrong user, key perms too open
- **UNREACHABLE**: security group/firewall, wrong IP, SSH service down

