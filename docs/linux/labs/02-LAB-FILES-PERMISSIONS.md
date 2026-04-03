# Linux Lab 02 — Files, permissions, and ownership

## Objective

- Understand permissions (r/w/x) and ownership (user/group)
- Practice `chmod`, `chown`, and `umask`

## Theory — UNIX permission model (r/w/x, 4-2-1, u/g/o)

- Every file/dir has:
  - Owner (user) and Group
  - Permission bits for three classes: User (u), Group (g), Others (o)
- Bits:
  - Read (r) = 4
  - Write (w) = 2
  - Execute (x) = 1
- Numeric mode adds these per class: e.g., `644` = `6(u)=4+2`, `4(g)=4`, `4(o)=4` → `rw-r--r--`
- For directories:
  - `x` = traverse/enter directory
  - `r` = list names (with `ls`), often needs `x` as well
  - `w` = create/delete/rename entries (needs `x` too)

Examples:

```text
-rw-------  (600)  owner can read/write; group/others no access (private file)
-rw-r--r--  (644)  owner read/write; group/others read-only (common for text)
-rwxr-xr-x  (755)  owner rwx; group/others rx (common for executables/dirs)
drwxr-x---  (750)  directory: owner rwx; group rx; others none
```

## Step 1 — Create a lab directory

```bash
mkdir -p ~/linux-labs/lab02
cd ~/linux-labs/lab02
```

## Step 2 — Create a file and inspect permissions

```bash
echo "hello" > hello.txt
ls -l hello.txt
stat hello.txt
```

Command breakdown:
- `ls -l`: long listing shows `type+mode owner group size date name`
- `stat`: detailed metadata including octal permissions

## Step 3 — Change permissions

```bash
chmod 600 hello.txt
ls -l hello.txt

chmod 644 hello.txt
ls -l hello.txt
```

`chmod` basics:
- Symbolic: `chmod u+rwx,g+rx,o-r` (add/remove per class)
- Numeric: `chmod 640 file` (u=6,g=4,o=0)
- Recursive (dirs): `chmod -R 755 somedir`

Directory tip:

```bash
mkdir demo && touch demo/a demo/b
ls -ld demo
chmod 755 demo         # allow traversal
ls -l demo
```

## Step 4 — Execute permission (script)

```bash
cat > runme.sh <<'EOF'
#!/usr/bin/env bash
echo "Running as: $(whoami)"
EOF

ls -l runme.sh
chmod +x runme.sh
./runme.sh
```

Notes:
- Files need `+x` to be executed directly (`./runme.sh`). Directories need `+x` to be entered (`cd demo`).

## Step 5 — Ownership (requires sudo)

```bash
sudo useradd -m -s /bin/bash appuser || true
sudo chown appuser:appuser hello.txt
ls -l hello.txt
```

`chown` and `chgrp`:

```bash
sudo chown newowner file
sudo chown newowner:newgroup file
sudo chgrp newgroup file
sudo chown -R appuser:appuser somedir    # recursive
```

When to change ownership:
- Move files to be managed by a service account/team group
- Ensure apps can read/write only what they need

## Step 6 — Umask (default permission mask)

```bash
umask
umask 027
touch umask-test.txt
ls -l umask-test.txt
```

Reset `umask` by opening a new shell session.

Theory — What `umask` does

- `umask` subtracts permissions from the default creation mode.
  - Files default: 666 (rw-rw-rw-) minus umask
  - Dirs default: 777 (rwxrwxrwx) minus umask
- Example:
  - `umask 022` → files 644, dirs 755 (others can read/execute dirs)
  - `umask 027` → files 640, dirs 750 (others get no access)

Examples:

```bash
umask 022 && touch f1 && mkdir d1 && ls -l f1 && ls -ld d1
umask 077 && touch f2 && mkdir d2 && ls -l f2 && ls -ld d2
```

## Step 7 — POSIX ACLs (fine-grained permissions)

Why ACLs:
- `chmod` applies to owner/group/others only. ACLs let you grant specific users/groups extra rights without changing the owning group or global “others”.

Enabling tools:

```bash
sudo apt-get update
sudo apt-get install -y acl
```

Basic usage:

```bash
touch report.txt
ls -l report.txt          # note the '+' indicator if ACL present

# Grant read to a specific user (alice) in addition to normal mode bits
sudo setfacl -m u:alice:r report.txt
getfacl report.txt

# Grant read+write to a group (devs)
sudo setfacl -m g:devs:rw report.txt
getfacl report.txt

# Remove an entry
sudo setfacl -x u:alice report.txt

# Clear all ACLs (keep basic mode bits)
sudo setfacl -b report.txt
```

Default ACLs on directories (inheritance for new files/dirs inside):

```bash
mkdir shared
sudo setfacl -m d:g:devs:rwx shared     # default ACL for group devs
getfacl shared
```

chmod vs ACL (summary):
- `chmod` (mode bits) applies to owner/group/others only; simple, universal
- ACLs are additive, per-user/group entries; great for exceptions and shared folders
- Effective permission is the combination: start with mode bits; ACLs can grant more (but not typically more than what the directory traversal allows)
