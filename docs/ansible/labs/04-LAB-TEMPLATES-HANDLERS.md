# Lab 04 — Templates + Handlers

## Objective

- Use a Jinja2 template to create a config file
- Restart a service only when config changes (handler)

## Step 1 — Prepare lab folder

```bash
mkdir -p ~/ansible-labs/lab04/templates
cd ~/ansible-labs/lab04
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create a simple HTML template

Create `templates/index.html.j2`:

```html
<html>
  <head><title>{{ page_title }}</title></head>
  <body>
    <h1>{{ page_title }}</h1>
    <p>Server hostname: {{ ansible_hostname }}</p>
    <p>Managed by Ansible at: {{ ansible_date_time.iso8601 }}</p>
  </body>
</html>
```

## Step 3 — Create playbook with handler

Create `04-template-handler.yml`:

```yaml
- name: Configure nginx homepage using template + handler
  hosts: nodes
  become: true
  vars:
    page_title: "Hello from Ansible Lab 04"
  tasks:
    - name: Ensure nginx installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Deploy custom index.html
      ansible.builtin.template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        mode: "0644"
      notify: Restart nginx

    - name: Ensure nginx running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

## Step 4 — Run

```bash
ansible-playbook -i inventory.ini 04-template-handler.yml
```

## Step 5 — Verify

```bash
ansible -i inventory.ini nodes -m shell -a "curl -s http://localhost | head -n 10"
```

## Step 6 — Confirm handler behavior

Run playbook twice:
- First run should show `changed` for template task and handler runs.
- Second run should show `ok` and handler should not run.

