
# Operation: ZERO — ZeroTrust: Security Theatre

> A deliberately vulnerable cybersecurity training environment built for undergraduate students.  
> Investigate a fictional corporate breach, chain exploits, escalate privileges, and capture 17 flags.

## 📋 Table of Contents

- [About the Project](#about-the-project)
- [Scenario](#scenario)
- [Architecture](#architecture)
- [System Requirements](#system-requirements)
- [Setup & Deployment](#setup--deployment)
- [Investigation Structure](#investigation-structure)
- [Flag Reference](#flag-reference)
- [Bonus Challenges](#bonus-challenges)
- [Tools Used](#tools-used)
- [Real-World Mitigations](#real-world-mitigations)
- [Project Documents](#project-documents)
- [Team](#team)
- [Disclaimer](#disclaimer)

---

## About the Project

**Operation: ZERO** is a portable, self-contained Capture the Flag (CTF) training environment developed as part of the ICT302 unit at Murdoch University. The system is designed to bridge the gap between theoretical cybersecurity concepts and hands-on offensive security practice.

The environment simulates a real-world enterprise breach investigation. Students take the role of a forensic investigator tasked with tracing the actions of a rogue insider — working through reconnaissance, credential discovery, SSH access, file forensics, privilege escalation, and data recovery in a structured, narrative-driven sequence.

**Key features:**
- 17 core flags arranged in a chained, dependency-driven attack path
- 6 optional bonus challenges covering OWASP Top 10 and CIA Triad concepts
- Two escalation paths to root: GTFOBins misconfiguration + Metasploit (CVE-2004-2687)
- Fully offline, isolated virtual network — no internet exposure required
- Three-tier walkthrough documentation (step-by-step, hinted, mission outline)
- Snapshot-based reset for repeatable lab sessions
- Flag verification via SHA-256 using the included `ZERO_Platform.html` portal

---

## Scenario

> *ZeroTrust Solutions publicly built its identity around a promise — that no user, no system, and no connection would ever be implicitly trusted.*
>
> *Their Senior DevOps Engineer, Jordan Mercer, built and maintained the server infrastructure from the ground up. When the company launched an internal review into unusual file access patterns traced to his account, Jordan was suspended and formally terminated. His access credentials were flagged for revocation. That revocation never completed.*
>
> *In the days that followed, someone used Jordan's credentials to re-enter the system. Internal documents were accessed. A client data register was altered. An unknown software package was installed on a non-standard port. Then the connection went quiet.*
>
> *Your team has been engaged to investigate. The attacker operated under an alias discovered in recovered communications — **GhostOps**. Follow the trail.*

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              HOST MACHINE                   │
│                                             │
│   ┌──────────────┐    ┌───────────────────┐ │
│   │  Kali Linux  │    │  Ubuntu Server    │ │
│   │  (Attacker)  │◄──►│  18.04.6 LTS      │ │
│   │              │    │  (Target)         │ │
│   │ 192.168.100  │    │  IP via DHCP      │ │
│   │    .100      │    │                   │ │
│   └──────────────┘    └───────────────────┘ │
│                                             │
│         Host-Only Network (vboxnet0)        │
│         Isolated — No Internet Route        │
└─────────────────────────────────────────────┘
```

| Component | Details |
|-----------|---------|
| Target OS | Ubuntu Server 18.04.6 LTS |
| Attacker OS | Kali Linux (latest stable) |
| Virtualisation | VirtualBox (OVF 2.0 / OVA) |
| Network | Host-Only adapter — fully isolated |
| Kali IP | `192.168.100.100` (static) |
| Target IP | Assigned via DHCP — students discover it |
| Services | Apache 2.4.29, OpenSSH 7.6p1, distccd (port 3632) |

---

## System Requirements

| Resource | Target VM | Attacker VM |
|----------|-----------|-------------|
| RAM | 2 GB minimum | 4 GB recommended |
| CPU | 2 cores | 2 cores |
| Disk | 20 GB | 40 GB |

- VirtualBox 6.x or later
- Host machine with hardware virtualisation support (Intel VT-x / AMD-V)
- Both VMs must be on the same Host-Only adapter

---

## Setup & Deployment

### 1. Import the VMs

```bash
# Import target VM
File → Import Appliance → ZERO_Target_v1.0.ova

# Import attacker VM
File → Import Appliance → ZERO_Kali_v1.0.ova
```

### 2. Configure the Host-Only Network

In VirtualBox: **File → Host Network Manager**

- Create adapter: `vboxnet0`
- Subnet: `192.168.100.0/24`
- Enable DHCP server (range: `192.168.100.101–200`)
- Assign both VMs to this adapter only

### 3. Verify Isolation

```bash
# From inside the target VM — this must FAIL
curl -m 5 http://example.com
# Expected: connection timed out
```

### 4. Start the Lab

Boot both VMs. From Kali, discover the target:

```bash
nmap -sn 192.168.100.0/24
```

Then follow the Student Mission Handout (`ZERO_Student_Handout_v5.pdf`) or begin with Act 1.

### Resetting Between Sessions

Revert both VMs to the `ZERO_CLEAN_v1.0` snapshot before each student session.

---

## Investigation Structure

The investigation is structured across **7 Acts**, progressing from passive reconnaissance to full system compromise and data recovery.

| Act | Theme | Difficulty | Flags |
|-----|-------|------------|-------|
| Act 1 | Reconnaissance | Easy | F01 – F03 |
| Act 2 | Archive Extraction | Easy | F04 – F06 |
| Act 3 | Initial Access | Easy | F07 – F08 |
| Act 4 | File Forensics | Medium | F09 – F12 |
| Act 5 | Privilege Escalation | Medium – Hard | F13 – F15 |
| Act 6 | Identity Exposure | Hard | F16 |
| Act 7 | Data Recovery | Hard | F17 |
| Bonus | Optional Challenges | Mixed | BF01 – BF06 |

Each flag is validated against a SHA-256 hash using the included `ZERO_Platform.html` flag submission portal — no internet required.

---

## Flag Reference

| Flag | Code | Location | Technique |
|------|------|----------|-----------|
| F01 | `HB-SCAN-1337` | SSH pre-auth banner | Service banner reading |
| F02 | `HB-ROBOT-2448` | `/robots.txt` comment | Web reconnaissance |
| F03 | `HB-SOURCE-3559` | HTML source comment | Source code inspection |
| F04 | `HB-BACKUP-4673` | `deployment_notes.txt` (in backup zip) | Archive extraction |
| F05 | `HB-SLACK-5784` | `slack_export.json` (in backup zip) | JSON analysis |
| F06 | `HB-CREDS-6895` | `config.env` (in backup zip) | Credential extraction |
| F07 | `HB-SHELL-7906` | `/home/devops/.flag_access.txt` | SSH access |
| F08 | `HB-HIST-8017` | `/home/devops/.bash_history` | Shell history forensics |
| F09 | `HB-NOTES-9128` | `/home/devops/sudoers_update.txt` | File enumeration |
| F10 | `HB-META-0239` | EXIF metadata in `zt_architecture_diagram.png` | Metadata analysis |
| F11 | `HB-CRON-1340` | `/etc/cron.d/zt_health_check` | Cron job inspection |
| F12 | `HB-CONF-2451` | `/etc/zt_app.env` | World-readable config file |
| F13 | `HB-SUDO-3562` | sudo lecture file | `sudo -l` enumeration |
| F14 | `HB-PRIV-4673` | `/root/board_memo.txt` | GTFOBins (`find`) escalation |
| F15 | `HB-GHOST-5784` | `/root/ghost_config.txt` | Metasploit distcc exploit |
| F16 | `HB-UNMASK-6895` | `/root/.ghost_identity.txt` | Root filesystem enumeration |
| F17 | `HB-RESTORE-7906` | `/var/backup/zt_original/client_data.txt.bak` | File diff / data recovery |

---

## Bonus Challenges

Six optional challenges are available beyond the main investigation. They are not required for completion but award additional marks when discovered. All are accessible via the web server on port 80 — no additional services required.

| Flag | Code | Concept | OWASP / Framework |
|------|------|---------|-------------------|
| BF01 | `HB-ACCESS-4456` | Broken Access Control — direct URL to `/admin/reports.html` | OWASP A01:2021 |
| BF02 | `HB-HIDDEN-7234` | Hidden staging directory — requires `gobuster` enumeration | Reconnaissance |
| BF03 | `HB-ZEROTRUST-7789` | Auto-login script served over unauthenticated HTTP | Zero Trust failure |
| BF04 | `HB-CIA-2024` | Client data exposed in `/public/` directory | CIA Triad — Confidentiality |
| BF05 | `HB-IDOR-6678` | IDOR — `profile.php?user_id=0` exposes admin account | OWASP A01:2021 |
| BF06 | `HB-ENCODE-8123` | Double Base64 encoded flag in `data_dump.txt` | Data obfuscation |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network and port discovery |
| `nc` (netcat) | Service banner reading |
| `curl` / `wget` | HTTP requests and file retrieval |
| `ssh` | Remote shell access |
| `unzip` | Archive extraction |
| `exiftool` | EXIF metadata analysis |
| `gobuster` | Web directory enumeration |
| `msfconsole` | Metasploit Framework — distcc exploit |
| `sudo -l` / `find` | Privilege enumeration and GTFOBins escalation |
| `diff` | File comparison for data recovery |

---

## Real-World Mitigations

Every flag in this lab maps to a real misconfiguration. Key mitigations demonstrated:

- **Exposed backup archives** → Never store deployment artifacts in the web root. Enforce authentication on all backup paths.
- **Hardcoded credentials** → Use a secrets manager (HashiCorp Vault, AWS Secrets Manager). Never commit credentials to config files.
- **SSH banner leakage** → Sanitise or remove SSH banners after staff changes.
- **World-readable config files** → Restrict sensitive files to `chmod 600` owned by the service account.
- **GTFOBins sudo misconfiguration** → Never grant `NOPASSWD` sudo for binaries listed on [gtfobins.github.io](https://gtfobins.github.io). Audit sudoers quarterly.
- **distcc listening on 0.0.0.0** → Restrict `ALLOWEDNETS` to specific build hosts only, or disable when not in active use.
- **Shell history exposure** → Set `HISTFILE=/dev/null` for service accounts. Restrict `.bash_history` to `chmod 600`.
- **Unrevoked credentials** → Implement and complete a leaver checklist before an employee's last day.
- **IDOR / Broken Access Control** → Implement server-side authorisation checks on every object reference.

---

## Project Documents

All formal project documentation is included in the `/docs` directory.

| Document | Description |
|----------|-------------|
| `R_A_Final.docx` | Requirements & Analysis — functional/non-functional requirements |
| `TD03_Project_Management_Plan_Final.docx` | Project Management Plan — WBS, schedule, risk, communications |
| `TD03CyberStorm_DesignDoc_Final.docx` | Design Document — data, process, architecture, interface design |
| `ZERO_Implementation_Plan_v3.docx` | Implementation Plan — 10-phase, 23-step verified build guide |
| `ZERO_Student_Handout_v5_final_draft.docx` | Student Mission Handout — 7-act investigation guide with hints |
| `ZEROTRUST_CTF_TESTING_PLAN.docx` | Testing Plan — 54 test cases across 9 categories, all passed |

---

## Team

**TD03 CyberStorm** — Murdoch University, ICT302, March 2026




---

## Disclaimer

> **This environment is intentionally vulnerable. It is designed exclusively for use in authorised, isolated training environments.**
>
> - Do not deploy this VM on any network with internet access or production infrastructure.
> - Do not use techniques demonstrated in this lab against systems you do not own or have explicit written permission to test.
> - All exploitation activities must remain within the isolated Host-Only virtual network.
> - This project was developed for academic purposes under the supervision of Murdoch University.
>
> *Distributed for authorised training use only. — TD03 CyberStorm*
