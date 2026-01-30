# Example Ansible Project (Use in AWS or VirtualBox)

This folder is a **ready-to-run Ansible skeleton** that matches the labs in `docs/ansible/labs/`.

## Quick start

1. Update inventory IPs:
   - Edit `inventory.ini`
2. Test connectivity:

```bash
cd ansible
ansible -i inventory.ini all -m ping
```

3. Run a playbook:

```bash
ansible-playbook -i inventory.ini playbooks/01-ping.yml
ansible-playbook -i inventory.ini playbooks/02-nginx.yml
ansible-playbook -i inventory.ini playbooks/03-users.yml
```

## Files

- `ansible.cfg`: defaults (host key checking off for labs)
- `inventory.ini`: example inventory for 2 nodes
- `group_vars/all.yml`: common vars
- `playbooks/`: sample playbooks
- `roles/nginx/`: example role used by `playbooks/02-nginx.yml`

## Safety notes (for training)

- This repo disables SSH host key checking in `ansible.cfg` to reduce classroom friction.
  - In real environments, keep host key checking enabled.

