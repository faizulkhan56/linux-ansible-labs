# Linux Lab 03 — Packages + services (systemd)

## Objective

- Install packages with `apt`
- Manage services with `systemctl`
- Understand enabled vs started

## Theory — systemd, systemctl, journalctl (how they relate)

- systemd: the init system and service manager. It starts units (services, sockets, timers) during boot and supervises them.
- systemctl: the CLI to query and control systemd-managed units (start/stop/enable/status).
- journalctl: the CLI to read logs from systemd-journald (the central log store that collects logs from services and the kernel).

Key ideas:
- started vs enabled: started = service is running now; enabled = will start automatically at boot.
- unit types: `.service`, `.socket`, `.timer`, `.target`, etc. We mostly use `.service` here.
- logs: services writing to stdout/stderr or syslog end up in the journal and are queried via `journalctl`.

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

Command breakdown:
- `status`: shows current state and last logs
- `stop` / `start`: control runtime state
- `enable`: persist across reboot (creates symlinks in systemd config)
- `is-enabled` / `is-active`: quick checks for boot-time and runtime state

## Step 4 — Check ports

```bash
ss -lntp | grep -E ':80\s' || true
curl -I http://localhost | head -n 1
```

## Step 5 — View service logs

```bash
journalctl -u nginx --no-pager -n 50
```

Command breakdown:
- `-u nginx`: filter by the nginx service unit
- `-n 50`: show last 50 lines
- `--since "10 min ago"` and `--follow` are also useful for time window and live tail:

```bash
journalctl -u nginx --since "10 min ago"
journalctl -u nginx -f
```

## Step 6 — Create a simple error and find it with journalctl

Simulate a config error and observe failure:

```bash
sudo bash -c 'echo "syntax error!" >> /etc/nginx/nginx.conf'
sudo systemctl restart nginx || echo "restart failed (expected)"
sudo systemctl status nginx --no-pager
```

Now inspect logs for the failure:

```bash
journalctl -u nginx --no-pager -n 100 | grep -i -E "error|fail|nginx"
```

Fix the error, then restart and verify:

```bash
sudo sed -i '$d' /etc/nginx/nginx.conf      # remove the bad last line
sudo nginx -t                               # test config
sudo systemctl restart nginx
sudo systemctl is-active nginx && echo "nginx OK"
```

## Extra — Explore units and dependencies

```bash
systemctl list-units --type=service --state=running | head -n 20
systemctl cat nginx.service
systemctl list-dependencies nginx.service
```

Notes:
- `systemctl cat` shows the effective unit definition (including drop-ins).
- Use `nginx -t` before restarts to catch syntax errors early.