# 🛡️ Secure Virtual Lab Deployment and Configuration

![Task ID](https://img.shields.io/badge/Task_ID-NSE--NETSEC--L--002--04-blue?style=for-the-badge)
![Track](https://img.shields.io/badge/Track-Junior_NetSec_Engineer_Internship_(Week_01)-informational?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-orange?style=for-the-badge)
![Duration](https://img.shields.io/badge/Duration-16_Hours-brightgreen?style=for-the-badge)
![CPE](https://img.shields.io/badge/CPE_Credits-4.0-purple?style=for-the-badge)

---

## 📋 1. Header & Regulatory Compliance Metadata

This repository documents the architectural design, deployment, hardening, and telemetry integration of a multi-segment virtual enterprise security environment. The implementation adheres to industry-standard security frameworks and defense-in-depth engineering principles.

| Framework / Standard | Control / Technique ID | Description & Implementation Alignment |
| :--- | :--- | :--- |
| **NIST SP 800-53** | `CM-6` (Configuration Settings)<br>`SI-2` (Flaw Remediation) | Enforced strict baseline firewall policies across perimeter and host layers (`CM-6`); executed systematic OS and signature patching across all nodes (`SI-2`). |
| **ISO/IEC 27001** | `A.12.6.1` (Vulnerability Mgmt)<br>`A.13.1.1` (Network Controls) | Mitigated technical vulnerabilities via baseline package updates (`A.12.6.1`); architected strict virtual network isolation separating trusted LAN assets from DMZ services (`A.13.1.1`). |
| **MITRE ATT&CK®** | `T1082` (System Info Discovery)<br>`T1562.001` (Disable or Modify Tools)<br>`T1200` (Hardware Additions) | Restricted lateral reconnaissance paths (`T1082`); enforced host-based AV/EDR and local firewall persistence (`T1562.001`); isolated virtual network switches to prevent unauthorized bridging (`T1200`). |
| **NICE Framework** | `PR-INF-001` | **Network Operations Specialist:** Configured, managed, and maintained network security appliances and virtualized routing infrastructures. |

---

## 🎯 2. Executive Summary & Lab Scope

### Overview
Modern enterprise networks require strict logical segmentation, host-level defense-in-depth, and centralized visibility to defend against lateral movement and unauthorized access. This project establishes a production-grade virtualized security laboratory utilizing **VMware Workstation / Oracle VirtualBox** and **pfSense** as the core perimeter and internal segmentation gateway.

### Core Objectives
* **Network Segmentation:** Design and enforce isolated network zones (`WAN`, `LAN`, and `DMZ`) to contain potential breaches and restrict lateral traffic flow.
* **Perimeter & Internal Firewalling:** Implement least-privilege stateful packet filtering rules on pfSense to govern inter-VLAN/subnet communication.
* **Endpoint Hardening:** Establish secure baselines across Linux (Ubuntu Server, Kali) and Windows 10 endpoints via automated patching, host-based firewalls (`UFW`), and anti-malware engines (`ClamAV`, `Windows Defender`).
* **Centralized Telemetry & SIEM:** Deploy **Wazuh / Security Onion** within the DMZ and configure real-time remote syslog ingestion from the pfSense gateway.
* **Hosted Lab Architecture Evaluation:** Contrast local hypervisor-based subnet isolation against cloud-hosted VPN namespace routing (HackTheBox).

---

## 🌐 3. Network Topology & Addressing Scheme

### Logical Architecture Diagram

```text
                       +-----------------------------+
                       |   External Network / Host   |
                       |     (Internet / Upstream)   |
                       +--------------+--------------+
                                      |
                                      | WAN Interface (DHCP / NAT / Bridged)
                                      v
                    +-----------------+-----------------+
                    |      pfSense Security Gateway     |
                    |      (Stateful Firewall & Router) |
                    +--------+-----------------+--------+
                             |                 |
        LAN Interface        |                 |        DMZ Interface
        192.168.10.1/24      |                 |        192.168.20.1/24
                             |                 |
              +--------------+                 +--------------+
              |                                               |
              v                                               v
+-------------+---------------+                 +-------------+---------------+
|      vSwitch1: LAN Zone     |                 |      vSwitch2: DMZ Zone     |
|    Subnet: 192.168.10.0/24  |                 |    Subnet: 192.168.20.0/24  |
+-----+-----------------+-----+                 +-------------+---------------+
      |                 |                                     |
      |                 |                                     | (HTTP/443 & Syslog/514 Only)
      v                 v                                     v
+-----+------+   +------+-----+                        +------+---------------+
| Kali Linux |   | Windows 10 |                        |  Ubuntu Server LTS   |
| Admin/Test |   | Client End |                        |  Web Server & SIEM   |
| .10.10/24  |   | .10.20/24  |                        |  192.168.20.10/24    |
+------------+   +------------+                        +----------------------+
```

### IP Addressing & Asset Inventory

| Hostname / Asset | Network Zone | IP Address / CIDR | Default Gateway | Role & Security Function |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense-GW** | WAN / LAN / DMZ | `DHCP` (WAN)<br>`192.168.10.1/24` (LAN)<br>`192.168.20.1/24` (DMZ) | Upstream Router | Perimeter Firewall, Inter-Subnet Router, Syslog Forwarder |
| **Kali-SecAdmin** | LAN (Trusted) | `192.168.10.10/24` | `192.168.10.1` | Security Administration, Vulnerability Scanning, Penetration Testing |
| **Win10-Client** | LAN (Trusted) | `192.168.10.20/24` | `192.168.10.1` | Standard Enterprise Workstation Baseline, Endpoint Security Validation |
| **Ubuntu-DMZ-Srv** | DMZ (Semi-Trusted)| `192.168.20.10/24` | `192.168.20.1` | Public/Internal Web Server (`80`/`443`), Wazuh / Security Onion SIEM Host |

---

## ⚙️ 4. Step-by-Step Implementation Guide

### 4.1 Hypervisor Network Provisioning
1. Provisioned three distinct virtual network adapters within the hypervisor (`VMware Workstation` / `VirtualBox`):
   * **Adapter 1 (WAN):** Configured as `NAT` / `Bridged` to provide upstream internet connectivity to the pfSense appliance.
   * **Adapter 2 (LAN):** Assigned to an isolated Host-Only/Internal virtual switch (`192.168.10.0/24`) with hypervisor DHCP disabled.
   * **Adapter 3 (DMZ):** Assigned to a separate isolated Host-Only/Internal virtual switch (`192.168.20.0/24`) with hypervisor DHCP disabled.

### 4.2 pfSense Gateway Configuration & Firewall Policy
1. Installed pfSense and mapped interfaces (`em0` -> `WAN`, `em1` -> `LAN`, `em2` -> `OPT1/DMZ`).
2. Configured static IPv4 addressing for `LAN` (`192.168.10.1/24`) and `DMZ` (`192.168.20.1/24`).
3. Enforced the following stateful firewall rule matrix under **Firewall > Rules**:

| Interface | Action | Protocol | Source | Destination | Port(s) | Policy Justification |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **WAN** | `BLOCK` | `ANY` | `ANY` | `WAN Address / Subnets` | `ANY` | Drop all unsolicited inbound internet traffic by default (Implicit/Explicit Deny). |
| **LAN** | `PASS` | `TCP` | `LAN Net` | `192.168.20.10` | `80, 443` | Strictly permit HTTP and HTTPS traffic from internal hosts to the DMZ Web Server. |
| **LAN** | `BLOCK` | `TCP/UDP` | `LAN Net` | `DMZ Net` | `22, 445, 3389, ANY` | Explicitly drop SSH, SMB, RDP, and all other non-web lateral protocols to the DMZ. |
| **DMZ** | `BLOCK` | `ANY` | `DMZ Net` | `LAN Net` | `ANY` | Block all DMZ-initiated connections to the LAN to prevent lateral movement if compromised. |
| **DMZ** | `PASS` | `TCP/UDP` | `DMZ Net` | `!LAN Net (ANY)` | `80, 443, 53, 123` | Allow DMZ server outbound access to WAN for OS/SIEM package updates and DNS/NTP. |

### 4.3 Ubuntu Server LTS Hardening (DMZ Host)
Executed baseline system updates, configured host-based firewall rules (`UFW`) to mirror perimeter controls, and deployed open-source anti-malware scanning (`ClamAV`).

```bash
# 1. Update package repositories and upgrade installed packages (NIST SI-2)
sudo apt update && sudo apt upgrade -y

# 2. Install and configure Uncomplicated Firewall (UFW) (NIST CM-6)
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp comment 'Allow HTTP Web Traffic'
sudo ufw allow 443/tcp comment 'Allow HTTPS Web Traffic'
sudo ufw allow from 192.168.20.1 to any port 514 proto udp comment 'Allow pfSense Syslog'
sudo ufw enable
sudo ufw status verbose

# 3. Install, update, and start ClamAV Antivirus Daemon
sudo apt install clamav clamav-daemon -y
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable --now clamav-daemon
sudo systemctl status clamav-daemon
```

### 4.4 Kali Linux Configuration (LAN Admin Host)
Updated the security administration workstation to ensure vulnerability scanning tools, exploit databases, and core libraries are current.

```bash
# Synchronize repositories and perform full distribution upgrade
sudo apt update && sudo apt upgrade -y
```

### 4.5 Windows 10 Workstation Hardening (LAN Client)
1. **Patch Management:** Navigated to **Settings > Update & Security > Windows Update** and applied all pending cumulative security updates, servicing stack updates, and `.NET` patches.
2. **Endpoint Protection:** Configured **Windows Security > Virus & threat protection**:
   * Verified **Real-Time Protection** is set to `On`.
   * Enabled **Cloud-delivered protection** and **Automatic sample submission** to defend against zero-day threats and prevent defense impairment (`MITRE T1562.001`).

```powershell
# PowerShell verification of Windows Defender status and signature versions
Get-MpComputerStatus | Select-Object AMServiceEnabled, RealTimeProtectionEnabled, IoavProtectionEnabled, AntivirusSignatureLastUpdated
```

---

## 📊 5. Monitoring, SIEM Integration & Hosted Lab Evaluation

### 5.1 Wazuh / Security Onion Deployment & pfSense Syslog Forwarding
To establish centralized visibility across network boundaries, **Wazuh / Security Onion** was deployed on the Ubuntu DMZ instance (`192.168.20.10`), ingesting real-time telemetry from the pfSense perimeter gateway.

#### Step 1: SIEM Listener Configuration (Ubuntu Server)
Configured the SIEM manager (`ossec.conf` / syslog input container) to listen for incoming remote syslog streams from the pfSense gateway over UDP/514:

```xml
<!-- Wazuh / OSSEC Remote Syslog Configuration (/var/ossec/etc/ossec.conf) -->
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.20.1/32</allowed-ips>
  <local_ip>192.168.20.10</local_ip>
</remote>
```

#### Step 2: pfSense Remote Logging Configuration
1. Logged into the pfSense WebGUI and navigated to **Status > System Logs > Settings**.
2. Checked **Enable Remote Logging**.
3. Configured **Source Address** as `DMZ (192.168.20.1)`.
4. Entered `192.168.20.10:514` under **Remote Log Servers**.
5. Checked **Firewall Events**, **System Events**, and **DHCP/Portal Auth** under Remote Syslog Contents and saved the configuration.
6. **Validation:** Generated test blocked traffic (`SSH` and `SMB` connection attempts from `192.168.10.10` to `192.168.20.10`) and verified real-time alert generation within the SIEM dashboard.

---

### 5.2 Architectural Comparison: Local Virtual Lab vs. Hosted VPN Labs (HackTheBox)
Completed the **HackTheBox "Setting Up"** module by establishing an OpenVPN tunnel (`tun0`) from the Kali Linux host and contrasting the networking mechanics against the local hypervisor deployment.

| Architectural Dimension | Local Virtual Lab (pfSense + Hypervisor) | Hosted Security Lab (HackTheBox VPN) |
| :--- | :--- | :--- |
| **Isolation Mechanism** | Layer 2 Virtual Switches (`vSwitch` / Host-Only adapters) and Layer 3/4 stateful firewall interfaces. | Point-to-Point encrypted VPN tunnel (`OpenVPN` / `WireGuard`) utilizing virtual `tun0` interfaces and cloud network namespaces. |
| **Routing & Visibility** | Full administrative control over routing tables, NAT, packet captures, and firewall rule sets on the gateway. | Split-tunnel routing; only traffic destined for lab subnets (`10.10.10.0/24` / `10.129.x.x`) traverses `tun0`. Underlying infrastructure is abstracted. |
| **Defensive Telemetry** | Complete Blue Team visibility (SIEM ingestion, host logs, network taps, firewall block logs). | Strictly offensive/client-side visibility; players cannot inspect target-side SIEM pipelines or hypervisor switches. |
| **Resource Overhead** | High local hardware consumption (CPU, RAM, Disk I/O required for running multiple concurrent VMs). | Minimal local footprint; compute and target hosting are offloaded to cloud infrastructure. |

---

## 📁 6. Evidence & Deliverables Checklist

All required verification artifacts, configuration proofs, and analytical documentation have been compiled and placed in the repository's deliverables directory:

| Deliverable File | Format | Scope & Verification Contents | Status |
| :--- | :---: | :--- | :---: |
| 📄 [`LabSetup.pdf`](./deliverables/LabSetup.pdf) | `PDF` | Logical & physical network topology diagram, IP allocation scheme, and hypervisor VM inventory screenshots showing active nodes (`pfSense`, `Kali`, `Win10`, `Ubuntu`). | ✅ Completed |
| 📄 [`Baseline.pdf`](./deliverables/Baseline.pdf) | `PDF` | Terminal and GUI evidence of OS patch cycles (`apt upgrade` / Windows Update), active `UFW` rule sets, `ClamAV` (`freshclam`) signature updates, and Windows Defender Real-Time Protection status. | ✅ Completed |
| 📄 [`SIEM.pdf`](./deliverables/SIEM.pdf) | `PDF` | pfSense remote syslog forwarding configuration and Wazuh / Security Onion dashboard screenshots demonstrating active log ingestion and firewall block alerts. | ✅ Completed |
| 🖼️ [`HTB.png`](./deliverables/HTB.png) | `PNG` | Screenshot proof of validated HackTheBox OpenVPN connection (`tun0` IP assignment) and successful platform connectivity verification. | ✅ Completed |
| 📘 [`SecureLabReport.pdf`](./deliverables/SecureLabReport.pdf) | `PDF` | Comprehensive final technical engineering report compiled using the official internship report template, detailing methodology, testing results, and risk analysis. | ✅ Completed |

---

## 💡 7. Reflection & Case Study Takeaways

Designing and hardening this virtual environment highlights how foundational network architecture and hygiene controls directly prevent catastrophic enterprise breaches.

### Case Study 1: The Target Data Breach (2013) — *The Necessity of DMZ Isolation & Micro-Segmentation*
* **Incident Context:** Attackers compromised third-party HVAC vendor credentials to gain an initial foothold in Target's network. Because the vendor portal and internal corporate networks lacked strict segmentation from the payment card environment, adversaries moved laterally to Point-of-Sale (PoS) terminals, exfiltrating 40 million credit card records.
* **Lab Engineering Alignment:** In this deployment, the **DMZ (`192.168.20.0/24`)** is strictly isolated from the internal **LAN (`192.168.10.0/24`)**. Even if an external threat actor compromises the public-facing Ubuntu Web Server via a web application vulnerability, pfSense firewall policies explicitly **drop all DMZ-initiated connections to the LAN**. This architectural boundary neutralizes lateral movement toward internal workstations (`Win10-Client`) and administrative hosts.

### Case Study 2: WannaCry Ransomware Epidemic (2017) — *Patch Management & Lateral Protocol Restriction*
* **Incident Context:** WannaCry leveraged the `EternalBlue` exploit (`MS17-010`) targeting unpatched Windows Server Message Block (`SMBv1`) services over TCP port `445`, rapidly encrypting hundreds of thousands of systems globally across flat networks.
* **Lab Engineering Alignment:** This lab demonstrates a multi-layered defense against wormable ransomware propagation:
  1. **Flaw Remediation (`NIST SI-2`):** Proactive patching across Windows 10, Ubuntu, and Kali eliminates known kernel and service vulnerabilities before exploitation can occur.
  2. **Protocol Restriction (`ISO A.13.1.1`):** By restricting LAN-to-DMZ traffic strictly to `TCP/80` and `TCP/443` while blocking administrative and file-sharing protocols (`SMB/445`, `RDP/3389`, `SSH/22`) at both the pfSense perimeter and host `UFW` layers, lateral propagation vectors are completely severed.
  3. **Endpoint & SIEM Telemetry:** Real-time Windows Defender Cloud Protection paired with centralized Wazuh/Security Onion syslog monitoring ensures that anomalous port scanning or exploit attempts trigger immediate high-severity alerts.

---
*Authored and maintained as part of the **Junior Network Security Engineer Internship (Track Week 01 — Task ID: NSE-NETSEC-L-002-04)**.*
