# Lab 01 examination (Part 01) — answer sheet / solution guide

This document is a **reference solution** for instructors and learners checking their work after attempting [01-LAB-EXPERIMENT-PROJECT-LINUX-part01.md](./01-LAB-EXPERIMENT-PROJECT-LINUX-part01.md). Replace example names (`webadmin`, `team_site`, paths) with your own choices where the brief allows it.

**Security:** Do not paste real passwords, private keys, or full `authorized_keys` into submissions.

---

## Phase 1 — Administrative user (access model)

### Valid approach

1. Create a **new** user (not your daily account, not `root`), e.g. `webadmin`:

   ```bash
   sudo adduser webadmin
   ```

2. Grant **Ubuntu-style sudo** by adding the user to the **`sudo`** group:

   ```bash
   sudo usermod -aG sudo webadmin
   ```

   Alternatively, during `adduser` you can be prompted to make the user an admin depending on image; always verify with `groups`.

3. **Verify** (log in as `webadmin` in a second session or use `su - webadmin` for testing):

   ```bash
   id webadmin
   groups webadmin
   ```

   **Expected:** `webadmin` appears in output; `sudo` is listed among groups.

4. **Prove sudo works** (as `webadmin`):

   ```bash
   sudo -v
   sudo true && echo ok
   ```

### Evidence (screenshot) checklist

- `id webadmin` and/or `groups webadmin` showing **`sudo`**.
- No password echoed; no private key material.

---

## Phase 2 — Site directory and permissions

### Example naming

- **Group:** e.g. `team_site` (avoid system names like `root`, `sudo`, `adm`).
- **Directory:** e.g. `~/site-content` (under **your** home—the brief says “under your home directory”).

### Valid approach

```bash
# As your normal user (directory owner)
sudo groupadd team_site
sudo usermod -aG team_site "$USER"          # optional: if YOU need group access for testing
newgrp team_site                            # or log out/in to refresh groups

mkdir -p "$HOME/site-content"
echo "placeholder for future static site" > "$HOME/site-content/README.txt"

sudo chgrp -R team_site "$HOME/site-content"
chmod 750 "$HOME/site-content"
chmod 640 "$HOME/site-content/README.txt"
```

**Permission policy check**

| Path | Owner | Group | Mode | Effect |
|------|--------|--------|------|--------|
| `site-content/` | you | `team_site` | `750` (`rwxr-x---`) | You: full; group: enter + list; others: **no** access |
| `README.txt` | you | `team_site` | `640` (`rw-r-----`) | You: rw; group: **read**; others: **no** read |

If the brief requires group members to read **new** files you create later without `chgrp` each time, set **setgid** on the directory:

```bash
chmod 2770 "$HOME/site-content"
```

(`2770` = setgid + `rwxrws---`; new files inherit group `team_site` if your umask allows group read—often set file default to `664` and umask `002` for a shared tree; for this exam one file with explicit `chgrp`/`chmod` is enough.)

### Evidence (screenshot) checklist

- `ls -ld ~/site-content` and `ls -l ~/site-content/README.txt` (or `stat`) showing **user**, **group**, and **mode** clearly.

---

## Phase 3 — nginx package; running now and enabled at boot

### Valid approach

```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Evidence (screenshots) — make “now” vs “boot” obvious

**Running now** (pick one or both):

```bash
systemctl status nginx --no-pager
```

Look for **Active: active (running)**.

```bash
systemctl is-active nginx
```

Expected: `active`.

**Enabled at boot:**

```bash
systemctl is-enabled nginx
```

Expected: `enabled` (or `enabled-runtime` in unusual setups; `enabled` is normal).

**Optional** — shows the symlink the brief mentions in Phase 6:

```bash
ls -l /etc/systemd/system/multi-user.target.wants/nginx.service
```

### Evidence checklist

- One capture showing **active (running)** (or `is-active` → `active`).
- Another showing **`enabled`** for boot (or the `multi-user.target.wants` symlink).

---

## Phase 4 — Recent logs for the nginx **unit**

### Valid approach

Use **journald** scoped to the unit name the package provides (usually `nginx.service` / unit `nginx`):

```bash
journalctl -u nginx -n 30 --no-pager
```

Or with sudo if your user lacks journal access:

```bash
sudo journalctl -u nginx -n 30 --no-pager
```

### Evidence checklist

- Timestamps or log lines visible; unit context clear (`-u nginx`).

---

## Phase 5 — Network facts appendix (template + how to gather)

Facts **must match your VM**. Use these commands, then write the half-page in prose.

```bash
ip -4 addr show
ip route show default
cat /etc/resolv.conf
resolvectl status          # systemd-resolved: shows per-link DNS and stub behavior
```

### Sample appendix structure (replace ALL bracketed text)

**Title:** Network facts for this VM.

- **Interfaces / IPv4:** Primary interface **`[e.g. enp0s3]`** has address **`[e.g. 10.0.2.15/24]`** (from `ip -4 addr`).
- **Default route:** **`[yes/no]`**; if yes, default gateway **`[e.g. 10.0.2.2]`** via **`[e.g. enp0s3]`** (from `ip route show default`).
- **`/etc/resolv.conf`:** Nameserver line(s) often **`nameserver 127.0.0.53`** on Ubuntu (local stub).
- **Upstream DNS:** From **`resolvectl status`**, link **`[e.g. enp0s3]`** shows **DNS Servers:** **`[e.g. 192.168.1.1 or ISP addresses]`** — that is upstream from the VM’s perspective; the stub at 127.0.0.53 forwards there.
- **Routing paragraph (example):** Traffic to addresses not on attached subnets is sent to the **default gateway** on the interface that owns the default route. Traffic to the VM’s own subnet is delivered on-LAN without using the default route. The gateway then forwards toward the public internet (NAT or provider routing).

---

## Phase 6 — Concept memo (model answers)

*Learners must paraphrase in their own words; below is factually correct reference prose.*

### 1) systemd: start vs enable; `multi-user.target.wants`

**Starting** a unit (e.g. `systemctl start nginx`) runs the service **now** for the current boot. It does **not** record “start this every boot” by itself.

**Enabling** a unit (e.g. `systemctl enable nginx`) creates a **persistent symlink** so systemd will start the unit when the **multi-user** target is reached (normal server/desktop login mode). On typical packages you will see something like:

`/etc/systemd/system/multi-user.target.wants/nginx.service` → `…/nginx.service`

That **want** is what ties the service to **boot-time** (multi-user) startup. Merely **starting** the service does not create that symlink, so after a reboot the service would stay **stopped** unless something else starts it or it is enabled.

### 2) DNS: `127.0.0.53` is not “Google DNS” by itself

On many Ubuntu systems **`systemd-resolved`** listens on **`127.0.0.53`** and **`/etc/resolv.conf`** points applications to that **stub** address. That only means “this machine’s **local** resolver is handling the query,” not which **upstream** resolvers (Google `8.8.8.8`, Cloudflare, your router, etc.) are used.

**Upstream** DNS is shown in tools such as **`resolvectl status`** (per-link **DNS Servers**), or in DHCP/network manager configuration—not inferred from the single line `nameserver 127.0.0.53` alone.

---

## Quick evaluation crosswalk

| Phase | Solution satisfies |
|-------|---------------------|
| 1 | Second user + `sudo` group + evidence without secrets |
| 2 | New group; `~/…` dir + one file; `750`/`640` (or equivalent) meeting owner/group/other rules |
| 3 | `nginx` from packages; separate proof of **active** and **enabled** |
| 4 | `journalctl -u nginx` (or equivalent unit-scoped journal) |
| 5 | Appendix consistent with `ip`, routes, `resolv.conf`, `resolvectl` |
| 6 | Correct distinction start/enable/wants; stub vs upstream DNS |

---

*End of solution guide.*
