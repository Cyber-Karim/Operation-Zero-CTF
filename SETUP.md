# 🛠️ Setup & Deployment Guide — Operation: ZERO

> Complete deployment instructions for instructors and lab administrators.  
> Estimated setup time: **20–30 minutes** on a prepared host machine.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Host Machine Requirements](#host-machine-requirements)
- [Step 1 — Install VirtualBox](#step-1--install-virtualbox)
- [Step 2 — Configure the Host-Only Network](#step-2--configure-the-host-only-network)
- [Step 3 — Import the Virtual Machines](#step-3--import-the-virtual-machines)
- [Step 4 — Verify VM Settings](#step-4--verify-vm-settings)
- [Step 5 — Boot and Validate](#step-5--boot-and-validate)
- [Step 6 — Verify Network Isolation](#step-6--verify-network-isolation)
- [Step 7 — Verify All Services](#step-7--verify-all-services)
- [Step 8 — Take the Clean Snapshot](#step-8--take-the-clean-snapshot)
- [Resetting Between Student Sessions](#resetting-between-student-sessions)
- [Distributing to Students](#distributing-to-students)
- [Credentials Reference](#credentials-reference)
- [Known Issues & Fixes](#known-issues--fixes)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before you begin, ensure you have the following:

- [ ] VirtualBox 6.1 or later installed on the host machine
- [ ] Both OVA files available:
  - `ZERO_Target_v1.0.ova` (Ubuntu Server 18.04.6 LTS — the vulnerable target)
  - `ZERO_Kali_v1.0.ova` (Kali Linux — the attacker machine)
- [ ] The `ZERO_Platform.html` flag submission portal (for students)
- [ ] The `ZERO_Student_Handout_v5.pdf` mission guide (for students)

---

## Host Machine Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 6 GB available for VMs | 8 GB or more |
| CPU | Dual-core with VT-x / AMD-V | Quad-core |
| Disk | 65 GB free | 80 GB free |
| OS | Windows 10, macOS 11, or Ubuntu 20.04+ | Any of the above |
| VirtualBox | 6.1 | 7.0+ |

> ⚠️ **Hardware virtualisation must be enabled in BIOS/UEFI** (Intel VT-x or AMD-V). If VMs fail to boot, check this setting first.

---

## Step 1 — Install VirtualBox

Download and install VirtualBox from the official site:

```
https://www.virtualbox.org/wiki/Downloads
```

Also install the **VirtualBox Extension Pack** (same page) — required for USB and OVA import compatibility.

---

## Step 2 — Configure the Host-Only Network

This step creates the isolated private network that both VMs will communicate over.

### Open the Host Network Manager

```
VirtualBox Menu → File → Host Network Manager
```

### Create the adapter

Click **Create**. A new adapter (`vboxnet0` on Linux/macOS, `VirtualBox Host-Only Ethernet Adapter` on Windows) will appear.

### Configure the adapter

Select the adapter and click **Properties**:

| Setting | Value |
|---------|-------|
| IPv4 Address | `192.168.100.1` |
| IPv4 Network Mask | `255.255.255.0` |

### Enable the DHCP server

Click the **DHCP Server** tab:

| Setting | Value |
|---------|-------|
| Enable Server | ✅ Checked |
| Server Address | `192.168.100.2` |
| Server Mask | `255.255.255.0` |
| Lower Address Bound | `192.168.100.101` |
| Upper Address Bound | `192.168.100.200` |

Click **Apply** and close the Host Network Manager.

> 💡 The Kali attacker VM uses a **static IP** (`192.168.100.100`) outside the DHCP range so students always know the attacker address. The target VM receives its IP via DHCP — discovering it is part of the first challenge (F01).

---

## Step 3 — Import the Virtual Machines

### Import the Target VM

```
VirtualBox Menu → File → Import Appliance
```

- Select `ZERO_Target_v1.0.ova`
- Leave all settings at their defaults
- Click **Import** and wait for completion

### Import the Kali Attacker VM

Repeat the process with `ZERO_Kali_v1.0.ova`.

---

## Step 4 — Verify VM Settings

After importing, confirm the network adapter settings for each VM before booting.

### Target VM (ZeroTrust_Target)

Right-click → **Settings → Network**:

| Adapter | Setting |
|---------|---------|
| Adapter 1 | Host-Only Adapter — `vboxnet0` |
| Adapter 2 | **Disabled** (NAT was used during build only) |

> ⚠️ **Adapter 2 must be disabled.** If NAT is enabled, the target VM will have internet access, breaking lab isolation and allowing the malicious cron job (F11) to reach its external IP.

### Kali VM (ZERO_Kali)

| Adapter | Setting |
|---------|---------|
| Adapter 1 | Host-Only Adapter — `vboxnet0` |

---

## Step 5 — Boot and Validate

Start both VMs.

### On the Kali VM

Confirm the static IP is assigned:

```bash
ip addr show eth1
# Expected: inet 192.168.100.100/24
```

If the IP is not set, apply it manually:

```bash
sudo ip addr add 192.168.100.100/24 dev eth1
sudo ip link set eth1 up
```

### On the Target VM

Log in as `admin` and check the DHCP-assigned IP:

```bash
ip addr show
# Note the inet address on the Host-Only adapter (e.g. 192.168.100.107)
```

### Confirm connectivity between VMs

From Kali:

```bash
ping -c 4 <target_ip>
# Expected: 0% packet loss
```

---

## Step 6 — Verify Network Isolation

The target VM must have **no internet access**. This is critical — the cron job in F11 calls an external IP and must fail silently.

### From inside the Target VM

```bash
curl -m 5 http://example.com && echo 'FAIL: internet reachable' || echo 'PASS: isolated correctly'
# Expected output: PASS: isolated correctly
```

### From Kali

```bash
ping -c 2 8.8.8.8
# Expected: Network unreachable OR 100% packet loss
```

> ❌ If either machine can reach the internet, recheck the adapter settings in Step 4 and ensure Adapter 2 (NAT) is disabled on the target.

---

## Step 7 — Verify All Services

From the Kali VM, confirm all required services are running on the target:

```bash
nmap -sV <target_ip>
```

Expected open ports:

| Port | Service | Purpose |
|------|---------|---------|
| 22 | OpenSSH 7.6p1 | SSH access (F01, F07) |
| 80 | Apache 2.4.29 | Web server (F02–F06) |
| 3632 | distccd | Metasploit exploit path (F15) |

> ⚠️ If port 3632 is not open, SSH into the target and run:
> ```bash
> sudo systemctl start distcc
> sudo systemctl enable distcc
> ```

### Verify the SSH banner

```bash
nc <target_ip> 22
# Expected: ZeroTrust Solutions banner containing HB-SCAN-1337
```

### Verify the web server

```bash
curl http://<target_ip>/robots.txt
# Expected: Disallow entries and HB-ROBOT-2448 comment
```

### Verify the backup archive

```bash
curl http://<target_ip>/backup/
# Expected: Apache directory listing showing prod_backup_final_v3.zip
```

---

## Step 8 — Take the Clean Snapshot

Once all services are verified and isolation is confirmed, take a snapshot of the target VM. This is the restore point used between student sessions.

### On the Target VM — shut down cleanly

```bash
sudo shutdown -h now
```

### In VirtualBox

```
Right-click ZeroTrust_Target → Snapshots → Take Snapshot
Name: ZERO_CLEAN_v1.0
Description: Verified clean state — all 17 flags confirmed
```

Repeat for the Kali VM:

```
Right-click ZERO_Kali → Snapshots → Take Snapshot
Name: KALI_CLEAN_v1.0
```

> ✅ The lab is now ready for student use.

---

## Resetting Between Student Sessions

Before each new student or group session, restore both VMs to their clean snapshots.

```
Right-click ZeroTrust_Target → Snapshots → Restore: ZERO_CLEAN_v1.0
Right-click ZERO_Kali → Snapshots → Restore: KALI_CLEAN_v1.0
```

This ensures:
- All flag files are restored to their original state
- No credentials or commands from the previous session persist
- The target IP may change after restore — students must rediscover it via `nmap`

---

## Distributing to Students

### Option A — USB Distribution

Copy both OVA files and supporting documents to a USB drive. Label it clearly:

```
HONEY BADGER LAB — AUTHORISED TRAINING USE ONLY
```

Students import the OVAs on their own machines following this guide.

### Option B — Local Network Share

Host the OVA files on an internal SMB or HTTP share within the lab network. Do **not** distribute via the public internet.

### Student package should include

- [ ] `ZERO_Target_v1.0.ova`
- [ ] `ZERO_Kali_v1.0.ova`
- [ ] `ZERO_Platform.html` (flag submission portal — works offline)
- [ ] `ZERO_Student_Handout_v5.pdf` (mission guide)
- [ ] `SETUP.md` (this file)

---

## Credentials Reference

> ⚠️ **Instructors only.** Do not distribute this section to students — credential discovery is part of the investigation.

| Account | Username | Password | Purpose |
|---------|----------|----------|---------|
| Target admin | `admin` | *(set during build)* | VM setup only — not used in the CTF |
| DevOps user | `devops` | `ZTsecure2024!` | Primary student SSH account (discovered via F06) |
| Root | `root` | N/A | Direct root SSH is disabled — escalation required |

---

## Known Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `$(hostname)` written literally into cron file | Heredoc delimiter not single-quoted during build | Already fixed in v2.0 — use `'CRONEOF'` not `CRONEOF` |
| F10 EXIF image not found | Generic `find` returned non-PNG file on some Ubuntu versions | Already fixed — using explicit path `/usr/share/pixmaps/debian-logo.png` |
| F13 sudo lecture only appears once | Default sudo behaviour shows lecture once per user | Already fixed — `Defaults lecture=always` in sudoers |
| distcc exploit (F15) fails | Module behaviour varies across VirtualBox/kernel versions | Test the full chain before each session. F14 (GTFOBins) is always available as fallback |
| BF05 IDOR page returns blank | PHP not installed on Apache | Run `sudo apt-get install -y php libapache2-mod-php && sudo systemctl restart apache2` |
| Target IP changes after snapshot restore | DHCP reassigns on boot | Expected behaviour — students run `nmap -sn 192.168.100.0/24` to rediscover |

---

## Troubleshooting

**VMs cannot ping each other**
- Confirm both are on the same Host-Only adapter (`vboxnet0`)
- Check the DHCP server is enabled in Host Network Manager
- Confirm the target VM received a DHCP lease: `ip addr show`

**Port 3632 not open after restore**
```bash
sudo systemctl start distcc && sudo systemctl enable distcc
sudo ss -tlnp | grep 3632
```

**Apache not serving files**
```bash
sudo systemctl start apache2
curl http://localhost/robots.txt
```

**SSH banner not showing**
```bash
sudo systemctl restart ssh
nc <target_ip> 22
```

**Metasploit distcc exploit opens no session**
- Confirm port 3632 is open from Kali: `nmap -p 3632 <target_ip>`
- Confirm `LHOST` is set to `192.168.100.100` (Kali static IP)
- Confirm `RHOSTS` is set to the current target IP
- If it still fails, use the GTFOBins escalation path (F14) instead

**OVA import fails**
- Ensure the VirtualBox Extension Pack is installed
- Try: `VBoxManage import ZERO_Target_v1.0.ova --vsys 0 --eula accept`

---

*Operation: ZERO — TD03 CyberStorm | Murdoch University | March 2026*  
*For authorised training use only.*
