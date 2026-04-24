# Lab examination — take-home project (Part 02)

**Coverage:** Draws on ideas from class labs **02** (files, permissions, ownership), **06** (cron), **07** (PostgreSQL dump to NFS, mounts, optional restore concepts), **08** (Nginx static sites, `location`, `try_files`), and **09** (disk, filesystem, persistent mount via `/etc/fstab`). It does **not** require you to demonstrate every topic from those labs—only what this brief asks for.

**Format:** One **scenario** with **phases**. You decide **how** to accomplish each requirement using what you learned. This paper intentionally **does not** list shell commands; your evidence should show that you actually performed the work on **your Ubuntu (or Debian-style) VM**.

**Prerequisite:** You should have attempted the referenced labs (or equivalent experience) so NFS, Postgres, and Nginx pieces are feasible on your VM topology.

---

## Scenario

You are preparing an **internal documentation** Linux host. Content editors will drop static HTML into a **protected project tree**; backups must land on **network-attached storage** that is only written when it is **actually mounted**; a **scheduler** records routine checks; the same machine (or your lab layout as applicable) serves the docs over **HTTP with predictable URL paths**; and long-lived data must sit on **dedicated storage** you added and made persistent.

Your manager wants a **short evidence pack** (screenshots + a small memo) proving each part below works on a real VM—not a theoretical write-up alone.

You will submit **screenshots and a brief written addendum** as described below.

---

## Allowed materials

- Your notes and the course lab documents:
  - `02-LAB-FILES-PERMISSIONS.md`
  - `06-LAB-CRON-BASIC-AUTOMATION.md`
  - `07-LAB-POSTGRES-NFS-BACKUP-RESTORE-CRON.md`
  - `08-LAB-NGINX-BASICS-AND-SAMPLE-PROJECT.md`
  - `09-LAB-DISKS-FILESYSTEM-LVM.md`
- Your lab VM only.

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

**Requirement:** Using an **additional block device** on your VM (as in the disks lab: extra VirtualBox disk or equivalent—**do not** repartition your OS system disk):

1. Create a **single** Linux filesystem on that device.
2. Mount it at **`/mnt/part02-data`** (create the mount point if needed).
3. Make the mount **persistent across reboots** using `/etc/fstab` with a **stable identifier** (UUID or LABEL—your choice, but it must be correct for your system).
4. Place **one** small marker file on that mount whose contents include the string **`PART02-DISK`** and the date you created it.

**Evidence to submit:** Screenshots that **together** prove: the device exists in listing output, the mount is active at `/mnt/part02-data`, the marker file is visible on that mount, and your `fstab` line for this mount is shown (redact only if your course policy requires partial redaction).

---

## Phase 3 — Scheduled job (automation)

**Requirement:** Schedule a **user** cron job (your own crontab, not root’s) that runs **at least once every five minutes** and appends **one line per run** to a log file **stored under your Part 02 project directory from Phase 1**. Each line must include a timestamp and the word **`HEARTBEAT`** so an instructor can see the schedule firing.

**Evidence to submit:** Screenshot of the cron listing for your user, plus a screenshot of the tail of the log file showing multiple **`HEARTBEAT`** lines spaced in time (wait long enough before capturing).

---

## Phase 4 — Postgres dump script (logic from lab 07, **not** a copy-paste)

**Requirement:** Create an executable Bash script **`~/linux-labs/part02/pg_dump_to_nfs.sh`** (create the `part02` directory if needed). The script must follow the **same control idea** as the lab’s `pg_backup_to_nfs.sh` (fail safely, verify the NFS target is a **mountpoint** before writing, run a custom-format dump, record success in a log), but you **must** use **only** the variable names below at the top of your script for the concepts shown—do **not** reuse the lab’s original names (`BACKUP_DIR`, `DB_NAME`, `DB_USER`, `TS`, `OUT_FILE`, `LOG_FILE`).

**Mandatory names and meanings (use these identifiers exactly):**

| Identifier | Role |
|------------|------|
| `NFS_DUMP_ROOT` | Directory that **must** be verified with `mountpoint` before any dump (equivalent role to the lab’s backup directory on NFS). |
| `PGDATABASE` | Database name passed to `pg_dump`. |
| `PG_OS_USER` | System user as whom the dump runs (typically `postgres`). |
| `RUN_STAMP` | Timestamp string; you **must** build it with **`date -u +%Y%m%dT%H%M%SZ`** (UTC). |
| `ARCHIVE_PATH` | Full path of the output dump file, built from `NFS_DUMP_ROOT`, `PGDATABASE`, and `RUN_STAMP`, ending in **`.dump`**. |
| `AUDIT_LOG` | Log file path; you **must** set this to **`$HOME/linux-labs/part02/backup-audit.log`** (expand `$HOME` appropriately in the script so it resolves reliably). |

**Behavior (minimum):**

- Use `set -euo pipefail` (or equivalent strictness your instructor accepts).
- If `NFS_DUMP_ROOT` is **not** a mountpoint, log an **`ERROR`** line to `AUDIT_LOG` (and stderr is fine) and exit non-zero—**do not** run `pg_dump` in that case.
- On success, create the dump at `ARCHIVE_PATH` using **`pg_dump -Fc`** for `PGDATABASE` as `PG_OS_USER`, and append a **`SUCCESS`** line to `AUDIT_LOG` mentioning the archive path.

**Evidence to submit:**

1. Screenshot of the **first ~30 lines** of your script showing the **six** identifiers above in use (not merely commented).
2. Screenshot of a **successful** manual run (terminal) **or** a principled failure (e.g. NFS down) where the **`ERROR`** path clearly references mount state—your submission should match what is true on your VM at capture time.
3. Screenshot of **`tail`** of `AUDIT_LOG` showing the matching **`SUCCESS`** or **`ERROR`** lines.

---

## Phase 5 — Nginx static routing (moderate challenge)

**Requirement:** Configure Nginx (package install as in the lab) to serve a small static site from a **`root`** directory you control (under `/var/www/...` or another path you justify). The site must meet **all** of the following:

1. **`/`** — Serves an `index.html` home page (file-level layout as in lab 08).
2. **`location /`** — Uses exactly:  
   `try_files $uri $uri.html $uri/ =404;`
3. **`/faq`** — Serves your FAQ page via the **pretty URL** (i.e. backed by `faq.html` in the same folder, without requiring a `/faq/` directory on disk).
4. **`/home`** — Must show the **same** document as **`/`** using an **exact** match location (`location = /home { ... }`), not by duplicating unrelated files.
5. **`/team`** — Must serve **`team.html`** via the pretty URL (create `team.html`; no `team/` directory required).
6. **`/secret`** — Must return **404** (no file served for that path).

**Evidence to submit:** Screenshot of **clean** config test output for Nginx, plus **`curl -I`** (or equivalent) screenshots for **`/`**, **`/home`**, **`/faq`**, **`/team`**, and **`/secret`** showing the expected status codes (200 family for the first four as appropriate, **404** for `/secret`).

---

## Phase 6 — Concept memo (three questions)

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
  - screenshots for Phases **1–5** (label files clearly, e.g. `Part02-Phase1.png`, …), and  
  - the **Phase 6** answers in the same PDF **or** as a separate `Part02-Memo-<YourName>.pdf` / `.docx`.

**Checklist before send:**

- [ ] Phase 1–5 evidence is present and readable  
- [ ] Phase 6 answered in prose  
- [ ] No secrets or private keys in the package  
- [ ] Subject line includes your name  

---

## How participants will be evaluated

| Criterion | Looks for |
|-----------|-----------|
| Phase 1 | Group exists; directory and files match the stated permission policy |
| Phase 2 | Extra filesystem; mount at `/mnt/part02-data`; persistent `fstab`; marker file with required string |
| Phase 3 | User crontab; interval ≤ 5 minutes; log under Phase 1 directory; visible **`HEARTBEAT`** lines over time |
| Phase 4 | Script path correct; **six** mandated variable names; mount check before dump; `pg_dump -Fc`; `AUDIT_LOG` path as specified; UTC `RUN_STAMP` format; evidence matches behavior |
| Phase 5 | Required `try_files` line; `/`, `/home`, `/faq`, `/team`, `/secret` behaviors; config test clean |
| Phase 6 | Correct concepts in own words |

---

*This is Part 02. Part 01 is `01-LAB-EXPERIMENT-PROJECT-LINUX-part01.md`. A separate solution guide may be published later.*
