# Linux Lab 02 — Files, permissions, and ownership

## Objective

- Understand permissions (r/w/x) and ownership (user/group)
- Practice `chmod`, `chown`, and `umask`

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

## Step 3 — Change permissions

```bash
chmod 600 hello.txt
ls -l hello.txt

chmod 644 hello.txt
ls -l hello.txt
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

## Step 5 — Ownership (requires sudo)

```bash
sudo useradd -m -s /bin/bash appuser || true
sudo chown appuser:appuser hello.txt
ls -l hello.txt
```

## Step 6 — Umask (default permission mask)

```bash
umask
umask 027
touch umask-test.txt
ls -l umask-test.txt
```

Reset `umask` by opening a new shell session.

