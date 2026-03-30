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

Ensure your inventory has a `nodes` group (aggregating your targets). If not, use `all` or a specific group like `slaves` or add:

```ini
[nodes:children]
masters
slaves
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

## Step 6 — Cleanup: remove nginx from all nodes

Option A — Use a dedicated cleanup playbook `03-nginx-remove.yml`:

Create `03-nginx-remove.yml`:

```yaml
- name: Remove nginx from all nodes
  hosts: nodes
  become: true
  tasks:
    - name: Stop and disable nginx if present
      ansible.builtin.service:
        name: nginx
        state: stopped
        enabled: false
      ignore_errors: true

    - name: Uninstall nginx (purge configuration)
      ansible.builtin.apt:
        name: nginx
        state: absent
        purge: true
        update_cache: true
        lock_timeout: 180

    - name: Autoremove unused packages (no upgrades)
      ansible.builtin.apt:
        autoremove: true
        lock_timeout: 180
      register: autoremove_result
      retries: 5
      delay: 10
      until: autoremove_result is succeeded

    - name: Remove default web root if left behind
      ansible.builtin.file:
        path: /var/www/html
        state: absent
```

Run it:

```bash
ansible-playbook -i inventory.ini 03-nginx-remove.yml
```

Option B — Quick ad‑hoc removal (without creating a file):

```bash
# Stop service if present (ignore failures)
ansible -i inventory.ini nodes -b -m service -a "name=nginx state=stopped" || true

# Remove package and purge configs
ansible -i inventory.ini nodes -b -m apt -a "name=nginx state=absent purge=yes update_cache=yes"

# Optional: clean leftovers and autoremove deps (will wait for locks)
ansible -i inventory.ini nodes -b -m apt -a "autoremove=yes lock_timeout=180"
ansible -i inventory.ini nodes -b -m file -a "path=/var/www/html state=absent"
```

Note:
- If you hit “Could not get lock /var/lib/dpkg/lock-frontend”, another apt process (e.g., unattended-upgrades) is running. The playbook above uses `lock_timeout` and retries to wait for the lock instead of failing. If it still fails, wait a minute and re-run.
