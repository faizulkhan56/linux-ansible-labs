# AWS Lab Checklist (Quick Reference)

Use this checklist during class so students don’t miss steps.

## Build

- [ ] Create Security Group: inbound SSH 22 from **My IP**
- [ ] Create Key Pair and download `.pem`
- [ ] Launch EC2 `ansible-master` (Ubuntu 22.04/24.04)
- [ ] Launch EC2 `ansible-slave1` (same)
- [ ] Note **master public IP**, **slave public IP**, **slave private IP**

## Access

- [ ] SSH from Windows to master:
  - `ssh -i C:\keys\ansible-lab-key.pem ubuntu@<MASTER_PUBLIC_IP>`
- [ ] SSH from Windows to slave:
  - `ssh -i C:\keys\ansible-lab-key.pem ubuntu@<SLAVE_PUBLIC_IP>`

## Prepare

- [ ] Update packages on both:
  - `sudo apt-get update && sudo apt-get -y upgrade`
- [ ] Install Ansible on master:
  - `sudo apt-get install -y ansible`

## SSH from master → slave

Choose one:

### Option A (reuse EC2 key)

- [ ] Copy key to master (scp)
- [ ] `chmod 600 /home/ubuntu/ansible-lab-key.pem`
- [ ] `ssh -i /home/ubuntu/ansible-lab-key.pem ubuntu@<SLAVE_PRIVATE_IP>`

### Option B (dedicated lab key)

- [ ] `ssh-keygen -t ed25519 -f ~/.ssh/aws_lab_ed25519`
- [ ] Append public key to slave `authorized_keys`
- [ ] Test: `ssh -i ~/.ssh/aws_lab_ed25519 ubuntu@<SLAVE_PRIVATE_IP>`

## Ansible test

- [ ] Create `inventory.ini`
- [ ] Run ping:
  - `ansible -i inventory.ini aws_slaves -m ping`

## Cleanup

- [ ] Terminate EC2 instances

