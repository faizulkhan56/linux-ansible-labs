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

### Crypto basics for SSH keys (theory)

- Symmetric (shared-key) crypto: one key does both encrypt and decrypt. Fast, but both sides must share the same secret (hard to distribute securely).
- Asymmetric (public/private) crypto: a keypair where the public key encrypts/validates and the private key decrypts/signs. You can share the public key safely; keep the private key secret. SSH uses this to authenticate without passwords.
- RSA, ED25519, ECDSA: different asymmetric algorithms. Modern recommendation is ED25519 (smaller keys, fast, strong). RSA is older, widely compatible, uses larger keys (e.g., 2048/4096 bits).
- SSH login flow (public-key auth):
  1) Client proves it has the private key by signing a server-provided challenge.
  2) Server verifies the signature using the user’s public key stored in `~/.ssh/authorized_keys`.
  3) If valid, access is granted without sending your private key or passphrase over the network.

Symmetric key workflow (at a glance)

1) Both sides pre-share the same secret key (out of band).
2) Sender encrypts data with the shared key.
3) Receiver decrypts using the same key.
4) Challenge: securely distributing and rotating the shared key at scale.

Asymmetric key workflow (SSH, at a glance)

1) Generate a keypair: public key (shareable) + private key (keep secret).
2) Place the public key on the server in `~/.ssh/authorized_keys`.
3) Client starts SSH; server sends a random challenge.
4) Client signs the challenge with its private key (never sent over network).
5) Server verifies the signature with the stored public key; if valid, grants access.

Symmetric workflow diagram (concept)

```text
[Client]                      [Server]
   |                              |
   |  (1) Share secret key K  ---->  (out-of-band/trusted channel)
   |                              |
   |  (2) plaintext               |
   |      Encrypt_K(plaintext)    |
   |  ------------------------->  |  (3) Decrypt_K(ciphertext) -> plaintext
   |         ciphertext           |
   |                              |
```

Asymmetric (SSH) login diagram (concept)

```text
[Client: has private key SK]                [Server: has public key PK in ~/.ssh/authorized_keys]
          |                                                        |
 (1) SSH connect (protocol hello)  ------------------------------> |
          |                                                        |
          | <------------------------------  (2) Random challenge  |
          |                                                        |
 (3) Sign challenge with SK                                        |
     send signature --------------------->  (4) Verify(signature, PK)
          |                                                        |
          | <----------------------  (5) If valid, grant access    |
          |                                                        |
  (6) Session encryption negotiated (e.g., Diffie–Hellman) for the channel
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

Command breakdown (Windows `ssh-keygen`)

- `-t ed25519`: choose key type/algorithm (ED25519 recommended)
- `-C "comment"`: add a label to help identify the key (e.g., email)
- Default path: stores private key as `id_ed25519` and matching public `.pub` file
- Passphrase: encrypts the private key at rest (adds a layer if the file is stolen)

### Create an SSH key on Ubuntu (alternative)

On any Linux/Ubuntu client (or on the VM if you’re making a lab key there):

```bash
ssh-keygen -t ed25519 -C "lab-key" -f ~/.ssh/lab_ed25519
```

Command breakdown (Linux `ssh-keygen`)

- `-t ed25519`: generate an ED25519 keypair
- `-C "lab-key"`: comment for identification (arbitrary text)
- `-f ~/.ssh/lab_ed25519`: output file path/prefix (private key), `.pub` holds the public key
- You will be prompted for a passphrase (recommended)

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

