# linux-ansible-labsetup

This repository contains **step-by-step lab guides** for:

- **Lab 1 (AWS)**: 2 Ubuntu VMs where **one VM is the Ansible control node (master)** and the other VM is a **managed node (slave)**.
- **Lab 2 (VirtualBox on Windows)**: 2 Ubuntu VMs on the **same network**, with **internet access**, **VM-to-VM connectivity**, and **host (Windows) ↔ VM SSH**.

It also includes **student practice labs** (markdown) for:
- **Ansible basics → intermediate**
- **Linux basics for DevOps**

## How to use

1. Start with the foundations (concepts and theory):
   - `docs/00-FOUNDATIONS.md`

2. Pick a lab environment:
   - **AWS lab**: `docs/aws/01-AWS-ANSIBLE-2VM-LAB.md`
   - **VirtualBox lab**: `docs/virtualbox/01-VIRTUALBOX-2VM-NETWORK-LAB.md`

3. Do the hands-on practice labs:
   - Ansible labs: `docs/ansible/labs/`
   - Linux labs: `docs/linux/labs/`

## Included Ansible example project (optional but recommended)

If you want a ready-to-run Ansible skeleton (inventory/config + small playbooks), see:

- `ansible/ansible.cfg`
- `ansible/inventory.ini`
- `ansible/playbooks/`

> You can use this folder in **both** AWS and VirtualBox labs—just update the IPs/hostnames in `ansible/inventory.ini`.

## Quick links

- **Foundations / theory**: `docs/00-FOUNDATIONS.md`
- **AWS**: `docs/aws/01-AWS-ANSIBLE-2VM-LAB.md`
- **VirtualBox**: `docs/virtualbox/01-VIRTUALBOX-2VM-NETWORK-LAB.md`
- **Ansible labs index**: `docs/ansible/labs/00-INDEX.md`
- **Linux labs index**: `docs/linux/labs/00-INDEX.md`

