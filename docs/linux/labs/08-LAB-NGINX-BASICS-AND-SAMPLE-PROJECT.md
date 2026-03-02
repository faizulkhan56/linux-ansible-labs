# Linux Lab 08 — Nginx basics + sample project (101)

## Objective

- Learn what Nginx does in simple terms
- Understand basic Nginx config layout on Ubuntu
- Install and run Nginx
- Create a simple static sample project
- Configure Nginx to serve that project
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

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

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

---

## Step 5 — Test your sample project

### Test on the same VM

```bash
curl -I http://localhost
curl http://localhost | head -n 10
```

### Test from another VM/host

From another machine on same network (replace IP):

```bash
curl -I http://<NGINX_VM_IP>
curl http://<NGINX_VM_IP> | head -n 10
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

