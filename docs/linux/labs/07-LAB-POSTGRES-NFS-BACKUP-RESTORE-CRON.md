# Linux Lab 07 — PostgreSQL + NFS backup/restore + cron

## Objective

- Use two VMs on the same network:
  - `192.168.56.104` = PostgreSQL server and backup client
  - `192.168.56.105` = NFS server
- Install and configure PostgreSQL on `192.168.56.104`
- Create admin/app users, test database, and table data
- Apply a **least-privilege** permission layer so `appuser` can connect and **read** application data (not administer the cluster)
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
# On Ubuntu 24.04, config path uses major version (for example: 16)
PG_VER=$(ls /etc/postgresql | sort -V | tail -n 1)
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

### Step A6: Permission layer for `appuser` (read-only on `appdb`)

**Why do this?** The application should not run as a superuser. Grant only what it needs: attach to the right database, use the `public` schema, **read** tables (and, when applicable, read sequences). That limits damage if credentials leak and keeps access easier to audit.

After Step A5, `appuser` can authenticate but has **no** rights on `appdb` objects until you grant them. Typical layers:

1. **CONNECT** on the database — required to open a session against `appdb`.
2. **USAGE** on the schema — required to reference objects in `public`.
3. **SELECT** on tables — read rows in existing tables.
4. **DEFAULT PRIVILEGES** — **new** tables and sequences created later get the same grants automatically; without this, migrations often break readers in production.
5. **Sequences** (recommended here) — `SERIAL` / identity columns use sequences; some clients need **USAGE** and **SELECT** on those objects.

Open a shell as the PostgreSQL superuser and connect to `appdb`:

```bash
sudo -u postgres psql -d appdb
```

Run:

```sql
-- CONNECT: allow sessions on database appdb
GRANT CONNECT ON DATABASE appdb TO appuser;
```

**Explanation:** `CONNECT` is the database-level gate. Without it, the role cannot select `appdb` even if you later add table grants.

```sql
-- USAGE: allow resolving names in schema public
GRANT USAGE ON SCHEMA public TO appuser;
```

**Explanation:** Permissions are scoped per schema. `USAGE` on `public` lets the role reference objects in that schema so table grants behave as expected.

```sql
-- SELECT: read all tables that already exist in public
GRANT SELECT ON ALL TABLES IN SCHEMA public TO appuser;
```

**Explanation:** `ALL TABLES` covers **current** tables only, not objects created tomorrow.

```sql
-- Default privileges for NEW tables created by role "postgres"
-- (This lab creates tables with sudo -u postgres; match FOR ROLE to the creator.)
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
  GRANT SELECT ON TABLES TO appuser;
```

**Explanation:** `ALTER DEFAULT PRIVILEGES` is **not** retroactive. It applies to objects **created by** the named role after this runs. If migrations use `dbadmin` (or another role) to create tables, add the same pattern with `FOR ROLE dbadmin`.

```sql
-- Sequences (SERIAL on employees.id, etc.)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;

ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO appuser;
```

**Explanation:** Sequences are separate objects from tables; granting them avoids surprises for tools that touch sequence metadata.

```sql
\q
```

**Verify grants** (as `postgres`):

```bash
sudo -u postgres psql -d appdb -c "SELECT table_schema, table_name, privilege_type FROM information_schema.role_table_grants WHERE grantee = 'appuser' ORDER BY table_name, privilege_type;"
```

You should see `SELECT` on `employees` (and related entries as applicable).

#### Optional: `readonly` role (RBAC)

Many teams define a **non-login** role that holds read grants, then grant **membership** to application users. Use a fresh `psql` session as `postgres` on `appdb` (for a greenfield lab VM you can use this pattern **instead of** the direct-to-`appuser` grants above, or keep both—direct grants and membership are redundant but harmless):

```bash
sudo -u postgres psql -d appdb
```

```sql
CREATE ROLE readonly NOLOGIN;

GRANT CONNECT ON DATABASE appdb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO readonly;

GRANT readonly TO appuser;
\q
```

**Why:** One definition of “reader”; add more users with `GRANT readonly TO otheruser`. Revoke or change access in one place instead of duplicating grants per login.

#### Test as `appuser`: database role vs Linux user

`appuser` is a **PostgreSQL role with LOGIN**. It does **not** need a Linux account. **`sudo -u appuser`** switches the **OS** user, so you may see `unknown user appuser`—that is expected if no Unix user exists.

Use the client’s **`-U`** to choose the **database** user and authenticate per `pg_hba.conf`. Over **TCP to localhost**, this lab’s `host ... md5` (or SCRAM) rules apply—**not** peer auth:

```bash
PGPASSWORD='AppPass123!' psql -h 127.0.0.1 -U appuser -d appdb -c "SELECT current_user, current_database();"
PGPASSWORD='AppPass123!' psql -h 127.0.0.1 -U appuser -d appdb -c "SELECT id, name, team FROM employees;"
```

**Peer authentication:** `psql -U appuser -d appdb` **without** `-h` often uses **peer** in `pg_hba.conf` (maps to the Linux username). That fails when there is no OS user `appuser`. Using **`-h 127.0.0.1`** forces TCP and password auth as configured above.

**Why `sudo -u postgres psql` works:** `postgres` is typically both a **Linux** user and a PostgreSQL superuser, so local peer connections from that OS account succeed.

For a remote database host, add **`-h <server-ip>`**.

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
sudo chown "$(whoami)":"$(whoami)" /var/log/pg_backup_cron.log
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
*/5 * * * * /home/<your-user>/linux-labs/lab07/pg_backup_to_nfs.sh
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
0 2 * * * /home/<your-user>/linux-labs/lab07/pg_backup_to_nfs.sh
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

