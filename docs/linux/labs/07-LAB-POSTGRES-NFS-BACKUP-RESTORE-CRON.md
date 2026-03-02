# Linux Lab 07 — PostgreSQL + NFS backup/restore + cron

## Objective

- Use two VMs on the same network:
  - `192.168.56.104` = PostgreSQL server and backup client
  - `192.168.56.105` = NFS server
- Install and configure PostgreSQL on `192.168.56.104`
- Create admin/app users, test database, and table data
- Configure NFS export on `192.168.56.105` and mount it on `192.168.56.104`
- Write a backup script that stores PostgreSQL dumps on NFS
- Validate restore by dropping and rebuilding the test DB from backup
- Automate with cron:
  - Step 1: every 5 minutes (classroom validation)
  - Step 2: daily run with logging

## Topology reminder

- VM1 (DB + backup client): `192.168.56.104`
- VM2 (NFS server): `192.168.56.105`
- Both VMs are in the same subnet, so they should reach each other directly.

Quick check from each VM:

```bash
ping -c 2 192.168.56.104
ping -c 2 192.168.56.105
```

---

## Part A — PostgreSQL setup on `192.168.56.104`

SSH to VM `192.168.56.104` first.

### Step A1: Install PostgreSQL

```bash
sudo apt-get update
sudo apt-get install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
sudo systemctl status postgresql --no-pager
```

### Step A2: Find PostgreSQL version and config path

```bash
psql --version
ls -l /etc/postgresql/
```

Set a helper variable:

```bash
PG_VER=$(psql --version | awk '{print $3}' | cut -d. -f1,2)
echo "$PG_VER"
```

### Step A3: Update `postgresql.conf`

Edit:

```bash
sudo nano /etc/postgresql/$PG_VER/main/postgresql.conf
```

Ensure:

```conf
listen_addresses = '*'
```

### Step A4: Update `pg_hba.conf`

Edit:

```bash
sudo nano /etc/postgresql/$PG_VER/main/pg_hba.conf
```

Add these lines near other host rules:

```conf
# Allow local lab subnet clients (adjust auth method for production)
host    all             all             192.168.56.0/24         md5
```

Reload:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql --no-pager
```

### Step A5: Create admin/app users and test DB

Open PostgreSQL shell:

```bash
sudo -u postgres psql
```

Run:

```sql
CREATE ROLE dbadmin WITH LOGIN SUPERUSER PASSWORD 'AdminPass123!';
CREATE ROLE appuser WITH LOGIN PASSWORD 'AppPass123!';
CREATE DATABASE appdb OWNER dbadmin;
\q
```

Create table and seed rows:

```bash
sudo -u postgres psql -d appdb -c "CREATE TABLE IF NOT EXISTS employees (id SERIAL PRIMARY KEY, name TEXT NOT NULL, team TEXT NOT NULL, created_at TIMESTAMP DEFAULT now());"
sudo -u postgres psql -d appdb -c "INSERT INTO employees (name, team) VALUES ('Amina', 'Platform'), ('Rahim', 'Data');"
sudo -u postgres psql -d appdb -c "SELECT * FROM employees;"
```

Optional TCP test to localhost (forces password auth path):

```bash
PGPASSWORD='AppPass123!' psql -h 127.0.0.1 -U appuser -d appdb -c "SELECT current_user, current_database();"
```

---

## Part B — NFS server setup on `192.168.56.105`

SSH to VM `192.168.56.105`.

### Step B1: Install and prepare export path

```bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/pg-backups
sudo chown -R nobody:nogroup /srv/nfs/pg-backups
sudo chmod 0777 /srv/nfs/pg-backups
```

### Step B2: Export to `192.168.56.104`

Edit exports:

```bash
sudo nano /etc/exports
```

Add:

```conf
/srv/nfs/pg-backups 192.168.56.104(rw,sync,no_subtree_check)
```

Apply export and verify:

```bash
sudo exportfs -ra
sudo exportfs -v
sudo systemctl enable --now nfs-kernel-server
sudo systemctl status nfs-kernel-server --no-pager
```

---

## Part C — Mount NFS on DB VM `192.168.56.104`

Back on VM `192.168.56.104`:

### Step C1: Install NFS client and mount

```bash
sudo apt-get update
sudo apt-get install -y nfs-common
sudo mkdir -p /mnt/nfs/pg-backups
sudo mount 192.168.56.105:/srv/nfs/pg-backups /mnt/nfs/pg-backups
mount | grep pg-backups
df -h | grep pg-backups
```

### Step C2: Persist mount in `/etc/fstab`

This line appends one permanent NFS mount entry to `/etc/fstab`:

```bash
echo "192.168.56.105:/srv/nfs/pg-backups /mnt/nfs/pg-backups nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo umount /mnt/nfs/pg-backups
sudo mount -a
df -h | grep pg-backups
```

Field-by-field explanation for:
`192.168.56.105:/srv/nfs/pg-backups /mnt/nfs/pg-backups nfs defaults,_netdev 0 0`

- `192.168.56.105:/srv/nfs/pg-backups` -> remote NFS source (`<server-ip>:<export-path>`)
- `/mnt/nfs/pg-backups` -> local mount point on `192.168.56.104`
- `nfs` -> filesystem type
- `defaults,_netdev` -> standard mount options + delay until network is ready
- first `0` -> disable `dump` backup flag for this entry
- second `0` -> disable `fsck` checks for this NFS mount

`sudo mount -a` validates all `/etc/fstab` entries immediately without reboot.

### Step C3: Write test to NFS

```bash
echo "NFS test from $(hostname) at $(date -Is)" | sudo tee /mnt/nfs/pg-backups/nfs_test.txt
ls -l /mnt/nfs/pg-backups
```

---

## Part D — Backup script on `192.168.56.104`

### Step D1: Create script

```bash
mkdir -p ~/linux-labs/lab07
cd ~/linux-labs/lab07
```

Create `pg_backup_to_nfs.sh`:

```bash
cat > ~/linux-labs/lab07/pg_backup_to_nfs.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/mnt/nfs/pg-backups"
DB_NAME="appdb"
DB_USER="postgres"
TS="$(date +%F_%H%M%S)"
OUT_FILE="$BACKUP_DIR/${DB_NAME}_${TS}.dump"
LOG_FILE="/var/log/pg_backup_cron.log"

mkdir -p "$BACKUP_DIR"

if ! mountpoint -q "$BACKUP_DIR"; then
  echo "$(date -Is) ERROR: $BACKUP_DIR is not mounted" | tee -a "$LOG_FILE"
  exit 1
fi

sudo -u "$DB_USER" pg_dump -Fc "$DB_NAME" > "$OUT_FILE"
echo "$(date -Is) SUCCESS: backup created $OUT_FILE" | tee -a "$LOG_FILE"

# Keep only last 7 backups (optional retention)
ls -1t "$BACKUP_DIR"/${DB_NAME}_*.dump 2>/dev/null | tail -n +8 | xargs -r rm -f
EOF

chmod +x ~/linux-labs/lab07/pg_backup_to_nfs.sh
```

Create log file and permissions:

```bash
sudo touch /var/log/pg_backup_cron.log
sudo chmod 664 /var/log/pg_backup_cron.log
```

### Step D2: Run script manually and verify

```bash
~/linux-labs/lab07/pg_backup_to_nfs.sh
ls -lh /mnt/nfs/pg-backups/appdb_*.dump
tail -n 20 /var/log/pg_backup_cron.log
```

---

## Part E — Restore validation test

This proves the backup can recover data.

### Step E1: Capture latest backup file name

```bash
LATEST_BACKUP=$(ls -1t /mnt/nfs/pg-backups/appdb_*.dump | head -n 1)
echo "$LATEST_BACKUP"
```

### Step E2: Confirm current data

```bash
sudo -u postgres psql -d appdb -c "SELECT count(*) AS row_count FROM employees;"
```

### Step E3: Drop and recreate DB, then restore

```bash
sudo -u postgres psql -c "DROP DATABASE appdb;"
sudo -u postgres psql -c "CREATE DATABASE appdb OWNER dbadmin;"
sudo -u postgres pg_restore -d appdb "$LATEST_BACKUP"
```

### Step E4: Validate restored data

```bash
sudo -u postgres psql -d appdb -c "\dt"
sudo -u postgres psql -d appdb -c "SELECT * FROM employees;"
```

If rows are visible again, backup/restore flow is successful.

---

## Part F — Cron automation with logs

### Step F1: Classroom demo schedule (every 5 minutes)

Edit root cron:

```bash
sudo crontab -e
```

Add:

```text
*/5 * * * * /home/<your-user>/linux-labs/lab07/pg_backup_to_nfs.sh >> /var/log/pg_backup_cron.log 2>&1
```

Validate:

```bash
sudo crontab -l
sudo tail -n 50 /var/log/pg_backup_cron.log
```

Wait 5 to 10 minutes and re-check:

```bash
ls -lh /mnt/nfs/pg-backups
sudo tail -n 50 /var/log/pg_backup_cron.log
```

### Step F2: Production-style schedule (daily once)

Replace the every-5-minute line with a daily run (example: 2:00 AM):

```text
0 2 * * * /home/<your-user>/linux-labs/lab07/pg_backup_to_nfs.sh >> /var/log/pg_backup_cron.log 2>&1
```

Re-check:

```bash
sudo crontab -l
```

---

## Troubleshooting quick notes

- `Permission denied` on dump file:
  - Check write permissions on `/mnt/nfs/pg-backups`
  - Check NFS export and mount options
- `connection refused` or auth issues in PostgreSQL:
  - Re-check `listen_addresses` and `pg_hba.conf`
  - Restart PostgreSQL after config changes
- Script works manually but not in cron:
  - Use full paths
  - Ensure cron runs as user with required permissions
  - Review `/var/log/pg_backup_cron.log`

## Cleanup (optional)

- Remove test cron entries: `sudo crontab -e`
- Remove lab files:

```bash
rm -rf ~/linux-labs/lab07
```

