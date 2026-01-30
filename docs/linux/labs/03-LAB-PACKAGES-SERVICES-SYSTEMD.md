# Linux Lab 03 — Packages + services (systemd)

## Objective

- Install packages with `apt`
- Manage services with `systemctl`
- Understand enabled vs started

## Step 1 — Update package index

```bash
sudo apt-get update
```

## Step 2 — Install a package

```bash
sudo apt-get install -y nginx
nginx -v
```

## Step 3 — Manage the service

```bash
sudo systemctl status nginx --no-pager
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl is-enabled nginx
sudo systemctl is-active nginx
```

## Step 4 — Check ports

```bash
ss -lntp | grep -E ':80\s' || true
curl -I http://localhost | head -n 1
```

## Step 5 — View service logs

```bash
journalctl -u nginx --no-pager -n 50
```

