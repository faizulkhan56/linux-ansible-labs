# Lab 02 — Command module basics (ad-hoc + playbook)

## Objective

- Understand what the **`command`** module does and when to use it instead of **`shell`**
- Run common **read-only checks** and simple **file operations** with ad-hoc Ansible
- Mirror the same ideas in a **playbook** using `command:`, **`register`**, and **`debug`**
- Practice with a real **`inventory.ini`** and `ansible-playbook`

**Prerequisite:** Complete `01-LAB-PING-AND-INVENTORY.md` and `02-LAB-ADHOC-COMMANDS.md` (inventory paths and `nodes` vs `all` patterns).

---

## Step 0 — Lab folder and inventory

Use the same layout as earlier labs (or the repo’s `ansible/` folder).

```bash
mkdir -p ~/ansible-labs/lab02-command
cd ~/ansible-labs/lab02-command
```

Copy or create **`inventory.ini`**. You can start from the example below (VirtualBox-style); adjust `ansible_host`, `ansible_user`, and key paths for AWS or your class network.

**Example `inventory.ini`** (matches the runnable copy in this repo at `ansible/inventory-command-lab.ini`):

```ini
[nodes]
node1 ansible_host=192.168.56.101
node2 ansible_host=192.168.56.102

[nodes:vars]
ansible_user=ubuntu
ansible_become=true
ansible_become_method=sudo
# ansible_ssh_private_key_file=~/.ssh/your_lab_key
```

**Host pattern:** These labs use the group **`nodes`** for managed servers. If your file only defines `[web]`, replace `nodes` in the commands below with `web` (or use `all` if you have no `localhost` control entry mixed in).

---

## Theory — What is the `command` module?

- **`command`** runs a program on the remote host **without** going through `/bin/sh` (no shell).
- **Why it matters:** No shell expansion of `*`, `|`, `>`, `&&`, etc. unless you use a shell wrapper. That reduces surprises and injection risk for simple invocations.
- **“Safe direct execution”:** Ansible splits the executable and arguments; use it when a single binary + argv is enough (`uptime`, `hostname`, `df`, …).
- **Default module:** If you omit **`-m`** on the `ansible` CLI, Ansible uses **`command`** for the string you pass with **`-a`**.

**When you need `shell` instead:** Pipelines (`|`), redirects, `&&` / `;`, or environment expansion that requires a shell — see `02-LAB-ADHOC-COMMANDS.md` Step 4.

---

## Part 1 — Ad-hoc `command` module examples

Run from the directory that contains **`inventory.ini`**. Replace **`nodes`** with your group name if different.

### 1. Check uptime

```bash
ansible -i inventory.ini nodes -m command -a "uptime"
```

**Meaning:** Run the `uptime` binary on every host in `nodes` and print stdout.

### 2. Check hostname

```bash
ansible -i inventory.ini nodes -m command -a "hostname"
```

### 3. Check disk usage

```bash
ansible -i inventory.ini nodes -m command -a "df -h"
```

### 4. Check memory

```bash
ansible -i inventory.ini nodes -m command -a "free -m"
```

### 5. Show effective remote user (often `ubuntu`)

```bash
ansible -i inventory.ini nodes -m command -a "whoami"
```

### 6. Check OS release

```bash
ansible -i inventory.ini nodes -m command -a "cat /etc/os-release"
```

### 7. List `/tmp`

```bash
ansible -i inventory.ini nodes -m command -a "ls -l /tmp"
```

### 8. Create a file (needs privilege on `/tmp` only if restricted — usually use `-b`)

```bash
ansible -i inventory.ini nodes -b -m command -a "touch /tmp/testfile"
```

**`-b` (`--become`):** run the task with privilege escalation (typically **sudo**) when writing system paths.

### 9. Remove the file

```bash
ansible -i inventory.ini nodes -b -m command -a "rm -f /tmp/testfile"
```

### 10. Last reboot line (no shell pipe)

You **cannot** rely on `|` with **`command`** (no shell). Use a single invocation that already limits output, for example:

```bash
ansible -i inventory.ini nodes -m command -a "last -n 1 reboot"
```

If you truly need `last reboot | head -1`, use **`-m shell`** instead for that one check.

### 11. Pipes: `shell` vs `command` (same `-a` string)

**Correct `shell` version** (pipeline runs on the remote host):

```bash
ansible -i inventory.ini nodes -m shell -a "echo hello | wc -c"
```

This works because **`shell`** runs your string through a shell on the managed node, so the shell understands **`|`** and wires **stdout** of `echo` into **stdin** of `wc`. You should see a small integer (line length including newline, often **6**).

**`command` module counterpart** (same string, wrong module):

```bash
ansible -i inventory.ini nodes -m command -a "echo hello | wc -c"
```

This does **not** run a pipeline. **`command`** invokes a program without a shell; Ansible passes the whole `-a` string as arguments to **`echo`**. Roughly, the remote process behaves like:

```text
echo  hello  '|'  wc  -c
```

So the printed line is literally something like:

```text
hello | wc -c
```

It does **not** count characters—there is no `wc` process fed by a pipe.

**Takeaway:** Any time you need **`|`, `>`, `>>`, `&&`, `;`, `*`** globbing from the shell, or similar, use **`-m shell`** (or refactor to a real script / dedicated Ansible modules). Use **`-m command`** when a single executable and its argv are enough.

---

## Part 2 — Same ideas as a playbook

### What a playbook adds

| Idea | Meaning |
|------|--------|
| **Play** | A mapping of **hosts** to **tasks** (and optional `vars`, `become`, etc.). |
| **`hosts:`** | Inventory group(s) or pattern this play runs against (`all`, `nodes`, …). |
| **`become: true`** | Same idea as **`-b`** on the CLI: tasks run elevated when needed. |
| **`command:`** task | Declarative equivalent of **`-m command`**. |
| **`register:`** | Save the module’s return value (stdout, rc, stderr, …) into a variable. |
| **`debug:`** | Print variables to the console — here we show captured output. |

**Execution flow:** Ansible connects to each host → runs each task in order → for `command`, captures stdout/stderr/rc → your **`debug`** tasks print selected fields (`stdout` or `stdout_lines`).

### Example file: `command-demo.yml`

Save next to `inventory.ini` (or under `playbooks/` in the repo skeleton). The repo copy lives at **`ansible/playbooks/command-demo.yml`**.

```yaml
---
- name: Command module demo
  hosts: nodes
  become: true

  tasks:
    - name: Check uptime
      ansible.builtin.command: uptime
      register: uptime_out

    - name: Show uptime output
      ansible.builtin.debug:
        var: uptime_out.stdout

    - name: Check hostname
      ansible.builtin.command: hostname
      register: host_out

    - name: Show hostname
      ansible.builtin.debug:
        var: host_out.stdout

    - name: Check disk usage
      ansible.builtin.command: df -h
      register: disk_out

    - name: Show disk (line by line)
      ansible.builtin.debug:
        var: disk_out.stdout_lines

    - name: Check memory
      ansible.builtin.command: free -m
      register: mem_out

    - name: Show memory
      ansible.builtin.debug:
        var: mem_out.stdout

    - name: Create demo file
      ansible.builtin.command: touch /tmp/ansible-demo.txt
      args:
        creates: /tmp/ansible-demo.txt

    - name: Last reboot entry
      ansible.builtin.command: last -n 1 reboot
      register: reboot_out

    - name: Show last reboot line
      ansible.builtin.debug:
        var: reboot_out.stdout
```

**Note on idempotency:** `command: touch ...` always reports **changed**. The optional **`creates:`** argument makes Ansible **skip** the task if that path already exists (closer to “safe reruns” for demos). For real file state, prefer **`ansible.builtin.file`** with `state: touch`.

### Run the playbook

From the directory that contains **`inventory.ini`** and **`command-demo.yml`**:

```bash
ansible-playbook -i inventory.ini command-demo.yml
```

From this repo’s **`ansible/`** root:

```bash
cd ansible
ansible-playbook -i inventory-command-lab.ini playbooks/command-demo.yml
```

---

## Step 3 — Verify and cleanup

- Re-run the playbook once; with **`creates:`**, the `touch` task should show **skipped** on the second run if the file still exists.
- Remove the demo file when finished (ad-hoc):

```bash
ansible -i inventory.ini nodes -b -m command -a "rm -f /tmp/ansible-demo.txt"
```

---

## Notes

- Prefer **`ansible.builtin.command`** (or `command:`) for single binaries; use **`shell`** only when shell syntax is required.
- **`stdout`** is one string; **`stdout_lines`** is a list — use whichever is easier to read in **`debug`**.
- Keep **secrets** out of playbooks; use **Vault** (`07-LAB-VAULT-INTRO.md`) when you need encrypted data.

---

## Reference — One-line summary of each ad-hoc example

| # | Command (pattern) |
|---|-------------------|
| 1 | `ansible -i inventory.ini nodes -m command -a "uptime"` |
| 2 | `ansible -i inventory.ini nodes -m command -a "hostname"` |
| 3 | `ansible -i inventory.ini nodes -m command -a "df -h"` |
| 4 | `ansible -i inventory.ini nodes -m command -a "free -m"` |
| 5 | `ansible -i inventory.ini nodes -m command -a "whoami"` |
| 6 | `ansible -i inventory.ini nodes -m command -a "cat /etc/os-release"` |
| 7 | `ansible -i inventory.ini nodes -m command -a "ls -l /tmp"` |
| 8 | `ansible -i inventory.ini nodes -b -m command -a "touch /tmp/testfile"` |
| 9 | `ansible -i inventory.ini nodes -b -m command -a "rm -f /tmp/testfile"` |
| 10 | `ansible -i inventory.ini nodes -m command -a "last -n 1 reboot"` |
| 11 | Pipe contrast: **`shell`** with `echo hello` + pipe + `wc -c` vs **`command`** with the same `-a` string — see **§11** above (two commands). |
