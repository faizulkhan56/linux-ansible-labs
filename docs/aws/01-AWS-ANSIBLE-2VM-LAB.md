# AWS Lab: 2 Ubuntu VMs (Ansible Master + Slave)

Goal: Create **two Ubuntu servers on AWS**:
- **VM1 (master/control node)**: install Ansible here
- **VM2 (slave/managed node)**: Ansible connects here using SSH

This guide is written so you can follow it later step-by-step.

## Prerequisites

- AWS account + permission to create EC2 instances + security groups + key pairs
- A basic understanding of SSH
- Recommended region: any (choose close to you)

## Naming (use consistent names)

- EC2 #1: `ansible-master`
- EC2 #2: `ansible-slave1`
- Security group: `sg-ansible-lab`
- Key pair: `ansible-lab-key`

## Step 1 — Create a security group

In AWS Console → EC2 → **Security Groups** → Create security group:

**Inbound rules**
- **SSH (22)** from **Your IP** (recommended)  
  Source: `My IP` (or your office public IP)
- (Optional for web labs) **HTTP (80)** from `0.0.0.0/0`

**Outbound rules**
- Keep default “All traffic” outbound allowed

> Security note: avoid opening SSH to `0.0.0.0/0` unless you really must.

## Step 2 — Create a Key Pair

EC2 → Key Pairs → Create key pair:
- Name: `ansible-lab-key`
- Type: **ED25519** (or RSA if needed)
- Format:
  - Windows: `.pem` works with OpenSSH; `.ppk` if using PuTTY

Download and store safely.

## Step 3 — Launch EC2 instance #1 (master)

EC2 → Instances → Launch instance:

- Name: `ansible-master`
- AMI: **Ubuntu Server 22.04 LTS** or **Ubuntu 24.04 LTS** (latest stable is fine)
- Instance type: `t2.micro` or `t3.micro` (free tier depends on account)
- Key pair: `ansible-lab-key`
- Network:
  - Auto-assign public IPv4: **Enable**
  - Security group: select `sg-ansible-lab`
- Storage: default is OK (8–16 GB)

Launch.

## Step 4 — Launch EC2 instance #2 (slave)

Repeat Step 3 with:
- Name: `ansible-slave1`
- Same AMI and type
- Same key pair
- Same security group

## Step 5 — Collect IP addresses

For each instance, note:
- **Public IPv4 address** (for SSH from your PC)
- **Private IPv4 address** (for internal VPC)

We will mainly use:
- SSH from your PC → public IPs
- Ansible master → slave: can use **private IP** (preferred) because they are in same VPC

## Step 6 — SSH into the master from your computer

### Windows PowerShell (OpenSSH)

1. Move your key somewhere safe, e.g. `C:\keys\ansible-lab-key.pem`
2. Fix permissions (important on Windows sometimes):
   - If SSH complains, keep the key in a protected folder and ensure it’s not accessible by “Everyone”.

Connect:

```powershell
ssh -i C:\keys\ansible-lab-key.pem ubuntu@<MASTER_PUBLIC_IP>
```

If you selected a different Ubuntu AMI, the username is usually:
- Ubuntu: `ubuntu`

## Step 7 — Update Ubuntu packages (master + slave)

On **master**:

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

Do the same on **slave** (you can SSH from your PC to slave public IP):

```powershell
ssh -i C:\keys\ansible-lab-key.pem ubuntu@<SLAVE_PUBLIC_IP>
```

Then:

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

## Step 8 — Install Ansible on the master

On **master**:

```bash
sudo apt-get update
sudo apt-get install -y ansible
ansible --version
```

> If `ansible` package is not found (rare on very new releases), install with:
>
> - `sudo apt-get install -y python3-pip`
> - `python3 -m pip install --user ansible`
> - Ensure `~/.local/bin` is in PATH.

## Step 9 — Ensure master can SSH to slave (key-based)

### Option A (recommended): reuse the EC2 key pair

If both instances were created with the same key pair, the master can use the same private key file to SSH to the slave.

1. Copy the private key to master (temporary for lab only)

From your Windows machine:

```powershell
scp -i C:\keys\ansible-lab-key.pem C:\keys\ansible-lab-key.pem ubuntu@<MASTER_PUBLIC_IP>:/home/ubuntu/ansible-lab-key.pem
```

2. On master: fix permissions

```bash
chmod 600 /home/ubuntu/ansible-lab-key.pem
```

3. SSH from master to slave using **private IP** (preferred)

```bash
ssh -i /home/ubuntu/ansible-lab-key.pem ubuntu@<SLAVE_PRIVATE_IP>
```

Type `exit` to return to master.

### Option B (better practice): create a dedicated lab key on master

On master:

```bash
ssh-keygen -t ed25519 -C "aws-ansible-lab" -f ~/.ssh/aws_lab_ed25519
```

Copy the public key to the slave:

```bash
cat ~/.ssh/aws_lab_ed25519.pub | ssh -i /home/ubuntu/ansible-lab-key.pem ubuntu@<SLAVE_PRIVATE_IP> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Test:

```bash
ssh -i ~/.ssh/aws_lab_ed25519 ubuntu@<SLAVE_PRIVATE_IP>
```

## Step 10 — Create an inventory on the master

On master:

```bash
mkdir -p ~/ansible-lab
cd ~/ansible-lab
```

Create `inventory.ini`:

```ini
[aws_slaves]
slave1 ansible_host=<SLAVE_PRIVATE_IP>

[aws_slaves:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/ansible-lab-key.pem
ansible_become=true
```

> If you used Option B (dedicated lab key), set `ansible_ssh_private_key_file=~/.ssh/aws_lab_ed25519`.

## Step 11 — First Ansible test (ping module)

From master:

```bash
ansible -i inventory.ini aws_slaves -m ping
```

Expected result:
- `SUCCESS => {"ping": "pong"}`

If it fails, see troubleshooting below.

## Step 12 — First playbook: install nginx on the slave

Create `01-nginx.yml`:

```yaml
- name: Install nginx on AWS slave
  hosts: aws_slaves
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run it:

```bash
ansible-playbook -i inventory.ini 01-nginx.yml
```

## Step 13 — Verify nginx

SSH to slave from your PC and check:

```bash
systemctl status nginx --no-pager
curl -I http://localhost
```

If you opened port 80 in the security group, you can also curl from your PC:

```powershell
curl http://<SLAVE_PUBLIC_IP>
```

## Troubleshooting (common)

### “Permission denied (publickey)”

- Wrong username (must be `ubuntu` for Ubuntu AMIs)
- Wrong key file in inventory
- Key file permissions too open (`chmod 600`)

### “UNREACHABLE! Failed to connect to the host via ssh”

- Security group inbound missing port 22
- You are using slave’s **private IP** but instances are not in same VPC/subnet or routing is wrong (rare in a basic lab)
- Slave does not have SSH running:
  - `sudo systemctl status ssh`

### “sudo: a password is required”

On Ubuntu cloud images, `ubuntu` usually has passwordless sudo.
If not, set up proper sudoers or use a different user.

## Cleanup (important to avoid charges)

When done:
- Terminate EC2 instances `ansible-master` and `ansible-slave1`
- Delete security group if not needed
- Delete key pair if this was only for training

