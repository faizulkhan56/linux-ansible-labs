# Linux Lab 04 — Networking tools

## Objective

- Inspect IP and routes
- Test connectivity and DNS
- Understand listening ports

## Step 1 — IP addresses and routes

```bash
ip a
ip route
```

## Step 2 — DNS checks

```bash
cat /etc/resolv.conf
getent hosts google.com
```

## Step 3 — Connectivity

```bash
ping -c 2 1.1.1.1
ping -c 2 google.com
```

## Step 4 — Ports and services

```bash
ss -lntp
sudo lsof -iTCP -sTCP:LISTEN -nP | head -n 20
```

## Step 5 — HTTP test (if nginx installed)

```bash
curl -I http://localhost | head -n 5
```

