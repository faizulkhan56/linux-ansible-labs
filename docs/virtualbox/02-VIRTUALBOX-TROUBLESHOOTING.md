# VirtualBox Lab Troubleshooting (Windows Host)

## 1) VM has no internet

Checklist:
- Adapter 1 is **NAT Network** (not “NAT” unless you intentionally used NAT)
- NAT Network `LabNAT` exists and **DHCP is enabled**
- In Ubuntu: `ip route` shows a default route via the NAT interface

Commands:

```bash
ip a
ip route
ping -c 2 1.1.1.1
```

If `ping 1.1.1.1` works but `ping google.com` fails:
- DNS issue. Check `/etc/resolv.conf` and netplan config.

## 2) Windows cannot SSH to the VM

Checklist:
- VM has a **host-only IP** (e.g. `192.168.56.101`)
- Windows host-only adapter exists and is in the same subnet
- SSH server is running:
  - `sudo systemctl status ssh --no-pager`
- Firewall on Ubuntu is not blocking port 22:
  - `sudo ufw status`

Test from Windows:

```powershell
Test-NetConnection 192.168.56.101 -Port 22
```

## 3) VM1 cannot ping VM2 over host-only

- Ensure both VMs have Adapter 2 set to the **same host-only network** (e.g. `vboxnet0`)
- Ensure static IPs are correct and unique
- Check netplan config and `sudo netplan apply`

## 4) Netplan changes break networking

Recovery:

```bash
sudo netplan try
```

If network is lost, you may need VirtualBox console access to revert the YAML file.

Tip: Keep NAT interface as DHCP, only set host-only to static.

## 5) “No route to host” or “Connection timed out”

- Usually wrong IP/subnet, wrong adapter mode, or SSH not running.
- Check:
  - `ip a`
  - `sudo systemctl status ssh --no-pager`
  - VirtualBox adapter “Cable connected” enabled

