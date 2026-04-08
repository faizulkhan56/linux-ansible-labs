# Linux Lab 04 — Networking tools

## Objective

- Inspect IP addresses, routes, and interfaces
- Verify DNS resolution (`getent`, `dig`)
- Test reachability (ping, traceroute)
- See which ports are listening and which process owns them (`ss`, `netstat`)
- Change or assign a static IP safely with **Netplan** (Ubuntu-style hosts)
- Spot-check HTTP from the command line

Commands below assume **Debian/Ubuntu** (`apt`). Adjust packages if you use another distribution.

---

## Step 1 — IP addresses and routes

See addresses on all interfaces and the default route:

```bash
ip a
ip route
```

Useful shortcuts:

```bash
ip -br a                    # brief one-line per interface
hostname -I                 # primary IP(s) reported by the host
```

Find the **interface name** you will use later (for example `eth0`, `ens33`, `enp0s3`) — you need it for Netplan.

---

## Step 2 — DNS: resolver, lookups, and `dig`

### Where the system looks for DNS

```bash
cat /etc/resolv.conf
```

On many systems this file is managed automatically (systemd-resolved, NetworkManager, or cloud-init). Use it to see which nameservers are in effect.

### libc-based lookup (what most apps use)

```bash
getent hosts google.com
```

### `dig` — query DNS directly

`dig` talks to a DNS server and shows the full answer (useful for debugging TTL, CNAME chains, and which server answered).

Install on Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y dnsutils
```

Examples:

```bash
dig google.com                    # A record (default)
dig google.com AAAA               # IPv6
dig @1.1.1.1 google.com           # ask a specific resolver
dig -x 8.8.8.8                    # reverse (PTR) lookup
```

Related (often installed with the same tooling):

```bash
nslookup google.com
host google.com
```

Optional: compare with `/etc/hosts` (static overrides):

```bash
grep -v '^#' /etc/hosts | grep -v '^$'
```

---

## Step 3 — Connectivity

### ICMP ping

```bash
ping -c 2 1.1.1.1
ping -c 2 google.com
```

### Path to a host (where packets go)

Install traceroute if needed:

```bash
sudo apt install -y traceroute
traceroute -n 1.1.1.1
```

`-n` avoids slow reverse-DNS lookups on each hop. Use this when ping works but a service on a port does not — it tells you whether the problem is on the path or local (firewall/listen address).

---

## Step 4 — Listening ports: `ss` (preferred) and `netstat`

### Modern default: `ss`

Socket statistics from `iproute2` (usually pre-installed):

```bash
ss -lntp
```

Typical flags:

- `-l` — listening sockets only  
- `-n` — numeric (no DNS for addresses/ports)  
- `-t` — TCP  
- `-p` — show process (often needs `sudo` to see all PIDs)

```bash
sudo ss -lntp
sudo lsof -iTCP -sTCP:LISTEN -nP | head -n 20
```

### Legacy: `netstat -tulpn`

`netstat` is the older tool many tutorials still mention. On Debian/Ubuntu it comes from the **net-tools** package:

```bash
sudo apt update
sudo apt install -y net-tools
```

Example:

```bash
sudo netstat -tulpn
```

What the flags mean:

| Flag | Meaning |
|------|--------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | **Listening** (servers waiting for connections) |
| `-n` | **Numeric** — show IPs and port numbers instead of resolving names |
| `-p` | **Program** — show PID and process name (requires root for full detail) |

So **`netstat -tulpn`** = “show listening TCP and UDP ports numerically, with owning process.”

**Note:** Prefer `ss` on new systems; `netstat` is maintained mainly for familiarity. Both answers should be consistent for listening ports.

### Established connections (optional)

See active sessions, not only listeners:

```bash
sudo ss -antp | head -n 20
```

---

## Step 5 — Change or set a static IP with Netplan

**Netplan** is the YAML-based network layer used on **Ubuntu Server** and many cloud images. Configuration lives under `/etc/netplan/` (for example `50-cloud-init.yaml` or `01-netcfg.yaml`).

**Before you edit:** make sure you know the correct interface name (`ip -br a`), gateway, subnet mask (CIDR), and DNS. A wrong gateway or typo can **drop SSH access** — use a console or schedule a revert if possible.

### Safer apply

```bash
sudo netplan try
```

This applies the config and asks you to confirm; it rolls back if you do not.

### Typical workflow

1. List config files:

   ```bash
   ls /etc/netplan/
   ```

2. Back up, then edit **one** YAML file (spacing matters in YAML):

   ```bash
   sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
   sudo nano /etc/netplan/50-cloud-init.yaml
   ```

3. Example **static IPv4** (replace `enp0s3`, addresses, and gateway with yours):

   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: false
         addresses:
           - 192.168.1.50/24
         routes:
           - to: default
             via: 192.168.1.1
         nameservers:
           addresses:
             - 8.8.8.8
             - 1.1.1.1
   ```

4. Validate and apply:

   ```bash
   sudo netplan generate
   sudo netplan try
   # or: sudo netplan apply
   ```

5. Verify:

   ```bash
   ip -br a
   ip route
   ping -c 2 1.1.1.1
   ```

**DHCP again:** set `dhcp4: true` and remove static `addresses` / `routes` / `nameservers` as needed, then `sudo netplan apply`.

---

## Step 6 — HTTP check (if a web server is installed)

```bash
curl -I http://localhost | head -n 5
```

`-I` fetches headers only — quick check that something is listening on port 80.

---

## Extra — Other useful networking checks

| Goal | Command / note |
|------|----------------|
| ARP / neighbor cache | `ip neigh` |
| Firewall (Ubuntu `ufw`) | `sudo ufw status verbose` |
| Open a test port with `nc` | `sudo apt install -y netcat-openbsd` then e.g. `nc -vz host 443` |
| Bandwidth / stats per interface | `ip -s link` |
| TLS / cert check | `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \| openssl x509 -noout -dates` |

Work through Steps 1 → 6 in order: observe addresses and DNS, confirm connectivity, inspect listeners, then only change Netplan when you have correct values and a recovery path.
