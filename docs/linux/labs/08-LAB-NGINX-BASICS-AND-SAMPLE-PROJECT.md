# Linux Lab 08 — Nginx basics + sample project (101)

## Objective

- Learn what Nginx does in simple terms
- Understand basic Nginx config layout on Ubuntu
- Install and run Nginx
- Create a simple static sample project (site root plus `/about`, `/faq`, and `/home`)
- Understand what **`$uri`** means and how **`try_files $uri $uri/`** maps URLs to files and directories
- Configure Nginx to serve that project (catch‑all `location /`, optional explicit `location` blocks)
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

### How `try_files $uri $uri/` relates to `$uri` (read before Step 4)

Many sample configs use:

```nginx
location / {
    try_files $uri $uri/ =404;
}
```

With `root /var/www/sample-project;`, Nginx resolves paths like this:

1. **`$uri` as a file** — For a request such as `GET /style.css`, Nginx checks whether the file  
   `/var/www/sample-project` + `/style.css` exists. If yes, that file is served.
2. **`$uri/` as a directory** — If the file step fails, Nginx checks whether a **directory** named after the path exists (for `GET /about`, it looks for a folder `about/` under `root`). If it exists, Nginx continues in that directory and can serve the configured **`index`** (for example `index.html`).
3. **`=404`** — If neither a matching file nor directory is found, Nginx returns **404 Not Found**.

So you can add URLs like `/about`, `/faq`, or `/home` **without** extra `location` blocks as long as you mirror them with folders (or files) under `root`:

| Browser URL (typical) | On disk under `root` (this lab) |
|----------------------|----------------------------------|
| `/` | `index.html` (via `index`) |
| `/about` or `/about/` | `about/index.html` |
| `/faq` or `/faq/` | `faq/index.html` |
| `/home` or `/home/` | `home/index.html` |

Later, if a path needs **different** rules (another `root`, authentication, caching headers, PHP, etc.), you add a dedicated `location` for that prefix. Step 4 includes the usual catch‑all; an optional explicit `location` example appears after Step 4 for comparison.

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
    <a href="/about/">About</a> ·
    <a href="/faq/">FAQ</a> ·
    <a href="/home/">Home</a>
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

Create extra pages for other paths (same `root`, no extra `location` required yet — `try_files $uri $uri/` will find each folder’s `index.html`):

```bash
sudo mkdir -p /var/www/sample-project/about /var/www/sample-project/faq /var/www/sample-project/home
```

`about` page:

```bash
sudo tee /var/www/sample-project/about/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8" /><title>About — Nginx 101</title></head>
<body>
  <h1>About</h1>
  <p>This page is served from <code>/var/www/sample-project/about/index.html</code> for URL <code>/about/</code>.</p>
  <p><a href="/">Back to home</a></p>
</body>
</html>
EOF
```

`faq` page:

```bash
sudo tee /var/www/sample-project/faq/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8" /><title>FAQ — Nginx 101</title></head>
<body>
  <h1>FAQ</h1>
  <p>This page is served from <code>/var/www/sample-project/faq/index.html</code> for URL <code>/faq/</code>.</p>
  <p><a href="/">Back to home</a></p>
</body>
</html>
EOF
```

`home` page:

```bash
sudo tee /var/www/sample-project/home/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head><meta charset="utf-8" /><title>Home section — Nginx 101</title></head>
<body>
  <h1>Home section</h1>
  <p>This page is served from <code>/var/www/sample-project/home/index.html</code> for URL <code>/home/</code>.</p>
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

    # $uri = request path; try_files: file → directory (with index) → 404
    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

Theory note (ties to the table in “How `try_files`…”):

- Requests to `/`, `/about`, `/faq`, `/home` are all handled by this single `location /` because each maps to a real file or directory under the same `root`.
- If you later need **different** behavior per prefix, add more specific `location` blocks (see optional block after this step).

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

### Optional — explicit `location` blocks for `/about`, `/faq`, `/home`

You **do not** need the following if the catch‑all `location /` with `try_files` already works. Use this pattern when you want a clear per‑URL section in config (headers, logging, `alias`, PHP, rate limits, etc.):

```bash
sudo tee /etc/nginx/sites-available/sample-project.conf > /dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;

    server_name _;
    root /var/www/sample-project;
    index index.html;

    # Prefix locations: longer/more specific paths can be listed before the catch‑all.
    location /about/ {
        try_files $uri $uri/ =404;
    }

    location /faq/ {
        try_files $uri $uri/ =404;
    }

    location /home/ {
        try_files $uri $uri/ =404;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

Then run `sudo nginx -t` and `sudo systemctl reload nginx` again.

Note: with the **same** `root` everywhere, behavior matches the single `location /` example; the value is **organization and a place to attach per‑path directives** later.

---

## Step 5 — Test your sample project

### Test on the same VM

```bash
curl -I http://localhost
curl http://localhost | head -n 10
```

Paths under the same site (check titles or status **200**):

```bash
curl -I http://localhost/about/
curl -I http://localhost/faq/
curl -I http://localhost/home/
curl -s http://localhost/about/ | head -n 5
```

Optional: many browsers request `/about` without a trailing slash; Nginx may respond with **301** redirect to `/about/` when the directory exists — both are normal:

```bash
curl -I http://localhost/about
```

### Test from another VM/host

From another machine on same network (replace IP):

```bash
curl -I http://<NGINX_VM_IP>
curl http://<NGINX_VM_IP> | head -n 10
curl -I http://<NGINX_VM_IP>/about/
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

