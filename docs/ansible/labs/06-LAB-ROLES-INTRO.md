# Lab 06 — Roles (intro)

## Objective

- Convert playbook logic into a role
- Understand role folder structure
- Apply a role to hosts

## Step 1 — Prepare folder

```bash
mkdir -p ~/ansible-labs/lab06
cd ~/ansible-labs/lab06
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create a role skeleton

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,defaults}
```

Create `roles/nginx/defaults/main.yml`:

```yaml
nginx_page_title: "Hello from Role: nginx"
```

Create `roles/nginx/templates/index.html.j2`:

```html
<html>
  <head><title>{{ nginx_page_title }}</title></head>
  <body>
    <h1>{{ nginx_page_title }}</h1>
    <p>Hostname: {{ ansible_hostname }}</p>
  </body>
</html>
```

Create `roles/nginx/handlers/main.yml`:

```yaml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

Create `roles/nginx/tasks/main.yml`:

```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Deploy homepage
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html
    mode: "0644"
  notify: Restart nginx

- name: Ensure nginx running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

## Step 3 — Create the playbook using the role

Create `06-role-nginx.yml`:

```yaml
- name: Nginx via role
  hosts: nodes
  become: true
  roles:
    - nginx
```

## Step 4 — Run

```bash
ansible-playbook -i inventory.ini 06-role-nginx.yml
```

## Step 5 — Override role vars (optional)

Update playbook:

```yaml
- name: Nginx via role
  hosts: nodes
  become: true
  vars:
    nginx_page_title: "Overridden title from playbook"
  roles:
    - nginx
```

Run again and verify with:

```bash
ansible -i inventory.ini nodes -m shell -a "curl -s http://localhost | head -n 5"
```

