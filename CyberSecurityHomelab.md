# Security Homelab

A self-contained cybersecurity lab for practicing network defense, detection engineering, and offensive security against my own infrastructure. Built on a Proxmox virtualization host behind a segmented UniFi network, with pfSense/Suricata for inline IDS/IPS and Security Onion for network security monitoring.

This repo documents the build so it can be rebuilt, extended, and used as a portfolio of hands-on work.

---

## Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Network Architecture](#network-architecture)
- [Virtualization Layer (Proxmox)](#virtualization-layer-proxmox)
- [Virtual Machines](#virtual-machines)
- [Remote Access (Tailscale)](#remote-access-tailscale)
- [Detection & Monitoring Stack](#detection--monitoring-stack)
- [Attack Simulations](#attack-simulations)
- [Lessons Learned](#lessons-learned)
- [Roadmap](#roadmap)

---

## Overview

| Property | Value |
|---|---|
| Purpose | Blue-team detection practice, offensive technique validation, network monitoring |
| Hypervisor | Proxmox VE |
| Perimeter / IDS | pfSense + Suricata |
| Network Security Monitoring | Security Onion |
| Offensive tooling | Kali Purple |
| Remote access | Tailscale (WireGuard-based mesh VPN) |

---

## Hardware

| Device | Role |
|---|---|
| UniFi Fiber Gateway (UXG-Fiber) | Edge router / gateway to ISP |
| Ubiquiti USW-Lite-8-PoE | 8-port PoE switch, distribution layer |
| Ubiquiti U7 (UX7) WAP | Wireless access point |
| Dell Micro PC | Proxmox virtualization host |

---

## Network Architecture

```
                    Internet
                       │
          ┌────────────────────────┐
          │  UniFi Fiber Gateway   │
          └────────────────────────┘
                       │
          ┌────────────────────────┐
          │  USW-Lite-8-PoE Switch │
          └────────────────────────┘
             │                   │
    ┌────────────────┐   ┌───────────────┐
    │  UX7 WAP       │   │  Dell Micro   │
    │  (wireless)    │   │  (Proxmox VE) │
    └────────────────┘   └───────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │        │          │          │                 │
   ┌──────────┐ ┌──────┐ ┌─────────┐ ┌──────────┐ ┌────────────────┐
   │ pfSense  │ │Kali  │ │ Ubuntu  │ │ Security │ │ (other VMs)    │
   │ Suricata │ │Purple│ │ Server  │ │  Onion   │ │                │
   └──────────┘ └──────┘ └─────────┘ └──────────┘ └────────────────┘
```

| Segment | VLAN ID | Subnet | Purpose |
|---|---|---|---|
| Management | VLAN 30 | /24 | Proxmox / UniFi management |
| Lab | VLAN 20 | /24 | Isolated VM lab network |
| Trusted | VLAN 10 | /24 | Home / personal devices |

---

## Virtualization Layer (Proxmox)

Proxmox VE runs on the Dell Micro PC and hosts all lab VMs.

- **Host:** Dell Micro PC
- **Hypervisor:** Proxmox VE
- **Networking:** Linux bridges mapped to the lab network; pfSense VM acts as the gateway/IDS for the isolated lab segment.

---

## Virtual Machines

| VM | OS / Role | Notes |
|---|---|---|
| pfSense | Firewall + Suricata IDS/IPS | Inline inspection for lab traffic |
| Security Onion | Network Security Monitoring | Zeek, Suricata, full packet capture, alerting |
| Kali Purple | Offensive + defensive toolkit | Attack origin for simulations |
| Ubuntu Server | Target / victim host | SSH service exposed as attack target |

### pfSense + Suricata
- Perimeter firewall for the lab segment.
- Suricata enabled on the relevant interface(s) for signature-based IDS/IPS.
- Document which rule sets are enabled (e.g. ET Open) and whether Suricata runs in IDS (alert) or IPS (block) mode.

### Security Onion
- Standalone deployment for network security monitoring.
- Ingests traffic for Zeek + Suricata analysis and full packet capture.
- Alerts reviewed in the Security Onion Console (SOC) / Kibana.
- Document how traffic reaches it (SPAN/mirror port, or a monitored bridge in Proxmox).

### Kali Purple
- Used as the attacker host to generate the traffic that the detection stack should catch.

### Ubuntu Server
- Acts as the victim host for SSH-based attack simulations.

---

## Remote Access (Tailscale)

Tailscale provides a WireGuard-based mesh VPN for remote administration of lab devices without exposing SSH or web UIs directly to the internet.

- **Purpose:** Remote SSH into VMs/hosts and access to web UIs (Proxmox, pfSense, Security Onion, UniFi) from outside the LAN.
- **Why it matters:** No inbound port-forwarding on the edge gateway means no SSH/web management ports exposed to the public internet — a meaningful reduction in attack surface.

### Hardening notes
Document how you've locked this down (these are strong defaults worth confirming):

- [ ] **ACLs:** Restrict which devices/users can reach which hosts and ports via Tailscale ACLs rather than allowing full mesh access.
- [ ] **MFA:** Enforce MFA on the identity provider used to log into Tailscale.
- [ ] **Key expiry:** Keep key expiry enabled on nodes.
- [ ] **Subnet routers / exit nodes:** Note whether any node advertises routes, and keep that scoped tightly.
- [ ] **No public exposure:** Confirm management web UIs and SSH are reachable *only* over Tailscale (or the local lab segment), not via gateway port-forwards.

> **Security note:** Because Tailscale can reach management interfaces, treat the identity account as a crown-jewel credential. MFA and ACLs are what keep remote access from becoming a liability.

---

## Detection & Monitoring Stack

| Tool | Layer | What it provides |
|---|---|---|
| Suricata (on pfSense) | Inline IDS/IPS | Signature-based detection/blocking at the perimeter |
| Security Onion | NSM | Zeek logs, Suricata alerts, PCAP, dashboards |
| Ubuntu auth logs | Host | `/var/log/auth.log` for authentication events |

Detection goal: every simulated attack below should produce corroborating evidence in **at least two** places (e.g. a Suricata alert *and* host auth logs), which mirrors how real detection engineering correlates signals.

---

## Attack Simulations

> ⚠️ All activity is performed **only** against hosts I own inside the isolated lab segment.

### 1. SSH Brute-Force Against Ubuntu Server

**Objective:** Generate failed-authentication noise and confirm it's detected on the network and on the host.

**Attacker:** Kali Purple
**Target:** Ubuntu Server (SSH)

**Example tooling (document the exact commands/wordlists you used):**
```bash
# Hydra example
hydra -l <user> -P <wordlist> ssh://<ubuntu-ip>

# or Medusa
medusa -h <ubuntu-ip> -u <user> -P <wordlist> -M ssh
```

**Expected detection evidence:**
- **Host:** repeated `Failed password` entries in `/var/log/auth.log`
  ```bash
  grep "Failed password" /var/log/auth.log | wc -l
  ```
- **Network:** Suricata alerts for SSH brute-force / scanning behavior
- **Security Onion:** Zeek `ssh.log` showing many connection attempts; corresponding Suricata alerts in the SOC

**What to capture for the repo:** screenshots of the auth log spike, the Suricata alert(s), and the Security Onion alert view.

---

### 2. SSH Exploitation

**Objective:** Attempt known SSH-related exploits/misconfigurations against the Ubuntu target and observe detection.

**Attacker:** Kali Purple
**Target:** Ubuntu Server

**Document for each attempt:**
- The specific technique or module used (e.g. an auth-related Metasploit module, user enumeration, or a misconfiguration you deliberately introduced)
- Whether it succeeded or failed, and why
- What the detection stack showed

**Expected detection evidence:**
- Suricata signature matches (note the SID/signature name)
- Zeek logs in Security Onion (`ssh.log`, `conn.log`)
- Host-side evidence in auth/syslog

**What to capture for the repo:** the command/module, the outcome, and the matching detections — side by side.

---

### Mapping to MITRE ATT&CK

| Simulation | Tactic | Technique |
|---|---|---|
| SSH brute-force | Credential Access | T1110 – Brute Force |
| SSH exploitation | Initial Access / Lateral Movement | T1021.004 – Remote Services: SSH |

> Filling in observed ATT&CK techniques is a great way to show detection-engineering thinking to anyone reviewing the repo.

---

## Lessons Learned

Document takeaways as you go — this section is often what hiring managers read most closely.

- What did each tool catch, and what did it miss?
- Did Suricata alert in IDS mode? Would IPS mode have blocked it?
- How quickly did the brute force surface in Security Onion?
- What tuning reduced false positives?

---

## Roadmap

- [ ] Add Sysmon + a Windows/Active Directory target for Windows-side telemetry
- [ ] Forward host logs (Ubuntu, Windows) into Security Onion or a separate SIEM (e.g. Wazuh)
- [ ] Write detection rules for the attacks above and validate them
- [ ] Add SOAR-style automation for alert triage
- [ ] Document VLAN segmentation and test isolation between lab and trusted networks
- [ ] Capture baseline PCAP vs. attack PCAP for comparison

---

*This lab is for educational and defensive research purposes. All testing is performed against systems I own.*
