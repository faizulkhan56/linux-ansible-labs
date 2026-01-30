# Linux Lab 05 — Logs and processes

## Objective

- Find processes and resource usage
- Understand logs for troubleshooting

## Step 1 — Processes

```bash
ps aux | head -n 10
top -b -n 1 | head -n 20
```

Find a process:

```bash
pgrep -a sshd || true
pgrep -a nginx || true
```

## Step 2 — System logs (journalctl)

```bash
journalctl --no-pager -n 50
```

Filter by service:

```bash
journalctl -u ssh --no-pager -n 50
```

## Step 3 — Disk usage

```bash
df -h
du -sh /var/log 2>/dev/null || true
```

## Step 4 — A practical troubleshooting loop

When “something is down”:
- Check service status: `systemctl status <service>`
- Check port: `ss -lntp | grep :PORT`
- Check logs: `journalctl -u <service> -n 100`

