# Linux Lab 03 — Packages + services (systemd)

## Objective

- Install packages with `apt`
- Manage services with `systemctl`
- Understand enabled vs started

## overview

1) Real-life hook

When your server restarts… how does NGINX automatically come back online?

There must be:

- something that installs software → APT
- something that runs and manages it → systemd
- something that logs what happened → journal (journalctl)

2) Big picture (architecture in words)

Package (nginx)
  ↓ installed via apt
Service (nginx.service)
  ↓ managed by systemd
Logs (journalctl)

One line: APT installs software, systemd runs it, journalctl helps you debug it.

3) APT — why?

- Concept: manages installation, dependencies, and updates.
- Example:

```bash
sudo apt update
sudo apt install nginx
```

- Internally: downloads the package, installs binaries, and usually drops a service unit file (e.g., nginx.service) so systemd can manage it.

Teaching line: APT brings software into the system.

4) systemd — why?

- Ask: After installing nginx… who starts it? → systemd
- Concept: init system (PID 1), controls services and boot process.
- Examples:

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl status nginx
```

- Started vs Enabled (very important):
  - started → running now: `sudo systemctl start nginx`
  - enabled → will run after reboot: `sudo systemctl enable nginx`

Teaching line: Start = now, Enable = after reboot.

5) journalctl — why?

- Ask: If nginx fails… how do you debug? → logs
- Concept: systemd collects logs from services and kernel, stored centrally.
- Examples:

```bash
journalctl -u nginx
journalctl -u nginx -f   # live logs
```

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

## Unit files (bridge concept)

Every service has a unit file that tells systemd how to start/stop it, restart policy, dependencies, etc.

- Example (apt-installed nginx):
  - Unit path (distro-provided): `/lib/systemd/system/nginx.service`
  - May also have drop-ins under `/etc/systemd/system/nginx.service.d/*.conf`

Teaching line: systemd reads unit files to know how to run services.

### Compare: apt-installed service vs custom .NET Core service

- Apt-installed service (e.g., nginx)
  - Installed by `apt`
  - Unit file usually under `/lib/systemd/system/`
  - Managed with `systemctl start|enable nginx`
  - Logs with `journalctl -u nginx`

- Custom .NET Core service you deploy
  - You place your app (e.g., `/opt/myapp/MyApi.dll`)
  - You create your own unit file under `/etc/systemd/system/myapi.service`
  - You control how it starts via `ExecStart`
  - You must run `sudo systemctl daemon-reload` after creating/updating the unit

Example `myapi.service`:

```ini
[Unit]
Description=My API (.NET)
After=network.target

[Service]
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/dotnet /opt/myapp/MyApi.dll
Restart=always
RestartSec=5
User=www-data
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000

[Install]
WantedBy=multi-user.target
```

Lifecycle for the custom service:

```bash
sudo cp myapi.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable myapi
sudo systemctl start myapi
sudo systemctl status myapi --no-pager
journalctl -u myapi -f
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