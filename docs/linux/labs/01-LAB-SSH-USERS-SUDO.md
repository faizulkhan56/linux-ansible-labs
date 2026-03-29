# Linux Lab 01 — SSH, users, and sudo

## Objective

- Confirm SSH access
- Create a new user
- Give sudo access safely

## Before you start — Install and enable OpenSSH server (Ubuntu)

On the Ubuntu machine (the server/VM you will SSH into):

```bash
sudo apt-get update
sudo apt-get install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

If a firewall is enabled (e.g., UFW), allow SSH:

```bash
sudo ufw allow OpenSSH
sudo ufw enable    # if ufw was disabled; confirm before enabling on remote systems
sudo ufw status
```

## Step 1 — SSH into the machine

From another machine (Windows PowerShell or another Linux host):

```powershell
ssh <user>@<ip>
```

### If you don't have an SSH key on Windows (generate one)

In Windows PowerShell (OpenSSH client):

```powershell
ssh -V   # confirm client exists; install via Windows Optional Features if missing
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- Press Enter to accept default path: `C:\Users\<you>\.ssh\id_ed25519`
- Optionally set a passphrase (recommended)

Your files:

- Private key: `C:\Users\<you>\.ssh\id_ed25519`
- Public key:  `C:\Users\<you>\.ssh\id_ed25519.pub`

## Step 2 — Check current user and sudo access

```bash
whoami
id
sudo -n true && echo "sudo OK (passwordless)" || echo "sudo requires password"
```

## Step 3 — Create a DevOps user

```bash
sudo useradd -m -s /bin/bash devops
sudo passwd devops
```

## Step 4 — Add user to sudo group (Ubuntu)

```bash
sudo usermod -aG sudo devops
```

Verify:

```bash
getent group sudo | grep devops || true
```

### Copy your Windows public key to the Linux user's authorized_keys

Option A — One-liner (pipe public key directly to server):

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@<ip> "umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

- Replace `<user>` with the target Linux account (e.g., `devops`)
- Ensures `~/.ssh` exists and appends your public key safely

Option B — Two-step (scp then append on server):

```powershell
scp $env:USERPROFILE\.ssh\id_ed25519.pub <user>@<ip>:/tmp/pubkey.pub
ssh <user>@<ip> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat /tmp/pubkey.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm -f /tmp/pubkey.pub"
```

Permissions reference (on Linux target):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## Step 5 — SSH as the new user (optional)

From your admin account, set up SSH key login for `devops`:

```bash
sudo mkdir -p /home/devops/.ssh
sudo chmod 700 /home/devops/.ssh
sudo cp ~/.ssh/authorized_keys /home/devops/.ssh/authorized_keys
sudo chmod 600 /home/devops/.ssh/authorized_keys
sudo chown -R devops:devops /home/devops/.ssh
```

Then from your client:

```powershell
ssh devops@<ip>
```

## Step 6 — Sudo best practice note

- Prefer least privilege in real systems.
- For training labs, `sudo` group is okay.

