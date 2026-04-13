# Lab 06 — Roles (intro)

## Objective

- Convert playbook logic into a role
- Understand role folder structure
- Apply a role to hosts
- Use **`ansible.builtin.template`** with `roles/<role>/templates/` and **`ansible.builtin.copy`** with `roles/<role>/files/`

## Lab directory layout — where everything lives

All paths in this lab are under your working directory:

```text
~/ansible-labs/lab06/
├── inventory.ini
├── 06-role-nginx.yml          ← main playbook (entry point)
└── roles/
    └── nginx/                 ← role named "nginx" (must match roles: list)
        ├── tasks/
        │   └── main.yml       ← task list Ansible runs for this role
        ├── handlers/
        │   └── main.yml       ← handlers notified from tasks
        ├── templates/
        │   └── index.html.j2  ← Jinja2 template used by the template module
        ├── files/
        │   └── health.txt     ← static file used by the copy module
        └── defaults/
            └── main.yml       ← default variables (lowest precedence; overridable)
```

**Why the playbook sits next to `roles/`:** You run `ansible-playbook` from `~/ansible-labs/lab06/` with `-i inventory.ini` and `06-role-nginx.yml`. Ansible resolves `roles: - nginx` by looking for `roles/nginx/` **relative to the playbook** (or configured role paths). Keeping `06-role-nginx.yml` and `roles/` in the **same** directory avoids path surprises and matches how we `cd` into the lab folder in Step 1.

## How files reference each other (the chain)

1. **Playbook (`06-role-nginx.yml`)** — Entry point. It selects hosts and lists `roles: - nginx`.
2. **Role directory (`roles/nginx/`)** — When the `nginx` role runs, Ansible loads **`roles/nginx/tasks/main.yml`** automatically. You do not `include` that path by hand in the playbook.
3. **Tasks (`tasks/main.yml`)** — The work: install package, copy static files, render templates, ensure service. A task can **`notify: Restart nginx`**, which queues a handler **defined** in `handlers/main.yml`.
4. **Template (`templates/index.html.j2`)** — Referenced from a task with `ansible.builtin.template` and `src: index.html.j2`. Ansible resolves that name inside this role’s **`templates/`** directory (see subsection below).
5. **Static file (`files/health.txt`)** — Referenced from a task with `ansible.builtin.copy` and `src: health.txt`. Ansible resolves that name inside this role’s **`files/`** directory (no Jinja processing unless you use `content:` elsewhere).
6. **Defaults (`defaults/main.yml`)** — Defines `nginx_page_title` (and any other defaults). Tasks and templates use `{{ nginx_page_title }}`. The playbook can override these with `vars:` (Step 5).

**Short version:** playbook calls the role → role runs `tasks/main.yml` → tasks use templates and variables → handlers run when notified and something changed.

## Why we split into a role (not one big playbook)

- **Separation of concerns:** Tasks describe *what* to do; **`templates/`** holds Jinja2 sources for **`template`**; **`files/`** holds static blobs for **`copy`**; handlers are *actions on change*; defaults are *configurable settings*.
- **Reusability:** The same `roles/nginx/` tree can be copied or shared across projects and included from many playbooks.
- **Maintainability:** Easier to read, debug, and extend than a single long playbook.
- **Scalability:** Real environments reuse roles across many hosts and teams; the layout matches Ansible community practice.

## Step 1 — Prepare folder

This lab (**Lab 06 only**) assumes you have an inventory that defines group **`nodes`** and that Ansible can SSH to those hosts with privilege escalation (same idea as **Lab 01**). If you already have `~/ansible-labs/lab01/inventory.ini`, copy it; otherwise create `inventory.ini` in this folder with your hosts under `nodes`.

```bash
mkdir -p ~/ansible-labs/lab06
cd ~/ansible-labs/lab06
cp ~/ansible-labs/lab01/inventory.ini .
```

## Step 2 — Create a role skeleton

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,files,defaults}
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

Create **`roles/nginx/files/health.txt`** — this file **must exist on your Ansible control machine** (where you run `ansible-playbook`). The `mkdir` line above already creates the empty `roles/nginx/files/` directory; you add the file next.

It is **plain text** (not Jinja2). Ansible’s **`copy`** module will upload it to the managed node unchanged.

**Option A — one line (from `~/ansible-labs/lab06`):**

```bash
printf 'lab06-ok\n' > roles/nginx/files/health.txt
```

**Option B — editor:**

```bash
nano roles/nginx/files/health.txt
```

Type exactly `lab06-ok` on one line, save, and exit.

**Option C — here-document:**

```bash
cat > roles/nginx/files/health.txt <<'EOF'
lab06-ok
EOF
```

**Verify the file is in the right place before you run the playbook:**

```bash
ls -l roles/nginx/files/health.txt
cat roles/nginx/files/health.txt
```

You should see one line: `lab06-ok`. If this file is missing or misnamed, the **Deploy static health check file** task will fail with “could not find or access `health.txt`”.

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

- name: Deploy static health check file (copy from role files/)
  ansible.builtin.copy:
    src: health.txt
    dest: /var/www/html/health.txt
    mode: "0644"
  # No notify: nginx serves new files under the docroot without a reload.

- name: Deploy homepage (template from role templates/)
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

### How `src:` resolves inside a role (template vs copy)

Inside a role, modules resolve certain `src` values **relative to standard subdirectories** of that role. This is **built-in Ansible role behavior**, not something you configure per task.

| Module (examples) | Looks under role subfolder |
|-------------------|----------------------------|
| `ansible.builtin.template` | `roles/<role_name>/templates/` |
| `ansible.builtin.copy` | `roles/<role_name>/files/` |

**Template example** — this task:

```yaml
ansible.builtin.template:
  src: index.html.j2
```

expects **`roles/nginx/templates/index.html.j2`** when the task runs as part of the **`nginx`** role. The file is processed as **Jinja2** before being written to `dest`.

**Copy example** — this task:

```yaml
ansible.builtin.copy:
  src: health.txt
  dest: /var/www/html/health.txt
```

expects **`roles/nginx/files/health.txt`**. The file is copied **byte-for-byte** (no variable substitution in the source file). Use **`copy`** for static assets; use **`template`** when you need `{{ variables }}` in the file.

You do **not** write `roles/nginx/templates/...` or `roles/nginx/files/...` in `src` for these defaults—Ansible supplies the role path. You can use absolute paths or other patterns only if you intentionally change how lookup works.

**When to notify the handler:** Here, only the **homepage template** notifies **Restart nginx**, because changing `index.html` is served by nginx and a restart is a safe way to pick up config-related changes in teaching labs. A **new static file** in the docroot is visible to nginx **without** a reload, so the **`copy`** task has no `notify`.

**Standard role layout (common directories):**

```text
roles/nginx/
├── tasks/main.yml      ← always loaded for the role
├── handlers/main.yml
├── templates/          ← template module
├── files/              ← copy module
├── defaults/main.yml
└── vars/               ← optional; higher precedence than defaults
```

If you put a file in the wrong folder (for example `health.txt` under `templates/` but use `copy`), Ansible will fail to find `src` unless you fix the path or layout. Following the convention keeps playbooks portable and predictable.

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
cd ~/ansible-labs/lab06
ansible-playbook -i inventory.ini 06-role-nginx.yml
```

The run should complete with **no failed tasks**.

**Verification (managed nodes):** Steps below use **`curl`** on each target. If `curl` is not installed (common on minimal images), install it once:

```bash
ansible -i inventory.ini nodes -b -m ansible.builtin.apt -a "name=curl state=present update_cache=true"
```

Then verify from your **control node** (change `nodes` if your inventory group name differs):

```bash
ansible -i inventory.ini nodes -m shell -a "curl -s http://localhost/health.txt"
ansible -i inventory.ini nodes -m shell -a "curl -s http://localhost/ | head -n 5"
```

**Success checks:**

- `health.txt` response body contains `lab06-ok` (proves **`copy`** + `roles/nginx/files/`).
- HTML from `/` shows your title and hostname (proves **`template`** + `roles/nginx/templates/` + defaults/facts).

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
ansible -i inventory.ini nodes -m shell -a "curl -s http://localhost/health.txt"
```

## Will Lab 06 run successfully?

**Yes**, for **this lab only**, on typical **Ubuntu/Debian** targets in inventory group **`nodes`**, if:

- Inventory and SSH work, and **`become`** (sudo) works on those hosts.
- Targets can run **`apt`** (the role uses `ansible.builtin.apt`).
- **`roles/nginx/files/health.txt`** exists on the **control** node (see Step 2).
- Nothing else blocks port **80** after nginx installs.

**Second run (idempotency):** Tasks should report **`ok`** (unchanged) where nothing drifted; the handler runs only if the template task reports **changed**.

**If something fails:**

- **`Could not find or access 'health.txt'`** — file missing under **`roles/nginx/files/`** on the control machine, or wrong `src` name.
- **`Could not find or access 'index.html.j2'`** — file missing under **`roles/nginx/templates/`**.
- **`curl: not found`** on verification — install **`curl`** on targets (see Step 4) or use another HTTP check you prefer.
- **Connection / permission errors** — fix inventory, SSH, and `become: true`.

