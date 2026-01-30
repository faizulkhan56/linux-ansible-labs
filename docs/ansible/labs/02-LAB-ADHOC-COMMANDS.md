# Lab 02 — Ad-hoc commands

## Objective

Learn to run safe one-liners:
- connectivity checks
- gather facts
- run shell/command modules
- install a package

## Step 0 — Setup

Use your `inventory.ini` from Lab 01.

```bash
cd ~/ansible-labs/lab01
```

## Step 1 — Ping

```bash
ansible -i inventory.ini nodes -m ping
```

## Step 2 — Gather facts

```bash
ansible -i inventory.ini nodes -m setup | head -n 40
```

## Step 3 — Run a command

```bash
ansible -i inventory.ini nodes -m command -a "uname -a"
```

## Step 4 — Run a shell pipeline (use shell module)

```bash
ansible -i inventory.ini nodes -m shell -a "df -h | tail -n +2"
```

## Step 5 — Install a package (Ubuntu)

```bash
ansible -i inventory.ini nodes -b -m apt -a "name=htop state=present update_cache=yes"
```

Verify:

```bash
ansible -i inventory.ini nodes -m command -a "htop --version"
```

## Notes

- Prefer `command` over `shell` unless you need shell features (pipes, redirects).
- Use `-b` (become) for tasks needing sudo.

