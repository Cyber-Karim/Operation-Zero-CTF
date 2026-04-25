# 🌐 Network Configuration — Operation: ZERO

> Complete reference for the virtual network architecture used in the Operation: ZERO lab environment.  
> This document covers topology, adapter configuration, IP addressing, isolation enforcement, and verification procedures.

---

## Table of Contents

- [Network Design Philosophy](#network-design-philosophy)
- [Topology Overview](#topology-overview)
- [Network Architecture Diagram](#network-architecture-diagram)
- [IP Address Reference](#ip-address-reference)
- [VirtualBox Adapter Configuration](#virtualbox-adapter-configuration)
  - [Host-Only Adapter (vboxnet0)](#host-only-adapter-vboxnet0)
  - [DHCP Server Settings](#dhcp-server-settings)
  - [Target VM — Adapter Settings](#target-vm--adapter-settings)
  - [Kali VM — Adapter Settings](#kali-vm--adapter-settings)
- [Static IP Configuration — Kali Linux](#static-ip-configuration--kali-linux)
- [DHCP Behaviour — Target VM](#dhcp-behaviour--target-vm)
- [Network Isolation Enforcement](#network-isolation-enforcement)
- [Open Ports & Service Exposure](#open-ports--service-exposure)
- [Traffic Flow by Investigation Stage](#traffic-flow-by-investigation-stage)
- [Isolation Verification Procedures](#isolation-verification-procedures)
- [Why This Architecture Was Chosen](#why-this-architecture-was-chosen)
- [Alternative Designs Considered](#alternative-designs-considered)
- [Network Troubleshooting](#network-troubleshooting)

---

## Network Design Philosophy

The Operation: ZERO lab network is designed around three principles:

**1. Complete isolation**  
All exploitation traffic must remain within the Host-Only virtual network. The target VM must have no internet route at any point during the lab. This prevents the malicious cron job (F11) from reaching its external C2 IP, ensures lab containment, and prevents accidental exposure of deliberately vulnerable services to external networks.

**2. Realistic discovery**  
Students are not given the target IP directly. They must perform a subnet sweep using `nmap` to discover it — mirroring real-world reconnaissance workflows. The Kali attacker VM uses a known static IP so students always have a reliable starting point.

**3. Reproducibility**  
The DHCP-based target IP assignment ensures the lab works on any host machine without requiring manual static IP configuration on the target. The network resets cleanly on snapshot restore, and the DHCP lease renews automatically on each boot.

---

## Topology Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         HOST MACHINE                             │
│                                                                  │
│  ┌─────────────────────┐         ┌────────────────────────────┐  │
│  │    Kali Linux VM    │         │    Ubuntu Server 18.04 VM  │  │
│  │   (ZERO_Kali)       │         │    (ZeroTrust_Target)      │  │
│  │                     │         │                            │  │
│  │  eth1               │         │  enp0s3                    │  │
│  │  192.168.100.100/24 │◄───────►│  192.168.100.x/24 (DHCP)  │  │
│  │  (static)           │         │  (discovered via nmap)     │  │
│  └─────────────────────┘         └────────────────────────────┘  │
│            │                                   │                  │
│            └───────────────┬───────────────────┘                  │
│                            │                                      │
│               ┌────────────▼────────────┐                        │
│               │   vboxnet0              │                        │
│               │   Host-Only Adapter     │                        │
│               │   192.168.100.1/24      │                        │
│               │   DHCP: .101 – .200     │                        │
│               └─────────────────────────┘                        │
│                                                                  │
│                    ❌ NO ROUTE TO INTERNET                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Network Architecture Diagram

```
INTERNET
    │
    │  ← No route from lab VMs
    │
┌───▼──────────────────────────────────────────────────────────┐
│  HOST MACHINE PHYSICAL NETWORK INTERFACE                     │
│  (Wi-Fi / Ethernet — used by host OS only)                   │
└──────────────────────────────────────────────────────────────┘
                          │
                          │  Host-Only boundary — VMs cannot cross this
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  VirtualBox Host-Only Network — vboxnet0                    │
│  Subnet: 192.168.100.0/24                                   │
│  Gateway: 192.168.100.1 (host adapter — no routing)         │
│  DHCP Server: 192.168.100.2  Range: .101–.200               │
│                                                             │
│   ┌──────────────────┐         ┌─────────────────────────┐  │
│   │  Kali Linux VM   │  TCP/IP  │  Ubuntu Server VM       │  │
│   │                  │◄────────►│                         │  │
│   │  eth1            │          │  enp0s3                 │  │
│   │  192.168.100.100 │          │  192.168.100.x (DHCP)   │  │
│   │                  │          │                         │  │
│   │  Ports used:     │          │  Listening ports:       │  │
│   │  - Ephemeral     │          │  22   — SSH             │  │
│   │    (outbound)    │          │  80   — HTTP (Apache)   │  │
│   │  - 4444 (LHOST   │          │  3632 — distccd         │  │
│   │    for Msfconsole│          │                         │  │
│   │    reverse shell)│          │                         │  │
│   └──────────────────┘         └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## IP Address Reference

| Host | Interface | IP Address | Assignment | Notes |
|------|-----------|------------|------------|-------|
| VirtualBox Host Adapter | `vboxnet0` | `192.168.100.1` | Static | Host-side gateway — no internet routing |
| VirtualBox DHCP Server | — | `192.168.100.2` | Static | Internal DHCP service only |
| Kali Linux (attacker) | `eth1` | `192.168.100.100` | Static | Known to students — always the attacker address |
| Ubuntu Target | `enp0s3` | `192.168.100.101–200` | DHCP | Changes on restore — students discover via `nmap` |
| Metasploit LHOST | — | `192.168.100.100` | Static | Kali IP — set in msfconsole for reverse shell (F15) |

> 💡 The target IP is intentionally not documented for students. Discovering it via `nmap -sn 192.168.100.0/24` is the first step of the investigation (Act 1 — Reconnaissance).

---

## VirtualBox Adapter Configuration

### Host-Only Adapter (vboxnet0)

Configure this in **File → Host Network Manager** before importing the VMs.

| Setting | Value |
|---------|-------|
| Adapter Name | `vboxnet0` |
| IPv4 Address | `192.168.100.1` |
| IPv4 Network Mask | `255.255.255.0` |
| IPv6 | Disabled |

### DHCP Server Settings

| Setting | Value |
|---------|-------|
| Enable Server | ✅ Yes |
| Server Address | `192.168.100.2` |
| Server Mask | `255.255.255.0` |
| Lower Address Bound | `192.168.100.101` |
| Upper Address Bound | `192.168.100.200` |

### Target VM — Adapter Settings

Navigate to: **ZeroTrust_Target → Settings → Network**

| Adapter | Mode | Adapter Name | Status |
|---------|------|--------------|--------|
| Adapter 1 | Host-Only Adapter | `vboxnet0` | ✅ Enabled |
| Adapter 2 | NAT | — | ❌ **Disabled** |

> ⚠️ **Adapter 2 must be disabled.** The NAT adapter was used during the build phase to install packages. It must be removed before student use. If NAT remains enabled, the target VM will have a default internet route — breaking isolation and allowing the cron job in F11 to reach its external C2 address (`185.220.101.47`).

### Kali VM — Adapter Settings

Navigate to: **ZERO_Kali → Settings → Network**

| Adapter | Mode | Adapter Name | Status |
|---------|------|--------------|--------|
| Adapter 1 | Host-Only Adapter | `vboxnet0` | ✅ Enabled |
| Adapter 2 | — | — | ❌ Disabled |

---

## Static IP Configuration — Kali Linux

The Kali attacker VM is configured with a static IP so students always know the attacker address. This is required for the Metasploit reverse shell payload (`LHOST`).

### Verify current assignment

```bash
ip addr show eth1
# Expected: inet 192.168.100.100/24
```

### If the static IP is not set — apply manually

```bash
sudo ip addr add 192.168.100.100/24 dev eth1
sudo ip link set eth1 up
```

### Make the static IP permanent

Edit `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces
```

Add or confirm the following block:

```
auto eth1
iface eth1 inet static
    address 192.168.100.100
    netmask 255.255.255.0
```

Apply without rebooting:

```bash
sudo ifdown eth1 && sudo ifup eth1
```

### Verify

```bash
ip addr show eth1
# Expected: inet 192.168.100.100/24 brd 192.168.100.255 scope global eth1
```

---

## DHCP Behaviour — Target VM

The target VM obtains its IP address automatically from the VirtualBox DHCP server on `vboxnet0`.

### How it works

1. On boot, the target VM sends a DHCP discover broadcast on the `192.168.100.0/24` subnet
2. The VirtualBox DHCP server (`192.168.100.2`) responds with an available lease in the `.101–.200` range
3. The target VM configures its interface and becomes reachable

### Check the assigned IP (from target VM console)

```bash
ip addr show enp0s3
# or
hostname -I
```

### Check the assigned IP (from Kali — discovery step)

```bash
nmap -sn 192.168.100.0/24
```

Expected output includes something like:

```
Nmap scan report for victim01 (192.168.100.107)
Host is up (0.00046s latency).
```

### Important — IP changes on snapshot restore

After restoring the `ZERO_CLEAN_v1.0` snapshot, the target VM will boot fresh and receive a new DHCP lease. The IP may differ from the previous session. Students must run the subnet sweep again at the start of each session. This is intentional — it reinforces the reconnaissance habit.

---

## Network Isolation Enforcement

Isolation is the most critical safety property of the lab. The following controls enforce it.

### Control 1 — No NAT adapter on target

The target VM has no NAT adapter, so it has no default route to the internet. Confirmed by:

```bash
# From inside the target VM
ip route show
# Expected: only 192.168.100.0/24 route — no default (0.0.0.0) route
```

### Control 2 — Host-Only adapter provides no routing

VirtualBox Host-Only adapters do not route traffic to the host's physical network or internet by design. The `vboxnet0` adapter is a closed subnet.

### Control 3 — Internet access verification

Run this from inside the target VM after setup:

```bash
curl -m 5 http://example.com && echo 'FAIL: internet reachable' || echo 'PASS: isolated correctly'
```

Expected output:

```
PASS: isolated correctly
```

Run this from Kali:

```bash
ping -c 2 8.8.8.8
# Expected: Network unreachable  OR  100% packet loss
```

### Control 4 — distcc service binding

The `distcc` daemon is configured to listen on `0.0.0.0:3632` but is only reachable within the `192.168.100.0/24` subnet. The `ALLOWEDNETS` setting in `/etc/default/distcc` restricts which hosts can connect:

```
ALLOWEDNETS="192.168.100.0/24"
LISTENER="0.0.0.0"
```

Because the target VM has no internet route, this service is inherently inaccessible from outside the lab — even though it listens on all interfaces.

---

## Open Ports & Service Exposure

The following ports are intentionally open on the target VM and form part of the attack surface:

| Port | Protocol | Service | Version | CTF Role |
|------|----------|---------|---------|----------|
| 22 | TCP | OpenSSH | 7.6p1 Ubuntu | SSH banner (F01), credential access (F07) |
| 80 | TCP | Apache httpd | 2.4.29 | Web recon (F02, F03), archive (F04–F06), bonus flags |
| 3632 | TCP | distccd | 3.1 | Metasploit exploit path — CVE-2004-2687 (F15) |

### Confirming open ports from Kali

```bash
nmap -sV <target_ip>
```

Expected output:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
3632/tcp open  distccd distccd v1 ((GNU) 3.1)
```

### Ports that should NOT be open

The following services are disabled or not installed. If any of these appear in a scan, something is misconfigured:

| Port | Service | Action |
|------|---------|--------|
| 21 | FTP | Should not be installed |
| 23 | Telnet | Should not be installed |
| 443 | HTTPS | Not configured |
| 3306 | MySQL | Not installed |
| 8080 | HTTP alt | Not in use (admin misdirection path only via Apache) |

---

## Traffic Flow by Investigation Stage

This table maps each investigation act to its corresponding network traffic type.

| Act | Stage | Traffic Type | Source → Destination | Port |
|-----|-------|-------------|----------------------|------|
| Act 1 | Host discovery | ICMP / ARP | Kali → subnet broadcast | — |
| Act 1 | Port scan | TCP SYN | Kali → Target | 1–65535 |
| Act 1 | SSH banner read | TCP | Kali → Target | 22 |
| Act 1 | HTTP recon | TCP / HTTP GET | Kali → Target | 80 |
| Act 2 | robots.txt retrieval | HTTP GET | Kali → Target | 80 |
| Act 2 | Backup archive download | HTTP GET | Kali → Target | 80 |
| Act 3 | SSH login | TCP / SSH | Kali → Target | 22 |
| Acts 4–6 | Post-exploitation | SSH session | Kali → Target | 22 |
| Act 5 (F15) | Metasploit reverse shell | TCP (reverse) | Target → Kali | 4444 |
| Act 5 (F14) | GTFOBins escalation | Local only | Target (local) | — |
| F11 (cron) | C2 callback attempt | TCP (outbound) | Target → `185.220.101.47` | 80 |

> 💡 The F11 cron job attempts to call `185.220.101.47` every 30 minutes. Because the target has no internet route, this connection times out silently — as designed. Students observe it in the cron file and record the external IP as forensic evidence. The connection must never succeed.

### Metasploit reverse shell flow (F15)

```
Kali (192.168.100.100:4444) ◄── reverse TCP ── Target (192.168.100.x)
         │                                              │
    msfconsole                                    distccd daemon
    multi/handler                              executes payload via
    listening on 4444                          CVE-2004-2687
```

The exploit sends a command to the distccd daemon on port 3632. The target initiates the reverse connection back to Kali on port 4444. This is why `LHOST` must be set to `192.168.100.100` (the Kali static IP) — not `localhost` or `0.0.0.0`.

---

## Isolation Verification Procedures

Run these checks before every lab session.

### Full isolation checklist

```bash
# 1. Confirm target has no default internet route
#    (run from inside target VM)
ip route show | grep default
# Expected: no output

# 2. Confirm target cannot reach internet
#    (run from inside target VM)
curl -m 5 http://example.com || echo 'ISOLATED OK'
# Expected: ISOLATED OK

# 3. Confirm Kali cannot reach internet
#    (run from Kali)
ping -c 2 8.8.8.8
# Expected: Network unreachable or 100% packet loss

# 4. Confirm Kali can reach target
#    (run from Kali)
ping -c 4 <target_ip>
# Expected: 0% packet loss

# 5. Confirm all required ports are open
#    (run from Kali)
nmap -p 22,80,3632 <target_ip>
# Expected: all three ports shown as open
```

---

## Why This Architecture Was Chosen

Three alternative network architectures were evaluated before settling on the Host-Only model.

| Architecture | Considered | Decision |
|-------------|-----------|----------|
| **Host-Only (chosen)** | ✅ | Fully isolated, no internet exposure, DHCP supported, portable across host machines |
| NAT Network | Evaluated | Rejected — provides internet access by default, requires additional firewall rules to isolate |
| Bridged Adapter | Evaluated | Rejected — exposes the vulnerable VM to the host's physical LAN, unacceptable security risk |
| Internal Network (no host) | Evaluated | Rejected — host cannot interact with VMs for management; DHCP requires separate VM server |

The Host-Only adapter provides the best balance of isolation, portability, and simplicity for a single-instructor lab environment. It requires no external infrastructure, works identically on Windows, macOS, and Linux hosts, and is reset automatically on snapshot restore.

---

## Alternative Designs Considered

### NAT Network

A NAT Network in VirtualBox allows multiple VMs to share a private address space while the host performs NAT for outbound traffic. This would give both VMs internet access unless explicitly blocked with firewall rules.

**Rejected because:** Internet access on the target VM would allow the F11 cron job to actually call its external IP, exposing `185.220.101.47` to real traffic. It would also require iptables rules to be configured and maintained, adding complexity and a potential misconfiguration risk.

### Bridged Adapter

A Bridged Adapter places the VM directly on the host's physical LAN, receiving an IP from the real DHCP server.

**Rejected because:** This exposes the deliberately vulnerable Ubuntu server — with SSH, Apache, and distcc open — directly to any device on the same physical network. This is a serious security risk in any shared lab or campus environment, and violates NFR-02 (Security Containment).

### Docker Networking

Containers with a custom bridge network could achieve similar isolation.

**Rejected because:** Docker abstracts key Linux networking and filesystem concepts that are central to the learning objectives. Students would lose direct exposure to real service configurations, interface naming, and system-level networking behaviour.

---

## Network Troubleshooting

**VMs cannot ping each other**
- Confirm both VMs are assigned to `vboxnet0` (not different adapters)
- Confirm the Host-Only adapter exists: **File → Host Network Manager**
- Confirm the DHCP server is enabled on `vboxnet0`
- Check the target received a lease: `ip addr show` (look for `192.168.100.x`)
- Check Kali has its static IP: `ip addr show eth1`

**Target IP not appearing in nmap scan**
```bash
# Try ARP scan — faster and more reliable on local subnets
sudo arp-scan 192.168.100.0/24
# or
sudo nmap -sn --send-ip 192.168.100.0/24
```

**Target has internet access (isolation failure)**
- Go to **ZeroTrust_Target → Settings → Network**
- Ensure Adapter 2 is fully disabled (unchecked, not just set to NAT with no cable)
- Reboot the target VM and retest with `curl -m 5 http://example.com`

**Kali static IP lost after reboot**
```bash
sudo ip addr add 192.168.100.100/24 dev eth1
sudo ip link set eth1 up
# Then make permanent via /etc/network/interfaces (see Static IP Configuration section)
```

**Port 3632 closed — distcc not running**
```bash
# SSH into target as admin
sudo systemctl start distcc
sudo systemctl enable distcc
sudo ss -tlnp | grep 3632
# Expected: LISTEN on 0.0.0.0:3632
```

**Metasploit reverse shell not connecting**
- Confirm `LHOST` is `192.168.100.100` — not `127.0.0.1` or the host machine's IP
- Confirm `RHOSTS` matches the current target DHCP IP (check with nmap first)
- Confirm port 3632 is open: `nmap -p 3632 <target_ip>`
- Confirm no host firewall is blocking port 4444 on Kali

**Port 80 not responding**
```bash
# SSH into target
sudo systemctl start apache2
sudo systemctl status apache2
curl http://localhost/robots.txt
```

---

*Operation: ZERO — TD03 CyberStorm | Murdoch University | March 2026*  
*For authorised training use only.*
