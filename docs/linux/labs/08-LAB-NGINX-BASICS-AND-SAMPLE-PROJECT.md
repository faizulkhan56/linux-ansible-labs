# Linux Lab 08 — Nginx basics + sample project (101)

## Objective

- Learn what Nginx does in simple terms
- Understand basic Nginx config layout on Ubuntu
- Install and run Nginx
- Create a **file-level** static layout: `index.html`, `about.html`, and `faq.html` in one folder (not `about/` directories)
- Understand **`$uri`** and **`try_files $uri $uri.html $uri/`** (exact file → same name with `.html` → directory → 404)
- Map **`/home`** to the site root page with **`location = /home`** (separate from a `/home/` directory on disk)
- Contrast **file-level** vs **directory-level** URL design so `location /about/` style blocks are not misapplied
- Test locally and from another machine

## Before you start (theory)

### What is Nginx?

Nginx is a fast web server. In beginner labs, we commonly use it to:
- serve static files (`.html`, `.css`, `.js`, images)
- listen on port `80` for HTTP requests
- route requests to the right site configuration

### Core Nginx terms (101)

- **worker process**: handles incoming client requests
- **server block**: virtual host definition (`server { ... }`)
- **root**: folder where web files are served from
- **index**: default file such as `index.html`
- **location**: URL path rule, for example `location /`
- **$uri**: Nginx variable holding the **normalized request path** (decoded, without the query string). Examples: `/`, `/about`, `/style.css`. Nginx uses it together with `root` to build the path to a file or directory on disk.

### Basic config structure on Ubuntu

- Main config: `/etc/nginx/nginx.conf`
- Site config pattern:
  - available definitions: `/etc/nginx/sites-available/`
  - enabled definitions (symlinks): `/etc/nginx/sites-enabled/`

Common workflow:
1. create config in `sites-available`
2. link it into `sites-enabled`
3. test syntax (`nginx -t`)
4. reload service (`systemctl reload nginx`)

### File-level vs directory-level URL design (read before Step 4)

**Two common static-site layouts** under one `root`:

| Design | On disk (examples) | URL style | Typical `try_files` idea |
|--------|-------------------|-----------|-------------------------|
| **File-level** (this lab) | `about.html`, `faq.html` next to `index.html` | `/about` → `about.html` | Also try **`$uri.html`** after `$uri` |
| **Directory-level** | `about/index.html`, `faq/index.html` | `/about/` or `/about` (often redirects) | **`$uri`** then **`$uri/`** + `index` |

**Why `location /about/` does not match flat `.html` files**

- `location /about/` is a **prefix** for directory-style paths such as `/about/` or `/about/foo`.
- If the real file is **`about.html`** in the site root, there is **no** `about/` folder. Using only directory-style locations for `/about/` is the wrong mental model; you want Nginx to map **`/about`** to **`about.html`**, not to a folder named `about`.

**`$uri` and `$uri.html`**

- **`$uri`**: normalized path (e.g. `/about`).
- **`$uri.html`**: Nginx concatenates the value of `$uri` with the literal `.html` (e.g. `/about.html`).

**Recommended pattern for file-level pages in one folder**

```nginx
location / {
    try_files $uri $uri.html $uri/ =404;
}
```

With `root /var/www/sample-project;`, Nginx tries **in order**:

1. **`$uri`** — Exact file: e.g. `GET /style.css` → `…/style.css`; `GET /about.html` → `…/about.html`.
2. **`$uri.html`** — “Pretty” URL without extension: `GET /about` → `…/about.html`.
3. **`$uri/`** — Directory (still useful for assets or if you mix in a subfolder later): e.g. a real `docs/` folder with `index.html` inside.
4. **`=404`** — Nothing matched.

**Mapping `/home` to the main `index.html` (no `home.html`)**

If you want the URL **`/home`** to show the **same** page as **`/`** (site root), use an **exact** match location (runs before the generic `location /`):

```nginx
location = /home {
    try_files /index.html =404;
}
```

Inside `location = /home`, `try_files /index.html` resolves **`index.html`** under the same `root` as the site entry page.

After editing config, always run **`sudo nginx -t`** then **`sudo systemctl reload nginx`**.

---

## Step 1 — Install and start Nginx

```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
nginx -v
```

Quick check:

```bash
curl -I http://localhost | head -n 1
```

Expected status line includes `200 OK`.

---

## Step 2 — Inspect default Nginx setup

```bash
ls -l /etc/nginx/
ls -l /etc/nginx/sites-available/
ls -l /etc/nginx/sites-enabled/
```

View default site:

```bash
sudo cat /etc/nginx/sites-available/default
```

Theory note:
- If no custom site is configured, Nginx serves this default site.

---

## Step 3 — Create a sample static project

Create project folder:

```bash
sudo mkdir -p /var/www/sample-project
```

Create `index.html`:

```bash
sudo tee /var/www/sample-project/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Nginx 101 Sample</title>
  <link rel="stylesheet" href="/style.css" />
</head>
<body>
  <h1>Nginx 101 Lab</h1>
  <nav>
    <a href="/about">About</a> ·
    <a href="/faq">FAQ</a> ·
    <a href="/home">Home</a> (same page as <a href="/">/</a> via Nginx)
  </nav>
  <p id="msg">If you can read this, Nginx is serving your sample project.</p>
  <script src="/app.js"></script>
</body>
</html>
EOF
```

Create `style.css`:

```bash
sudo tee /var/www/sample-project/style.css > /dev/null <<'EOF'
body { font-family: Arial, sans-serif; margin: 2rem; background: #f5f7fb; }
h1 { color: #1f3b8f; }
#msg { background: #fff; padding: 1rem; border-left: 4px solid #1f3b8f; }
EOF
```

Create `app.js`:

```bash
sudo tee /var/www/sample-project/app.js > /dev/null <<'EOF'
console.log("Nginx sample project loaded");
EOF
```

Create **file-level** pages in the **same** folder as `index.html` (`about.html`, `faq.html`). URLs **`/about`** and **`/faq`** are served via **`$uri.html`** in Step 4; there is no `about/` or `faq/` directory.

`about.html`:

```bash
sudo tee /var/www/sample-project/about.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8" /><title>About — Nginx 101</title></head>
<body>
  <h1>About</h1>
  <p>This page is the file <code>/var/www/sample-project/about.html</code>. The browser can use <code>/about</code> or <code>/about.html</code>.</p>
  <p><a href="/">Back to site root</a></p>
</body>
</html>
EOF
```

`faq.html`:

```bash
sudo tee /var/www/sample-project/faq.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8" /><title>FAQ — Nginx 101</title></head>
<body>
  <h1>FAQ</h1>
  <p>This page is the file <code>/var/www/sample-project/faq.html</code>. The browser can use <code>/faq</code> or <code>/faq.html</code>.</p>
  <p><a href="/">Back to site root</a></p>
</body>
</html>
EOF
```

Set readable permissions:

```bash
sudo chown -R www-data:www-data /var/www/sample-project
sudo chmod -R 755 /var/www/sample-project
```

---

## Step 4 — Create a basic Nginx server block

Create site config:

```bash
sudo tee /etc/nginx/sites-available/sample-project.conf > /dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;

    server_name _;
    root /var/www/sample-project;
    index index.html;

    # /home → same document as / (index.html); exact match runs before location /
    location = /home {
        try_files /index.html =404;
    }

    # File-level: /about → about.html; still supports real files and directories
    location / {
        try_files $uri $uri.html $uri/ =404;
    }
}
EOF
```

Theory note (ties to “File-level vs directory-level…”):

- **`/`** → `index.html` (via `index` and `try_files`).
- **`/about`** and **`/faq`** → `about.html` / `faq.html` via the **`$uri.html`** step.
- **`/about.html`** and **`/faq.html`** → served on the first **`$uri`** step.
- **`/home`** → **`location = /home`** serves **`index.html`**; there is no `home.html` in this lab.
- If a path needs **different** headers, auth, or another `root`, add a dedicated `location` for that prefix.

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/sample-project.conf /etc/nginx/sites-enabled/sample-project.conf
```

Disable default site to avoid confusion:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager
```

### Optional — SPA-style fallback (unknown paths → `index.html`)

Single-page apps often want **any** unknown path to return **`index.html`** so the client router can run. Replace the **`=404`** tail with a **URI** fallback (internal redirect):

```nginx
location / {
    try_files $uri $uri.html $uri/ /index.html;
}
```

Caution: every non-file URL can then return **200** with the same HTML (good for SPAs, confusing for traditional static sites). Use only when you intend that behavior.

### Optional — directory-level layout (when `location /about/` *is* appropriate)

If you **actually** have folders such as `about/index.html` under `root`, then directory-style URLs and prefix locations match the disk:

```text
/var/www/sample-project/
├── index.html
├── about/
│   └── index.html
└── faq/
    └── index.html
```

Then `try_files $uri $uri/ =404;` (without **`$uri.html`**) is natural, and **`location /about/`** can make sense when you need **different** rules under that subtree (extra `try_files`, headers, etc.). Do **not** mix that mental model with a flat **`about.html`**-only layout unless you know why both exist.

---

## Step 5 — Test your sample project

### Test on the same VM

```bash
curl -I http://localhost
curl http://localhost | head -n 10
```

Paths under the same site (expect **200**):

```bash
curl -I http://localhost/about
curl -I http://localhost/about.html
curl -I http://localhost/faq
curl -I http://localhost/faq.html
curl -I http://localhost/home
curl -s http://localhost/about | head -n 5
```

**`/home`** should return the same body as **`/`** (both use `index.html`). Quick compare:

```bash
curl -s http://localhost/ | sha256sum
curl -s http://localhost/home | sha256sum
```

### Test from another VM/host

From another machine on same network (replace IP):

```bash
curl -I http://<NGINX_VM_IP>
curl http://<NGINX_VM_IP> | head -n 10
curl -I http://<NGINX_VM_IP>/about
curl -I http://<NGINX_VM_IP>/home
```

If using browser, open:
- `http://<NGINX_VM_IP>`

You should see the “Nginx 101 Lab” page.

---

## Step 6 — Basic troubleshooting

- Nginx config syntax check:
  - `sudo nginx -t`
- Service state:
  - `sudo systemctl status nginx --no-pager`
- Is Nginx listening on port 80?
  - `ss -lntp | grep -E ':80\s' || true`
- Logs:
  - `sudo journalctl -u nginx --no-pager -n 100`
  - `sudo tail -n 100 /var/log/nginx/error.log`

---

## Cleanup (optional)

```bash
sudo rm -f /etc/nginx/sites-enabled/sample-project.conf
sudo rm -f /etc/nginx/sites-available/sample-project.conf
sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
sudo rm -rf /var/www/sample-project
sudo nginx -t && sudo systemctl reload nginx
```

