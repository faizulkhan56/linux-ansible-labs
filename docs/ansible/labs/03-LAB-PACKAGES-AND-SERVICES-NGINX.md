# Lab 03 — Install Nginx and manage a service (Playbook)

## Objective

- Write your first playbook
- Install `nginx`
- Ensure service is started and enabled

## Step 1 — Create a new lab folder

```bash
mkdir -p ~/ansible-labs/lab03
cd ~/ansible-labs/lab03
```

Copy `inventory.ini` from Lab 01:

```bash
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create the playbook

Create `03-nginx.yml`:

```yaml
- name: Install and start nginx
  hosts: nodes
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

## Step 3 — Run the playbook

```bash
ansible-playbook -i inventory.ini 03-nginx.yml
```

## Step 4 — Verify

```bash
ansible -i inventory.ini nodes -m shell -a "systemctl is-active nginx && curl -I http://localhost | head -n 1"
```

## Step 5 — Idempotency check

Run it again:

```bash
ansible-playbook -i inventory.ini 03-nginx.yml
```

Expected:
- Most tasks show `ok` (not `changed`)

