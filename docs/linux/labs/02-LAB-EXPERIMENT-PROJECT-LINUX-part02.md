# Lab examination — take-home project (Part 02)

**Coverage:** Draws on ideas from class labs **02** (files, permissions, ownership), **06** (cron), **07** (PostgreSQL install and tuning, roles, database and sample data, NFS export/mount, dump to NFS), **08** (Nginx static sites, `location`, `try_files`), and **09** (disk, filesystem, persistent mount via `/etc/fstab`). It does **not** require you to demonstrate every topic from those labs—only what this brief asks for.

**Format:** One **scenario** with **phases**. You decide **how** to accomplish each requirement using what you learned. This paper intentionally **does not** list shell commands; your evidence should show that you actually performed the work on **your Ubuntu (or Debian-style) VMs**.

**Prerequisite:** You should have attempted the referenced labs (or equivalent experience). Part 02 assumes **two Ubuntu VMs** on the same lab network for PostgreSQL + NFS (same roles as lab 07: one machine hosts Postgres and acts as the **NFS client** for backups; the other acts as the **NFS server**). Adjust IP addresses to match your environment; keep the **roles** clear in your evidence.

---

## Scenario

You are preparing an **internal documentation** platform. Editors use a **protected project tree** on the Postgres/Nginx host; **PostgreSQL** holds application data you must be able to back up; dumps must land on **NFS** only when the share is **actually mounted**; a **scheduler** proves automation is working; the same host serves **static docs over HTTP**; and **dedicated local disk** holds long-lived markers.

Phases are **ordered on purpose**: you install and configure the database first, then wire **NFS** (server VM + client mount on the DB host), then implement the **backup script** that depends on both, then add **cron**, then **Nginx**, then the **concept memo**.

Your manager wants a **short evidence pack** (screenshots + a small memo) proving each part works on real VMs—not a theoretical write-up alone.

You will submit **screenshots and a brief written addendum** as described below.

---

## Story flow (sequence you must follow)

```text
Phase 1 — Project files + permissions
Phase 2 — Extra local disk + fstab
Phase 3 — PostgreSQL: install → configure → roles → DB → sample data (+ least-privilege reader)
Phase 4 — NFS: server VM (export) + DB VM (mount); one agreed backup mount path
Phase 5 — Backup script (mount check + pg_dump to NFS); uses same path as Phase 4
Phase 6 — User cron + HEARTBEAT log (proves scheduler)
Phase 7 — Nginx static routes
Phase 8 — Concept memo
```

---

## Allowed materials

- Your notes and the course lab documents:
  - `02-LAB-FILES-PERMISSIONS.md`
  - `06-LAB-CRON-BASIC-AUTOMATION.md`
  - `07-LAB-POSTGRES-NFS-BACKUP-RESTORE-CRON.md`
  - `08-LAB-NGINX-BASICS-AND-SAMPLE-PROJECT.md`
  - `09-LAB-DISKS-FILESYSTEM-LVM.md`
- Your lab VMs only.

Do **not** include passwords, private SSH key material, or full `authorized_keys` dumps in your submission. Redact secrets and, if your instructor allows, partially redact sensitive UUIDs in `fstab` screenshots while still showing the line is yours.

---

## Phase 1 — Project tree and permissions (collaboration model)

**Requirement:**

1. Create a **Unix group** whose name you choose (avoid reserved system names).
2. Create a dedicated directory **under your home directory** (path you choose) that will hold Part 02 project files. Inside it:
   - at least **one** small HTML file (any harmless placeholder content), and  
   - at least **one** small plain text file (e.g. a “readme” or log placeholder).
3. Set **ownership and permissions** so that:
   - you (as owner) retain full control of the directory and its contents;
   - members of your new group can **enter** the directory and **read** files inside;
   - users who are **not** you and **not** in that group have **no** access to the directory.

**Evidence to submit:** Screenshots of metadata listings that clearly show the directory and files **owner**, **group**, and **permission bits** (the same style of detail you practiced in the files lab).

---

## Phase 2 — Dedicated disk space and persistent mount

**Requirement:** On the **same VM** you use for Phases 1 and 3–7 (the “application / DB / Nginx” host), using an **additional block device** (as in the disks lab: extra VirtualBox disk or equivalent—**do not** repartition your OS system disk):

1. Create a **single** Linux filesystem on that device.
2. Mount it at **`/mnt/part02-data`** (create the mount point if needed).
3. Make the mount **persistent across reboots** using `/etc/fstab` with a **stable identifier** (UUID or LABEL—your choice, but it must be correct for your system).
4. Place **one** small marker file on that mount whose contents include the string **`PART02-DISK`** and the date you created it.

**Evidence to submit:** Screenshots that **together** prove: the device exists in listing output, the mount is active at `/mnt/part02-data`, the marker file is visible on that mount, and your `fstab` line for this mount is shown (redact only if your course policy requires partial redaction).

---

## Phase 3 — PostgreSQL: install, configure, database, and sample data

**Requirement (on the DB / application VM—same host as Phases 1–2 and 5–7):** Follow the **same outcomes** as in `07-LAB-POSTGRES-NFS-BACKUP-RESTORE-CRON` (Part A style), adapted to your IPs and passwords:

1. **Install** the PostgreSQL server packages from the distribution and ensure the database service is **enabled** and **running** when you capture evidence.
2. **Configure** the server so it **listens** for client connections on the addresses required for your lab (not only a loopback socket), and **`pg_hba.conf`** (or equivalent) allows access from your **lab subnet** using a defined authentication method (e.g. `md5`/`scram-sha-256` as in the lab—your choice, but it must be consistent and safe for a classroom VM).
3. **Create roles** (database users) with distinct purposes, aligned with lab 07:
   - **`dbadmin`** — login role that **owns** `appdb` and can administer it (the lab uses `SUPERUSER` for this role; match that pattern unless your instructor specifies a stricter template).
   - **`appuser`** — login role for **least-privilege** application access: it must be able to **connect** to `appdb` and **read** application data, but must **not** be a cluster superuser.
4. **Create** an application database (use the name **`appdb`** so Phase 5 and your dumps stay comparable to the course lab).
5. **Create** a table **`employees`** with columns at least equivalent to the lab sample (`id`, `name`, `team`, and a timestamp column for row creation), and **insert at least two** clearly distinct rows (names/teams of your choice).
6. **Apply grants** so `appuser` can **SELECT** from `employees` (and any sequences needed for `SERIAL`/identity), following the **read-only application** pattern from lab 07 (including default privileges for future tables if you add them the same way as the lab).

**Evidence to submit:** Screenshots that **together** show: service state for PostgreSQL; proof of `listen_addresses` (or equivalent) allowing LAN access; a **`pg_hba`** snippet showing your subnet rule (redact passwords); `psql` (or meta-query) output listing **`appdb`** and the **`employees`** rows; proof that **`appuser`** can read `employees` while not being a cluster superuser (e.g. role attributes or a failed superuser-only command as `appuser`—your evidence should make the separation obvious).

---

## Phase 4 — NFS: two Ubuntu servers (export + client mount)

**Requirement:** Using **two** Ubuntu VMs:

1. **NFS server VM** — Install and configure NFS server software. Export a directory dedicated to Postgres dumps (the lab uses a path under `/srv/nfs/`; you may use the same layout or another path you document). The export must be reachable from your **lab subnet** (same idea as lab 07).
2. **NFS client (DB VM)** — On the **same VM** as PostgreSQL (Phase 3), install NFS client tooling, create a **single client mount point** for backups, and mount the server export there. Use **`/mnt/nfs/pg-backups`** as the **client mount point** so it matches the course story and your Phase 5 script variable `NFS_DUMP_ROOT`.
3. Make the NFS mount **persistent on the client** across reboots (e.g. `/etc/fstab` with `_netdev` or equivalent appropriate options, as in lab 07).
4. Prove connectivity: from the client, show the mount is active and that a **test file** written under the mount is visible on the server side (or vice versa).

**Evidence to submit:** Screenshots showing: export definition on the server; **`showmount`** or equivalent from the client; **`findmnt`** or **`mount`** output on the client showing **`/mnt/nfs/pg-backups`** backed by NFS; your client **`fstab`** line for this mount; the shared test file on one or both sides.

---

## Phase 5 — Postgres dump script (logic from lab 07, **not** a copy-paste)

**Requirement:** This phase **depends on Phases 3–4**. The directory **`/mnt/nfs/pg-backups`** on the DB VM must be what you assign to **`NFS_DUMP_ROOT`** (it must be the real NFS mount from Phase 4).

Create an executable Bash script **`~/linux-labs/part02/pg_dump_to_nfs.sh`** (create the `part02` directory if needed). The script must follow the **same control idea** as the lab’s `pg_backup_to_nfs.sh` (fail safely, verify the NFS target is a **mountpoint** before writing, run a custom-format dump, record outcome in a log), but you **must** use **only** the variable names below for the concepts shown—do **not** reuse the lab’s original names (`BACKUP_DIR`, `DB_NAME`, `DB_USER`, `TS`, `OUT_FILE`, `LOG_FILE`).

**Mandatory names and meanings (use these identifiers exactly):**

| Identifier | Role |
|------------|------|
| `NFS_DUMP_ROOT` | Must be **`/mnt/nfs/pg-backups`**. Verified with `mountpoint` before any dump. |
| `PGDATABASE` | Must be **`appdb`**. |
| `PG_OS_USER` | System user as whom the dump runs (typically **`postgres`**). |
| `RUN_STAMP` | Timestamp string; you **must** build it with **`date -u +%Y%m%dT%H%M%SZ`** (UTC). |
| `ARCHIVE_PATH` | Full path of the output dump file, built from `NFS_DUMP_ROOT`, `PGDATABASE`, and `RUN_STAMP`, ending in **`.dump`**. |
| `AUDIT_LOG` | Log file path; you **must** set this to **`$HOME/linux-labs/part02/backup-audit.log`** (expand `$HOME` appropriately in the script so it resolves reliably). |

**Behavior (minimum):**

- Use `set -euo pipefail` (or equivalent strictness your instructor accepts).
- If `NFS_DUMP_ROOT` is **not** a mountpoint, log an **`ERROR`** line to `AUDIT_LOG` (and stderr is fine) and exit non-zero—**do not** run `pg_dump` in that case.
- On success, create the dump at `ARCHIVE_PATH` using **`pg_dump -Fc`** for `PGDATABASE` as `PG_OS_USER`, and append a **`SUCCESS`** line to `AUDIT_LOG` mentioning the archive path.

**Evidence to submit:**

1. Screenshot of the **first ~35 lines** of your script showing the **six** identifiers above in use (not merely commented), including `NFS_DUMP_ROOT=/mnt/nfs/pg-backups`.
2. Screenshot of a **successful** manual run (terminal) **or** a principled failure where the **`ERROR`** path clearly references mount state—your submission should match what is true on your VM at capture time.
3. Screenshot of **`tail`** of `AUDIT_LOG` showing the matching **`SUCCESS`** or **`ERROR`** lines, and a listing showing a **`.dump`** file on the NFS mount when successful.

---

## Phase 6 — Scheduled job (automation)

### Objective (what this phase proves)

You must show that the **cron daemon** on the **DB / application VM** can run work on a schedule **without you typing commands**, and that each run leaves a **durable trace** in a log file. That is how operations teams prove “automation is alive” in real environments.

### What the requirement means

You need a **user** cron job (**your** crontab on the DB VM—not root’s unless your instructor explicitly allows an exception) that:

- Runs **automatically** on a fixed schedule of **every five minutes or more often** (e.g. `*/5 * * * *` or `* * * * *`).
- Executes a **command or script** each time the schedule fires.
- **Appends** (does not overwrite) **one line per run** to a log file stored **under your Part 02 project directory from Phase 1**.
- Each appended line **must** contain:
  - a **timestamp** (any unambiguous form you choose—ISO-style is fine), and  
  - the literal word **`HEARTBEAT`** (uppercase, exact spelling),

so an instructor can scroll the log and see **multiple lines spaced in time**, matching the cron cadence.

### Recommended approach

Create a small **Bash script** (for example under `~/linux-labs/part02/` or inside your Phase 1 project tree) that appends exactly one line per invocation to your chosen log file under Phase 1. Schedule **cron** to invoke that script on the required interval. Keeping the logic in a script avoids cramming fragile quoting into a single crontab line and matches how lab 06 introduces automation.

### Concept flow

```text
cron daemon
   ↓   every ≤5 minutes
your script.sh
   ↓
append one line to log file
   (timestamp … HEARTBEAT …)
```

**Evidence to submit:** Screenshot of **`crontab -l`** for your user showing the schedule, plus a screenshot of **`tail`** of the log file showing **multiple** **`HEARTBEAT`** lines with timestamps consistent with repeated runs (wait long enough before capturing).

---

## Phase 7 — Nginx static routing (moderate challenge)

**Requirement:** Configure Nginx (package install as in the lab) on the **DB / application VM** to serve a small static site from a **`root`** directory you control (under `/var/www/...` or another path you justify). The site must meet **all** of the following:

1. **`/`** — Serves an `index.html` home page (file-level layout as in lab 08).
2. **`location /`** — Uses exactly:  
   `try_files $uri $uri.html $uri/ =404;`
3. **`/faq`** — Serves your FAQ page via the **pretty URL** (i.e. backed by `faq.html` in the same folder, without requiring a `/faq/` directory on disk).
4. **`/home`** — Must show the **same** document as **`/`** using an **exact** match location (`location = /home { ... }`), not by duplicating unrelated files.
5. **`/team`** — Must serve **`team.html`** via the pretty URL (create `team.html`; no `team/` directory required).
6. **`/secret`** — Must return **404** (no file served for that path).

**Evidence to submit:** Screenshot of **clean** config test output for Nginx, plus **`curl -I`** (or equivalent) screenshots for **`/`**, **`/home`**, **`/faq`**, **`/team`**, and **`/secret`** showing the expected status codes (200 family for the first four as appropriate, **404** for `/secret`).

---

## Phase 8 — Concept memo (three questions)

Answer in **your own words** (roughly **one short paragraph each**). No screenshots.

1. **Mounts and backups:** Why is checking **`mountpoint`** on `NFS_DUMP_ROOT` before writing a dump safer than only checking that the directory exists? What could go wrong if the NFS share is offline but the mount point directory is still present on the local root filesystem?

2. **`try_files`:** In the directive `try_files $uri $uri.html $uri/ =404;`, explain **in order** what Nginx tries for a request like **`/faq`**, assuming `root` points at your site folder and `faq.html` exists there.

3. **Cron environment:** Why might a script behave differently when run **from your interactive shell** versus **from cron**? Name at least **two** environment or path-related factors students often forget.

---

## How to submit

Send everything to:

**faizulkhan56@gmail.com**

**Subject line (required):**

```text
Linux Lab Exam Part02 - <Your Full Name> - <Roll or ID if applicable>
```

**Attachments:**

- A **single ZIP** or **one PDF** containing:
  - screenshots for Phases **1–7** (label files clearly, e.g. `Part02-Phase1.png`, …), and  
  - the **Phase 8** answers in the same PDF **or** as a separate `Part02-Memo-<YourName>.pdf` / `.docx`.

**Checklist before send:**

- [ ] Phase 1–7 evidence is present and readable  
- [ ] Phase 8 answered in prose  
- [ ] No secrets or private keys in the package  
- [ ] Subject line includes your name  

---

## How participants will be evaluated

| Criterion | Looks for |
|-----------|-----------|
| Phase 1 | Group exists; directory and files match the stated permission policy |
| Phase 2 | Extra filesystem; mount at `/mnt/part02-data`; persistent `fstab`; marker file with required string |
| Phase 3 | Postgres installed; listen + `pg_hba` for subnet; `dbadmin`-style and `appuser`-style roles; `appdb`; `employees` with ≥2 rows; read-only pattern for `appuser` |
| Phase 4 | Two VMs; NFS export; client mount at `/mnt/nfs/pg-backups`; persistent client fstab; connectivity proof |
| Phase 5 | Script path; **six** mandated variable names; `NFS_DUMP_ROOT` matches Phase 4; mount check before dump; `pg_dump -Fc`; `AUDIT_LOG` path; UTC `RUN_STAMP`; evidence matches behavior |
| Phase 6 | User crontab; interval ≤ 5 minutes; log under Phase 1 directory; multiple **`HEARTBEAT`** lines over time |
| Phase 7 | Required `try_files` line; `/`, `/home`, `/faq`, `/team`, `/secret` behaviors; config test clean |
| Phase 8 | Correct concepts in own words |

---

*This is Part 02. Part 01 is `01-LAB-EXPERIMENT-PROJECT-LINUX-part01.md`. A separate solution guide may be published later.*
