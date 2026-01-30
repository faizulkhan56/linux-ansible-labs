# Linux Lab 06 — Cron (basic automation)

## Objective

- Create a simple script
- Schedule it using cron

## Step 1 — Create a script

```bash
mkdir -p ~/linux-labs/lab06
cd ~/linux-labs/lab06

cat > heartbeat.sh <<'EOF'
#!/usr/bin/env bash
echo "$(date -Is) heartbeat from $(hostname)" >> "$HOME/heartbeat.log"
EOF

chmod +x heartbeat.sh
./heartbeat.sh
tail -n 3 ~/heartbeat.log
```

## Step 2 — Create a cron job

Edit your crontab:

```bash
crontab -e
```

Add (runs every minute):

```text
* * * * * /home/<your-user>/linux-labs/lab06/heartbeat.sh
```

List your cron jobs:

```bash
crontab -l
```

## Step 3 — Validate

Wait 2–3 minutes, then:

```bash
tail -n 5 ~/heartbeat.log
```

## Cleanup (optional)

Remove the cron line from `crontab -e` when done.

