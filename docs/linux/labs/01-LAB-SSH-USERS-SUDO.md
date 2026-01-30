# Linux Lab 01 — SSH, users, and sudo

## Objective

- Confirm SSH access
- Create a new user
- Give sudo access safely

## Step 1 — SSH into the machine

From another machine (Windows PowerShell or another Linux host):

```powershell
ssh <user>@<ip>
```

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

