# VirtualBox Lab: 2 Ubuntu VMs (Ansible Master + Slave)

Goal: Use your existing two VirtualBox Ubuntu VMs:
- VM1 (master/control node): install Ansible here
- VM2 (slave/managed node): Ansible connects here over the Host-Only network

Assumptions for this lab (from your setup):
- Host-Only network: `192.168.56.0/24`
- VM1 (master): `192.168.56.101`
- VM2 (slave): `192.168.56.102`
- Linux username: `ubuntu`
- Linux password: `test123` (lab-only)

## Prerequisites

- Complete: `docs/virtualbox/01-VIRTUALBOX-2VM-NETWORK-LAB.md`
  - Two adapters per VM: `NAT Network` (internet) + `Host-Only` (192.168.56.0/24)
  - Static host-only IPs as above
- Windows PowerShell with OpenSSH client
- Both VMs are up and reachable from Windows over Host-Only IPs (ping/SSH)

## Step 1 — Ensure SSH server is installed and running (both VMs)

On each VM (do this from the VM console if SSH is not ready yet):

```bash
sudo apt-get update
sudo apt-get install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

Optional: If using UFW on the VMs, allow SSH:

```bash
sudo ufw allow OpenSSH
sudo ufw enable    # only if ufw is intended to be on
sudo ufw status
```

## Step 2 — Test SSH from Windows to each VM

From Windows PowerShell (replace IP with each VM):

```powershell
ssh ubuntu@192.168.56.101
ssh ubuntu@192.168.56.102
```

If prompted for password, use: `test123`.

Tip: If SSH says “Permission denied (publickey)”, either:
- Log in via the VM console once and ensure `openssh-server` is running, or
- Temporarily enable password auth on the VM:
  - Edit `/etc/ssh/sshd_config` and set: `PasswordAuthentication yes`
  - `sudo systemctl restart ssh`
  - Revert later to keys for better security

## Step 3 — Update Ubuntu packages (master + slave)

On master (`192.168.56.101`):

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

On slave (`192.168.56.102`):

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

## Step 4 — Install Ansible on the master

On master (`192.168.56.101`):

```bash
sudo apt-get update
sudo apt-get install -y ansible
ansible --version
```

> If not found on very new releases, use `python3 -m pip install --user ansible` and ensure `~/.local/bin` is in `PATH`.

## Step 5 — Choose how master will SSH to the slave

You have two simple options for this lab.

### Option A — Use password auth (fast for labs)

Create an inventory that includes the SSH password (lab-only):

```bash
mkdir -p ~/ansible-lab
cd ~/ansible-lab
```

Create `inventory.ini`:

```ini
[masters]
master ansible_host=192.168.56.101

[slaves]
slave1 ansible_host=192.168.56.102

[all:vars]
ansible_user=ubuntu
ansible_password=test123
ansible_become=true
ansible_become_method=sudo
ansible_become_password=test123
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Install `sshpass` on the master (required for password-based SSH):

```bash
sudo apt-get update
sudo apt-get install -y sshpass
```

Test the inventory:

```bash
ansible-inventory -i inventory.ini --graph
ansible all -i inventory.ini -m ping -k
# Or prompt for sudo/become password instead of storing it:
# ansible all -i inventory.ini -m ping -k -K
```

Notes:
- `sshpass` is required by Ansible whenever SSH password auth is used (with `-k` or with `ansible_password` in inventory).
- `-k` asks for the SSH password interactively. If `ansible_password` is set in inventory, you can omit `-k`.
- `-K` asks for the sudo/become password interactively. If `ansible_become_password` is set in inventory, you can omit `-K`.

### Option B — Use SSH keys (recommended)

On master, create a dedicated key:

```bash
ssh-keygen -t ed25519 -C "vb-ansible-lab" -f ~/.ssh/vb_lab_ed25519
```

Copy the public key to the slave (and to master itself to allow localhost operations if you like):

```bash
# Copy to slave
cat ~/.ssh/vb_lab_ed25519.pub | ssh ubuntu@192.168.56.102 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Optional: copy to master (self)
cat ~/.ssh/vb_lab_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Create `inventory.ini` for key-based auth:

```ini
[masters]
master ansible_host=192.168.56.101

[slaves]
slave1 ansible_host=192.168.56.102

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/vb_lab_ed25519
ansible_become=true
ansible_become_method=sudo
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Test:

```bash
ansible all -i inventory.ini -m ping
```

## Step 6 — First Ad-Hoc commands

Examples:

```bash
# All hosts (use shell for compound commands like &&, |, >)
ansible all -i inventory.ini -b -m shell -a "hostname && uptime"

# Masters only
ansible masters -i inventory.ini -b -a "ip a | grep -E 'inet '"

# Slaves only
ansible slaves -i inventory.ini -b -a "df -h"
```

Note:
- The default ad-hoc module is `command`, which does not process shell operators (`&&`, `|`, redirects). For compound commands, specify `-m shell` or run separate commands:

```bash
ansible all -i inventory.ini -b -a "hostname"
ansible all -i inventory.ini -b -a "uptime"
```

## Step 7 — First playbook: install nginx on the slave

Create `01-nginx.yml`:

```yaml
- name: Install nginx on VirtualBox slave
  hosts: slaves
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run it:

```bash
ansible-playbook -i inventory.ini 01-nginx.yml
```

## Step 8 — Verify nginx on the slave

SSH to the slave and check:

```bash
ssh ubuntu@192.168.56.102
systemctl status nginx --no-pager
curl -I http://localhost
```

## Troubleshooting (common)

### “Permission denied (publickey)”
- Ensure `openssh-server` is installed and running on both VMs
- If using keys, confirm the correct private key path in `inventory.ini` and permissions
- If using passwords, ensure `PasswordAuthentication yes` in `/etc/ssh/sshd_config` and restart `ssh`

### “UNREACHABLE! Failed to connect to the host via ssh”
- Confirm you can `ssh ubuntu@192.168.56.102` from the master
- Ensure VMs are on the same Host-Only network and IPs are correct
- Check that no local firewall is blocking port 22

### “sudo: a password is required”
- This lab assumes `ubuntu` has sudo without a password. If prompted, either:
  - Configure passwordless sudo for `ubuntu` in `/etc/sudoers.d/ubuntu`, or
  - Supply `ansible_become_password=test123` in inventory (lab-only)

### “Missing sudo password”
- Provide the become password in inventory:

```ini
[all:vars]
ansible_become=true
ansible_become_method=sudo
ansible_become_password=test123
```

- Or add `-K` to prompt for the sudo password at runtime:

```bash
ansible all -i inventory.ini -m ping -k -K
```

### “to use the 'ssh' connection type with passwords... install the sshpass program”
- Install `sshpass` on the master control node:

```bash
sudo apt-get update
sudo apt-get install -y sshpass
```

## Next steps

- Explore more labs: `docs/ansible/labs/00-INDEX.md`
- Convert ad-hoc tasks into roles and playbooks
- Add a second slave and update the inventory

