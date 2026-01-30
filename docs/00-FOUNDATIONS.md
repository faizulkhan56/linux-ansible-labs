# Foundations: Ansible + VirtualBox Networking (Student Notes)

This file gives **basic theory** and practical mental models. Use it before running the labs.

## Ansible foundations (quick theory)

### What is Ansible?

**Ansible** is a configuration management + automation tool that:
- Connects to servers over **SSH**
- Runs tasks using **modules** (e.g., `apt`, `service`, `copy`, `template`)
- Uses **YAML playbooks** to describe desired state
- Is **agentless** on managed nodes (no Ansible daemon needed on the remote)

### Core terms

- **Control node (master)**: the machine where you run `ansible` / `ansible-playbook`.
- **Managed node (slave)**: the machines Ansible connects to via SSH.
- **Inventory**: list of hosts + groups (INI or YAML).
- **Module**: a unit of work (install packages, manage users, copy files).
- **Playbook**: YAML file describing a set of tasks to run on hosts.
- **Role**: a structured, reusable bundle of tasks/vars/templates/handlers.
- **Idempotency**: running the same playbook multiple times should produce the same final state (no repeated changes when nothing changed).

### How Ansible connects

Most beginner labs use:
- SSH key authentication (recommended)
- `ansible_user=ubuntu` (AWS Ubuntu) or your created user
- `become: true` to run tasks with sudo

**Ports**
- SSH: TCP **22**

### Inventory basics (INI)

Example:

```ini
[web]
node1 ansible_host=192.168.56.101

[web:vars]
ansible_user=ubuntu
ansible_become=true
```

Key fields:
- `ansible_host`: IP/DNS to connect
- `ansible_user`: SSH username
- `ansible_ssh_private_key_file`: path to private key (optional)

### Command types

- **Ad-hoc commands** (one-liners):
  - `ansible all -i inventory.ini -m ping`
  - `ansible web -i inventory.ini -b -m apt -a "name=nginx state=present update_cache=yes"`
- **Playbooks** (repeatable automation):
  - `ansible-playbook -i inventory.ini playbooks/01-ping.yml`

### Playbook structure (basic)

```yaml
- name: Example play
  hosts: all
  become: true
  tasks:
    - name: Ensure nginx installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
```

### Variables and precedence (simple view)

Common variable locations:
- inventory `group_vars/` and `host_vars/`
- playbook `vars:`
- role defaults/vars

Beginner rule:
- Prefer **group_vars** for group settings, **host_vars** for host-specific settings.

### Handlers (restart services only when needed)

Tasks can “notify” a handler only when they change something:

```yaml
- name: Copy config
  ansible.builtin.copy:
    src: my.conf
    dest: /etc/my.conf
  notify: Restart service

handlers:
  - name: Restart service
    ansible.builtin.service:
      name: myservice
      state: restarted
```

### Ansible Vault (secrets)

Use Vault to encrypt sensitive vars like passwords.

- Create encrypted file: `ansible-vault create group_vars/all/vault.yml`
- Run playbook: `ansible-playbook ... --ask-vault-pass`

## VirtualBox networking foundations (quick theory)

VirtualBox provides **network adapters** for each VM. Common modes:

### NAT (simple internet only)

- VM can access internet through the host.
- Host usually **cannot** directly reach the VM (no inbound).
- VM-to-VM depends on configuration (often not ideal for labs).

### NAT Network (recommended for 2+ VMs + internet)

- Multiple VMs share a NAT “switch”.
- VMs can talk to each other on that NAT Network.
- VMs have internet access.
- Host still usually cannot directly connect to NAT Network IPs.

### Host-Only Adapter (recommended for host ↔ VM access)

- Creates a private network between host and VMs.
- Host can SSH to VMs using host-only IPs.
- No internet via host-only (unless you add routing/NAT separately).

### Bridged Adapter (sometimes OK, but depends on your LAN)

- VM appears as another device on your physical network.
- Works well in some offices, blocked in others (Wi-Fi restrictions).

## Recommended VirtualBox lab design (best for classrooms)

Use **two NICs per VM**:

1. **Adapter 1: NAT Network** → internet access + VM-to-VM possible
2. **Adapter 2: Host-Only Adapter** → host Windows can SSH to each VM, and VMs can also talk to each other

Then:
- Enable SSH on both VMs
- Use the **host-only** IPs for SSH from Windows and for Ansible inventory

## SSH essentials (for both AWS and VirtualBox)

### Generate SSH keys (control node)

```bash
ssh-keygen -t ed25519 -C "student-lab" -f ~/.ssh/lab_ed25519
```

### Copy public key to a target

```bash
ssh-copy-id -i ~/.ssh/lab_ed25519.pub ubuntu@<TARGET_IP>
```

If `ssh-copy-id` is not available:

```bash
cat ~/.ssh/lab_ed25519.pub | ssh ubuntu@<TARGET_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Common SSH troubleshooting

- Wrong username (`ubuntu` on AWS Ubuntu; on VirtualBox it depends on what you created)
- Wrong key file permissions (`chmod 600 ~/.ssh/lab_ed25519`)
- Firewall/security group blocking port 22

