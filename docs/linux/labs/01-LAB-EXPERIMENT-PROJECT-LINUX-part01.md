# Lab examination — take-home project (Part 01)

**Coverage:** Draws on ideas from class labs **01–04** (SSH/users/sudo, permissions, packages and systemd, basic networking). It does **not** require you to demonstrate every topic from those labs—only what this brief asks for.

**Format:** One small **scenario** with **phases**. You decide **how** to accomplish each requirement using what you learned. This paper intentionally **does not** list shell commands; your evidence should show that you actually performed the work on **your Ubuntu (or Debian-style) VM**.

---

## Scenario

Your team is handing you a fresh Linux VM that will eventually serve an internal static website. Before the application team drops in content, your manager wants a **short evidence pack** showing that:

- administrative access is set up sensibly,
- a protected area for site files exists on disk,
- the distribution’s web server package is installed and managed like a proper service,
- you can orient yourself on the network.

You will submit **screenshots and a brief written addendum** as described below.

---

## Allowed materials

- Your notes and the course lab documents: `01-LAB-SSH-USERS-SUDO.md` through `04-LAB-NETWORKING-TOOLS.md`.
- Your lab VM only.

Do **not** include passwords, private SSH key material, or full `authorized_keys` dumps in your submission. Redact if necessary.

---

## Phase 1 — Administrative user (access model)

**Requirement:** Create a **second** user account (not your own usual login, and not root) that is meant for ongoing administration. That account must be able to run privileged commands using the **normal Ubuntu sudo mechanism** (membership in the sudo group, not a custom root password share).

**Evidence to submit:** Terminal output (screenshot) that, taken together, proves the account exists and is in the sudo group, **without** showing a password or key.

---

## Phase 2 — Site directory and permissions (collaboration model)

**Requirement:**

1. Create a **Unix group** whose name you choose (for example a team name—avoid reserved system names).
2. Under your home directory, create a dedicated directory that will hold future site files. Inside it, place **one** small text file (any harmless content).
3. Set **ownership and permissions** so that:
   - you (as owner) retain full control of the directory and its contents;
   - members of your new group can **enter** the directory and **read** files inside;
   - users who are **not** you and **not** in that group have **no** access to the directory.

**Evidence to submit:** Screenshots of metadata listings that clearly show the directory and file **owner**, **group**, and **permission bits** (the same style of detail you practiced in the permissions lab).

---

## Phase 3 — Web server from packages (service lifecycle)

**Requirement:** Install the **nginx** web server using the distribution’s package tooling. Ensure it is **running at the time you capture evidence**, and ensure it is **configured to start automatically** when the machine boots to normal multi-user operation.

**Evidence to submit:** Screenshots that **together** demonstrate both “running now” and “enabled for future boots.” Your manager should not have to guess—make the distinction obvious from the output you choose to show.

---

## Phase 4 — Service troubleshooting awareness (logs)

**Requirement:** Show that you can **retrieve recent log lines** associated with the **nginx** systemd unit (the same unit name the package provides).

**Evidence to submit:** One screenshot showing a slice of that log output (timestamps or messages visible).

---

## Phase 5 — Network facts sheet (short appendix)

**Requirement:** Add a **half-page appendix** (PDF section or the body of your email below your screenshots) titled **“Network facts for this VM.”** In plain English, include:

- The **interface name(s)** and **IPv4 address(es)** you consider primary for this machine.
- Whether a **default route** exists, and if so, **to which gateway** and **through which interface** (describe in words; you may quote addresses from your output).
- What **`/etc/resolv.conf`** lists as the nameserver line(s), and **separately**, what the system reports as **upstream DNS** for at least one link (the lab material discusses how stub resolvers relate to upstream resolvers—use that knowledge).
- **One short paragraph** explaining how a packet destined for the public internet would leave this VM at the routing level (concept of default route vs directly connected subnets is enough).

**Evidence:** This appendix is **text** you write, informed by what you observed on the VM. You do **not** need to paste every command, but the facts must be **consistent** with a real machine (fabricated network reports are not acceptable).

---

## Phase 6 — Concept memo (two questions)

Answer in **your own words** (roughly **one short paragraph each**). No screenshots.

1. **systemd:** Explain the difference between **starting** a unit once and **enabling** it for boot. Mention what kind of entry appears under **`multi-user.target.wants`** when a typical service is enabled, and why merely starting a service does not create that boot-time link.

2. **DNS:** Explain why seeing **`127.0.0.53`** in `resolv.conf` on Ubuntu does **not** by itself mean “we use Google DNS.” Describe the role of the local stub resolver and where **upstream** DNS appears in the tooling from the networking lab.

---

## How to submit

Send everything to:

**faizulkhan56@gmail.com**

**Subject line (required):**

```text
Linux Lab Exam Part01 - <Your Full Name> - <Roll or ID if applicable>
```

**Attachments:**

- A **single ZIP** or **one PDF** containing:
  - screenshots for Phases **1–5** (label files: `Phase1.png`, `Phase2-dir.png`, …), and  
  - the **Network facts** appendix and **Phase 6** answers in the same PDF **or** as a separate `Part01-Memo-<YourName>.pdf` / `.docx`.

**Checklist before send:**

- [ ] Phase 1–5 evidence is present and readable  
- [ ] Phase 6 answered in prose  
- [ ] No secrets or private keys in the package  
- [ ] Subject line includes your name  

---

## How participants will be evaluated

| Criterion | Looks for |
|-----------|-----------|
| Phase 1 | Second admin-capable user exists; sudo via group; no leaked secrets |
| Phase 2 | Group created; directory permissions match the stated policy |
| Phase 3 | nginx from packages; clear evidence of **active now** vs **enabled at boot** |
| Phase 4 | Correct log source for the nginx **unit** |
| Phase 5 | Accurate, internally consistent network facts; stub vs upstream understood in text |
| Phase 6 | Correct concepts in own words |

---

*This is Part 01. Later exams may extend the scenario after additional labs.*
