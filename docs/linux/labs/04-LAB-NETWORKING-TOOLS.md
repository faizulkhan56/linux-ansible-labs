# Linux Lab 04 — Networking tools

## Objective

- Inspect IP addresses, routes, and interfaces
- Verify DNS resolution (`getent`, `dig`) and interpret **stub** vs **upstream** DNS (`/etc/resolv.conf`, `resolvectl`)
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

### Reading `ip route` output

`ip route` (same data as `route -n`, but clearer) shows **where the kernel sends packets** for each destination. More specific prefixes win over broader ones; **default** is the catch-all (equivalent to `0.0.0.0/0`).

Example (typical **VirtualBox**: `enp0s3` = NAT, `enp0s8` = host-only):

```text
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
8.8.4.4 via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
8.8.8.8 via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.105
```

Line by line:

| Line | Meaning |
|------|--------|
| `default via 10.0.2.2 dev enp0s3 …` | **Default route:** traffic to any destination that does not match a more specific route is sent to **next-hop gateway** `10.0.2.2`, using interface **`enp0s3`**. **`proto dhcp`** = this route was installed from DHCP. **`src 10.0.2.15`** = preferred source address on that interface. **`metric 100`** = lower is preferred when multiple routes could match the same kind of destination. |
| `8.8.4.4 via 10.0.2.2 dev enp0s3 …` | **Host route** to resolver `8.8.4.4` via the same gateway. Often appears when DHCP (or the network stack) publishes **per-address** or **classless static routes** so DNS traffic uses a known path. |
| `8.8.8.8 via 10.0.2.2 dev enp0s3 …` | Same idea for `8.8.8.8`. |
| `10.0.2.0/24 dev enp0s3 … scope link` | **Directly connected subnet** on `enp0s3`: hosts in `10.0.2.0/24` are reached **without** a router (on-link). **`proto kernel`** = installed automatically when the address was configured. |
| `10.0.2.2 dev enp0s3 … scope link` | **On-link path to the gateway** itself (`10.0.2.2` is treated as reachable on `enp0s3`). |
| `192.168.56.0/24 dev enp0s8 …` | Second NIC: **directly connected** host-only (or lab) network `192.168.56.0/24` via **`enp0s8`**, source `192.168.56.105`. No default route here — only that subnet is local on this interface unless you add routes. |

Check which route the kernel will use for a given destination:

```bash
ip route get 8.8.8.8
ip route get 192.168.56.1
```

### Adding a route

**Temporary (until reboot):** use `ip route add`. You must specify destination, and either `via <gateway>` or `dev <interface>` (for on-link subnets).

Examples (adjust IPs and interface names to your lab):

```bash
# Reach a remote subnet through a router on your host-only network (enp0s8)
sudo ip route add 172.16.0.0/16 via 192.168.56.1 dev enp0s8

# On-link subnet on a specific interface (no gateway)
sudo ip route add 10.99.0.0/24 dev enp0s8

# Optional second default via another interface (higher metric = lower priority)
sudo ip route add default via 192.168.56.1 dev enp0s8 metric 200
```

Verify and remove:

```bash
ip route
ip route get 172.16.0.1
sudo ip route del 172.16.0.0/16 via 192.168.56.1 dev enp0s8
```

**Persistent:** routes added with `ip route add` are lost on reboot. On Ubuntu, add them under **`routes:`** in Netplan (see Step 5).

---

## Step 2 — DNS: resolver, lookups, and `dig`

### Where the system looks for DNS

Most applications read **nameservers** from `/etc/resolv.conf` (or use the same settings through the C library resolver).

```bash
cat /etc/resolv.conf
```

On **Ubuntu and many Debian-based VMs**, you often see something like:

```text
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

- **`options edns0`** — allow EDNS0 extensions (larger responses, better behavior with modern DNS).
- **`trust-ad`** — trust AD (authenticated data) bit from the resolver when using DNSSEC-related features (details matter less for day-to-day labs).
- **`search .`** — search list for short hostnames; a lone `.` effectively means “do not append a domain” for bare names.

### Theory — stub resolver and `127.0.0.53`

**`127.0.0.53` is not a public DNS service** like Google (`8.8.8.8`) or Cloudflare (`1.1.1.1`). It is an address on **your own machine** (loopback). On systems with **systemd-resolved**, that address is a **local DNS stub**: a small listener that accepts queries from apps, then **forwards** them to the real upstream resolvers it learned from DHCP, static config, VPN, etc., and may **cache** answers.

Think of the path like this:

```text
Application  →  127.0.0.53 (systemd-resolved on loopback)  →  upstream DNS (router, ISP, 8.8.8.8, …)  →  answer back
```

So `/etc/resolv.conf` is often a **pointer to the local stub**, not the final server that talks to the internet. The **effective** upstream list is what you see in `resolvectl` (below).

### End-to-end flow (example: `ping google.com`)

1. You run `ping google.com`.
2. The resolver stack reads **`nameserver 127.0.0.53`** from `/etc/resolv.conf`.
3. The query is sent to **systemd-resolved** on the loopback stub.
4. **systemd-resolved** picks upstream DNS (from **DHCP**, Netplan, NetworkManager, VPN, manual config), uses **cache** when valid, and may apply **split DNS** per interface.
5. It forwards to the real resolver (e.g. what DHCP gave you), gets `google.com` → an A/AAAA record.
6. The answer returns to your application; `ping` then uses IP connectivity as usual.

### NAT (e.g. Oracle VirtualBox) — where DNS comes from

With **NAT**, the hypervisor usually runs a **virtual DHCP server** for the guest. DHCP typically supplies:

- IPv4 address and mask  
- **Default gateway**  
- **DNS server(s)** (on VirtualBox NAT, the guest often sees something like **`10.0.2.3`** as the DNS address)

The guest still may show **`nameserver 127.0.0.53`** in `/etc/resolv.conf`; **systemd-resolved** receives that DHCP information on the **NAT interface** and forwards queries **out** through NAT to the host / ISP / whatever the host uses. Conceptually:

```text
VM application  →  127.0.0.53  →  DNS from DHCP (NAT)  →  host / ISP / public resolver  →  reply
```

### See the real upstream servers: `resolvectl`

To see **which DNS servers systemd-resolved is actually using** (per link), run:

```bash
resolvectl status
```

Example shape (names and indexes vary):

```text
Link 2 (enp0s3)
    DNS Servers: 10.0.2.3
```

That **`10.0.2.3`**-style address is the **upstream** resolver for that interface (common on VirtualBox NAT), while **`127.0.0.53`** in `resolv.conf` remains the **local stub** entry point.

**Other setups:** If `systemd-resolved` is not used, `/etc/resolv.conf` may list `8.8.8.8` or your router directly, or be managed by **NetworkManager** / **cloud-init**. The idea is the same: the file tells apps **where to send DNS first**; that may be a stub or a full resolver.

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

### Add or keep extra routes (Netplan)

To make a route survive reboot, declare it on the correct interface. Example: **same machine as above**, send `172.16.0.0/16` via a router at `192.168.56.1` on `enp0s8` (merge into your existing `ethernets` stanza; do not duplicate the top-level `network:` key):

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      routes:
        - to: 172.16.0.0/16
          via: 192.168.56.1
```

You can combine **addresses**, **default** route, and **extra** `routes:` on the same interface. After editing:

```bash
sudo netplan try
ip route | grep 172.16
```

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
