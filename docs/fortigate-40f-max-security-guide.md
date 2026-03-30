# FortiGate 40F – Maximum Security Configuration Guide with VLAN Segmentation

## Table of Contents

1. [Introduction](#1-introduction)
2. [Required Devices & Hardware](#2-required-devices--hardware)
3. [Network Topology & VLAN Design](#3-network-topology--vlan-design)
4. [Initial FortiGate 40F Setup](#4-initial-fortigate-40f-setup)
5. [VLAN Segmentation Configuration](#5-vlan-segmentation-configuration)
6. [Firewall Policies](#6-firewall-policies)
7. [Security Profiles](#7-security-profiles)
8. [SSL/TLS Inspection](#8-ssltls-inspection)
9. [VPN Configuration](#9-vpn-configuration)
10. [Advanced Security Features](#10-advanced-security-features)
11. [Logging & Monitoring](#11-logging--monitoring)
12. [Hardening Checklist](#12-hardening-checklist)

---

## 1. Introduction

The **Fortinet FortiGate 40F** is a compact next-generation firewall (NGFW) designed for small to medium-sized businesses. It delivers enterprise-grade security features including:

- Intrusion Prevention System (IPS)
- Application Control
- Web Filtering
- Anti-Malware / Anti-Virus
- SSL Deep Inspection
- DNS Filtering
- SD-WAN
- IPsec & SSL VPN

This guide walks you through configuring the FortiGate 40F for **maximum security** using **VLAN segmentation** to isolate network traffic between different zones (Trusted, IoT, Guest, DMZ, Management).

> **FortiOS Version**: This guide targets **FortiOS 7.4.x**. Minor differences may exist in earlier versions.

---

## 2. Required Devices & Hardware

### Mandatory

| Device | Purpose |
|---|---|
| **Fortinet FortiGate 40F** | Core firewall / NGFW |
| **Managed Layer-2/3 Switch** (e.g., FortiSwitch 108F, Cisco SG350, UniFi USW-24) | VLAN-aware switching for internal segments |
| **Modem / ONT** (provided by ISP) | WAN uplink – connects to FortiGate WAN1 port |
| **UPS (Uninterruptible Power Supply)** | Protects against power outages; prevents config corruption |
| **Ethernet cables (Cat6 or higher)** | Wired connections between devices |

### Recommended

| Device | Purpose |
|---|---|
| **FortiAP** (e.g., FortiAP 231G) or another WPA3-capable access point | Secure Wi-Fi with VLAN-based SSID separation |
| **FortiAnalyzer** (physical or VM) | Centralised log management, reporting, and incident response |
| **FortiManager** (physical or VM) | Centralised policy and configuration management |
| **FortiToken / FortiAuthenticator** | Multi-factor authentication (MFA) for admin and VPN users |
| **Network TAP or SPAN-capable switch** | Passive traffic mirroring for network monitoring |
| **Dedicated management laptop/PC** | Out-of-band management on an isolated Management VLAN |

### Optional / Enhanced Security

| Device | Purpose |
|---|---|
| **FortiSandbox** | Dynamic malware analysis of suspicious files |
| **FortiNAC** | Network Access Control – posture checking before LAN access |
| **FortiClient EMS** | Endpoint management for FortiClient VPN / ZTNA |
| **Hardware Security Module (HSM)** | Secure certificate and key storage |

---

## 3. Network Topology & VLAN Design

### Physical Port Layout (FortiGate 40F)

```
Internet (ISP)
      │
  [WAN1 port]
  FortiGate 40F
  [LAN ports 1–4 / Internal switch] ──── Managed Switch
                                              │
                          ┌───────────────────┼──────────────────────┐
                     VLAN 10              VLAN 20                VLAN 30
                   (Trusted LAN)         (IoT)                  (Guest Wi-Fi)
                          │                   │                      │
                    PCs / Laptops      Smart TVs / Cameras      Guest Devices
                    
                    VLAN 40             VLAN 50
                    (DMZ)              (Management)
                       │                   │
               Web/Mail Servers     FortiGate GUI / SSH
```

### VLAN Plan

| VLAN ID | Name | Subnet | Default Gateway | Purpose |
|---|---|---|---|---|
| 10 | Trusted | `192.168.10.0/24` | `192.168.10.1` | Corporate PCs, laptops, printers |
| 20 | IoT | `192.168.20.0/24` | `192.168.20.1` | Smart devices, cameras, sensors |
| 30 | Guest | `192.168.30.0/24` | `192.168.30.1` | Visitor Wi-Fi, BYOD |
| 40 | DMZ | `192.168.40.0/24` | `192.168.40.1` | Publicly accessible servers |
| 50 | Management | `192.168.50.0/24` | `192.168.50.1` | Network devices, firewall management |

### Inter-VLAN Communication Policy (default – least privilege)

| Source VLAN | Destination | Allowed |
|---|---|---|
| Trusted (10) | Internet | ✅ Yes (with UTM inspection) |
| Trusted (10) | DMZ (40) | ✅ Yes (controlled) |
| Trusted (10) | IoT (20), Guest (30) | ❌ No |
| IoT (20) | Internet | ✅ Yes (restricted ports only) |
| IoT (20) | All other VLANs | ❌ No |
| Guest (30) | Internet | ✅ Yes (HTTP/HTTPS only) |
| Guest (30) | All other VLANs | ❌ No |
| DMZ (40) | Internet | ✅ Yes (outbound) |
| DMZ (40) | Trusted/IoT/Guest | ❌ No |
| Management (50) | All VLANs | ✅ Yes (admin only) |
| Any | Management (50) | ❌ No |

---

## 4. Initial FortiGate 40F Setup

### Step 4.1 – Physical Connection

1. Connect the **ISP modem/ONT** to the **WAN1** port of the FortiGate 40F.
2. Connect your **management PC** directly to **LAN port 1** using an Ethernet cable.
3. Power on the FortiGate 40F.

### Step 4.2 – Initial GUI Access

1. Set your management PC's IP address to **`192.168.1.x`** (default FortiGate LAN IP is `192.168.1.99`).
2. Open a browser and navigate to **`https://192.168.1.99`**.
3. Accept the self-signed certificate warning.
4. Log in with the default credentials:
   - **Username**: `admin`
   - **Password**: *(blank – press Enter)*
5. You will be prompted to set a new password immediately. Use a strong password (16+ characters, uppercase, lowercase, numbers, symbols).

### Step 4.3 – Run the Setup Wizard

Navigate to **Dashboard → Setup Wizard** and complete:

1. **Hostname**: Set a descriptive hostname (e.g., `FGT40F-HQ`).
2. **Time zone**: Set the correct timezone and enable NTP (use Fortinet NTP servers or `pool.ntp.org`).
3. **WAN Interface**: Configure WAN1 as DHCP (or Static/PPPoE as required by your ISP).
4. **DNS**: Set primary DNS to `9.9.9.9` (Quad9) and secondary to `1.1.1.1` (Cloudflare) for privacy-respecting DNS.
5. Complete the wizard and apply settings.

### Step 4.4 – Firmware Update

1. Go to **System → Firmware & Registration**.
2. Check for the latest stable FortiOS release.
3. Download and install the latest firmware.
4. Reboot when prompted.
5. Re-login after reboot.

### Step 4.5 – Secure Management Access

Go to **System → Settings**:

```
Admin Settings:
  ✅ Enable HTTPS on port 443 (or custom port, e.g., 8443)
  ✅ Disable HTTP (redirect to HTTPS)
  ✅ Enable SSH (restrict to Management VLAN only)
  ✅ Disable Telnet
  ✅ Disable SNMP v1/v2 (use SNMPv3 if required)

Idle Timeout:
  Set to 5 minutes

Trusted Hosts (for admin accounts):
  Restrict admin login to Management VLAN subnet: 192.168.50.0/24
```

### Step 4.6 – Create Admin Accounts with Least Privilege

1. Go to **System → Administrators**.
2. Delete or disable the default `admin` account after creating a named account.
3. Create a new super-admin account:
   - **Username**: `<your-name>-admin`
   - **Password**: Strong, unique password
   - **Trusted Hosts**: `192.168.50.0/24`
   - **Two-factor Authentication**: Enable with FortiToken
4. Create read-only accounts for monitoring staff:
   - **Profile**: `prof_admin` → set to Read-only

---

## 5. VLAN Segmentation Configuration

### Step 5.1 – Create VLAN Interfaces on FortiGate

Navigate to **Network → Interfaces → Create New → Interface**:

#### VLAN 10 – Trusted

```
Name:           VLAN10_Trusted
Type:           VLAN
Interface:      internal (hardware switch / lan)
VLAN ID:        10
Role:           LAN
IP/Netmask:     192.168.10.1 / 255.255.255.0
Administrative Access:
  ✅ HTTPS    ✅ SSH    ✅ PING
DHCP Server:    Enable
  Start IP:   192.168.10.100
  End IP:     192.168.10.200
  DNS Server: Same as System
  Lease Time: 8 hours
```

#### VLAN 20 – IoT

```
Name:           VLAN20_IoT
Type:           VLAN
Interface:      internal
VLAN ID:        20
Role:           LAN
IP/Netmask:     192.168.20.1 / 255.255.255.0
Administrative Access:
  ✅ PING only (disable HTTPS, SSH)
DHCP Server:    Enable
  Start IP:   192.168.20.100
  End IP:     192.168.20.200
  DNS Server: 9.9.9.9 (Quad9 – blocks malicious domains)
```

#### VLAN 30 – Guest

```
Name:           VLAN30_Guest
Type:           VLAN
Interface:      internal
VLAN ID:        30
Role:           LAN
IP/Netmask:     192.168.30.1 / 255.255.255.0
Administrative Access:
  ❌ None (no management access)
DHCP Server:    Enable
  Start IP:   192.168.30.100
  End IP:     192.168.30.200
  DNS Server: 1.1.1.3 (Cloudflare – blocks malware + adult content)
  Lease Time: 2 hours
```

#### VLAN 40 – DMZ

```
Name:           VLAN40_DMZ
Type:           VLAN
Interface:      internal
VLAN ID:        40
Role:           DMZ
IP/Netmask:     192.168.40.1 / 255.255.255.0
Administrative Access:
  ❌ None
DHCP Server:    Enable (or use static IPs for servers)
```

#### VLAN 50 – Management

```
Name:           VLAN50_Mgmt
Type:           VLAN
Interface:      internal
VLAN ID:        50
Role:           LAN
IP/Netmask:     192.168.50.1 / 255.255.255.0
Administrative Access:
  ✅ HTTPS    ✅ SSH    ✅ PING
DHCP Server:    Enable (or assign static IPs)
  Start IP:   192.168.50.10
  End IP:     192.168.50.50
```

> **CLI Alternative** – You can configure VLAN interfaces via FortiGate CLI:
>
> ```bash
> config system interface
>     edit "VLAN10_Trusted"
>         set vdom "root"
>         set ip 192.168.10.1 255.255.255.0
>         set allowaccess https ssh ping
>         set type vlan
>         set vlanid 10
>         set interface "internal"
>     next
>     edit "VLAN20_IoT"
>         set vdom "root"
>         set ip 192.168.20.1 255.255.255.0
>         set allowaccess ping
>         set type vlan
>         set vlanid 20
>         set interface "internal"
>     next
>     edit "VLAN30_Guest"
>         set vdom "root"
>         set ip 192.168.30.1 255.255.255.0
>         set allowaccess none
>         set type vlan
>         set vlanid 30
>         set interface "internal"
>     next
>     edit "VLAN40_DMZ"
>         set vdom "root"
>         set ip 192.168.40.1 255.255.255.0
>         set allowaccess none
>         set type vlan
>         set vlanid 40
>         set interface "internal"
>     next
>     edit "VLAN50_Mgmt"
>         set vdom "root"
>         set ip 192.168.50.1 255.255.255.0
>         set allowaccess https ssh ping
>         set type vlan
>         set vlanid 50
>         set interface "internal"
>     next
> end
> ```

### Step 5.2 – Configure the Managed Switch

On your managed switch (example uses generic 802.1Q VLAN syntax):

1. **Create VLANs 10, 20, 30, 40, 50** in the switch VLAN database.
2. **Uplink port** (connected to FortiGate LAN port): Set as **trunk/tagged** for all VLANs (10, 20, 30, 40, 50).
3. **Access ports** for end devices: Set as **access/untagged** in the appropriate VLAN:
   - Server room ports → VLAN 40
   - Admin workstations → VLAN 10
   - Network equipment management → VLAN 50
4. **Access point port** (if using a managed AP for VLAN-based SSIDs): Set as trunk for VLANs 10, 30.

### Step 5.3 – Configure Wireless SSIDs (if using FortiAP)

Navigate to **WiFi & Switch Controller → SSIDs → Create New**:

#### Corporate SSID (Trusted VLAN)

```
SSID:             Corp-Secure
Security:         WPA3-Enterprise (or WPA3-Personal with a strong PSK)
Traffic Mode:     Tunnel to FortiGate
VLAN:             VLAN10_Trusted
Broadcast SSID:   Disabled (hidden – optional for extra obfuscation)
```

#### Guest SSID

```
SSID:             Guest-WiFi
Security:         WPA3-Personal (strong passphrase, changed regularly)
Traffic Mode:     Tunnel to FortiGate
VLAN:             VLAN30_Guest
Client Isolation: Enabled
Broadcast SSID:   Enabled
```

---

## 6. Firewall Policies

Firewall policies are the core of traffic control. Navigate to **Policy & Objects → Firewall Policy → Create New**.

> **Important**: Policies are evaluated **top-down**. Place more specific policies above general ones. End with an **explicit deny-all** policy.

### Step 6.1 – Trusted LAN → Internet (with UTM)

```
Name:               Trusted-to-Internet
Incoming Interface: VLAN10_Trusted
Outgoing Interface: WAN1
Source:             all
Destination:        all
Schedule:           always
Service:            ALL
Action:             ACCEPT

Security Profiles:
  ✅ AntiVirus:     default (or custom – see Section 7)
  ✅ Web Filter:    default (or custom)
  ✅ App Control:   default (or custom)
  ✅ IPS:           default
  ✅ SSL Inspection: deep-inspection
  ✅ DNS Filter:    default

NAT:                Enable (outbound masquerade)
Log Allowed Traffic: All Sessions
```

### Step 6.2 – IoT → Internet (Restricted)

```
Name:               IoT-to-Internet
Incoming Interface: VLAN20_IoT
Outgoing Interface: WAN1
Source:             all
Destination:        all
Schedule:           always
Service:            HTTP, HTTPS, DNS, NTP
Action:             ACCEPT

Security Profiles:
  ✅ AntiVirus
  ✅ IPS
  ✅ DNS Filter (with IoT botnet blocking)
  ✅ App Control (block P2P, remote access tools)

NAT:                Enable
Log Allowed Traffic: All Sessions
```

### Step 6.3 – Guest → Internet (Web Only)

```
Name:               Guest-to-Internet
Incoming Interface: VLAN30_Guest
Outgoing Interface: WAN1
Source:             all
Destination:        all
Schedule:           always
Service:            HTTP, HTTPS, DNS
Action:             ACCEPT

Security Profiles:
  ✅ AntiVirus
  ✅ Web Filter (Guest profile – block P2P, illegal content, social media optional)
  ✅ IPS
  ✅ App Control
  ✅ SSL Inspection: certificate-inspection

NAT:                Enable
Log Allowed Traffic: All Sessions
```

### Step 6.4 – Trusted → DMZ (Controlled Access)

```
Name:               Trusted-to-DMZ
Incoming Interface: VLAN10_Trusted
Outgoing Interface: VLAN40_DMZ
Source:             [Trusted Admin Address Group]
Destination:        [DMZ Server Address Group]
Schedule:           always
Service:            HTTPS, SSH, RDP (only required services)
Action:             ACCEPT
Log:                All Sessions
```

### Step 6.5 – DMZ → Internet (Outbound)

```
Name:               DMZ-to-Internet
Incoming Interface: VLAN40_DMZ
Outgoing Interface: WAN1
Source:             all
Destination:        all
Schedule:           always
Service:            HTTP, HTTPS, DNS, SMTP (as needed)
Action:             ACCEPT

Security Profiles:
  ✅ AntiVirus
  ✅ IPS
  ✅ Web Filter
NAT:                Enable
```

### Step 6.6 – Internet → DMZ (Inbound / VIP)

For publicly accessible servers, use **Virtual IPs (VIPs)**:

1. Go to **Policy & Objects → Virtual IPs → Create New**:
   ```
   Name:             VIP-WebServer
   External Interface: WAN1
   External IP:      <your-WAN-IP>
   Mapped IP:        192.168.40.10
   Port Forwarding:  Enable
     External Port:  443
     Map to Port:    443
   ```
2. Create inbound policy:
   ```
   Name:               Internet-to-DMZ-Web
   Incoming Interface: WAN1
   Outgoing Interface: VLAN40_DMZ
   Source:             all
   Destination:        VIP-WebServer
   Service:            HTTPS
   Action:             ACCEPT
   Security Profiles:
     ✅ AntiVirus
     ✅ IPS (enable WAF signatures)
     ✅ Web Application Firewall (WAF)
   ```

### Step 6.7 – Management → All VLANs

```
Name:               Mgmt-to-All
Incoming Interface: VLAN50_Mgmt
Outgoing Interface: [all internal VLANs]
Source:             [Management Hosts Address Group]
Destination:        all
Service:            ALL
Action:             ACCEPT
Log:                All Sessions
```

### Step 6.8 – Explicit Deny-All (Bottom of Policy List)

```
Name:               Deny-All
Incoming Interface: any
Outgoing Interface: any
Source:             all
Destination:        all
Service:            ALL
Action:             DENY
Log Violation Traffic: Enable
```

---

## 7. Security Profiles

### Step 7.1 – AntiVirus Profile

Navigate to **Security Profiles → AntiVirus → Create New**:

```
Name:               AV-Maximum
Scan Mode:          Full Scan (or Flow-based for performance)
Inspection Mode:    Proxy-based (recommended for maximum detection)

Protocols to scan:
  ✅ HTTP    ✅ HTTPS (requires SSL inspection)
  ✅ IMAP    ✅ POP3    ✅ SMTP
  ✅ FTP     ✅ SMB

Advanced Options:
  ✅ Use FortiSandbox cloud (submit unknown files)
  ✅ Detect and block grayware
  ✅ Enable Content Disarm & Reconstruction (CDR) for Office/PDF files
  ✅ Quarantine infected files
```

### Step 7.2 – Web Filter Profile

Navigate to **Security Profiles → Web Filter → Create New**:

```
Name:               WebFilter-Strict

FortiGuard Categories:
  Block:  Security Risk, Malicious Sites, Phishing, Spam, Proxy/Anonymizer,
          P2P, Illegal/Questionable content
  Monitor: Social Media (or block based on policy)
  Allow: Business, News, Education, Technology

URL Filter:
  ✅ Enable custom block list for known bad domains

Advanced:
  ✅ Enable Safe Search enforcement (Google, Bing, YouTube)
  ✅ Block override (prevent users bypassing)
  ✅ Log all URL filtering
```

### Step 7.3 – IPS Profile

Navigate to **Security Profiles → Intrusion Prevention → Create New**:

```
Name:               IPS-Maximum

IPS Signatures:
  ✅ Enable all critical and high severity signatures – Action: Block
  ✅ Enable medium severity signatures – Action: Block
  ✅ Enable low severity – Action: Monitor (or Block)
  ✅ Botnet C&C detection – Action: Block
  ✅ Protocol anomaly detection – Action: Block

Rate-Based Signatures:
  ✅ Enable protection against DoS/DDoS
```

### Step 7.4 – Application Control Profile

Navigate to **Security Profiles → Application Control → Create New**:

```
Name:               AppCtrl-Strict

Categories:
  Block:   Botnet, P2P, Remote Access (unless business-approved),
           Proxy, Cryptocurrency mining
  Monitor: Social Media, Video Streaming
  Allow:   Business Applications, Collaboration tools

Unknown Applications:
  Action: Monitor (or Block in high-security environments)
Log:     All Application Sessions
```

### Step 7.5 – DNS Filter Profile

Navigate to **Security Profiles → DNS Filter → Create New**:

```
Name:               DNS-Maximum

FortiGuard DNS Categories:
  Block: Malicious Websites, Phishing, Spam, Botnets, Command & Control

DNS Translation:
  ✅ Redirect blocked DNS queries to FortiGuard block page

External IP Block List:
  ✅ Enable Botnet C&C domain blocking
```

---

## 8. SSL/TLS Inspection

SSL deep inspection decrypts and inspects HTTPS traffic. This is critical for detecting threats hidden in encrypted traffic.

### Step 8.1 – Create a Deep Inspection Profile

Navigate to **Security Profiles → SSL/SSH Inspection → Create New**:

```
Name:               Deep-Inspection-Custom
Inspection Method:  SSL Certificate Inspection (for initial rollout)
                    OR Full SSL Inspection (maximum security – requires CA distribution)

CA Certificate:     Use Fortinet_CA_SSL (built-in) OR import your organisation's CA

Inspect:
  ✅ HTTPS (port 443)
  ✅ SMTPS (port 465)
  ✅ POP3S (port 995)
  ✅ IMAPS (port 993)
  ✅ FTPS

Exceptions (SSL Exemptions):
  Add trusted financial, healthcare domains that use certificate pinning
  (e.g., banking apps, government sites)

Minimum SSL/TLS Version:
  ✅ TLS 1.2 (block TLS 1.0 and 1.1)
```

### Step 8.2 – Distribute the CA Certificate to Clients

For full SSL inspection, clients must trust the FortiGate CA:

1. Export the FortiGate CA certificate: **System → Certificates → CA Certificates → Fortinet_CA_SSL → Download**.
2. Deploy to endpoints via:
   - **Windows**: Group Policy (GPO) → Computer Configuration → Windows Settings → Public Key Policies → Trusted Root CAs
   - **macOS**: System Preferences → Profiles, or via MDM
   - **iOS/Android**: Via MDM profile

---

## 9. VPN Configuration

### Step 9.1 – IPsec Site-to-Site VPN (for branch offices)

Navigate to **VPN → IPsec Wizard**:

```
Template:     Site to Site
Remote Device: Fortinet (or Custom for non-Fortinet peers)

Phase 1:
  Authentication: Pre-Shared Key (use 32+ character random key)
                  OR Certificate-based (recommended for maximum security)
  IKE Version:    IKEv2
  Encryption:     AES-256-GCM
  Authentication: SHA-256 or SHA-384
  DH Group:       19 (256-bit ECC) or 20 (384-bit ECC)

Phase 2:
  Encryption:     AES-256-GCM
  Authentication: SHA-256
  PFS:            Enable (Group 19 or 20)
  Key Lifetime:   3600 seconds
```

### Step 9.2 – SSL VPN for Remote Users

Navigate to **VPN → SSL-VPN Settings**:

```
Listen on Interface: WAN1
Listen on Port:      443 (or custom port, e.g., 10443)
Restrict Access:     Limit to specific source IPs if possible

Authentication:
  ✅ Require Two-Factor Authentication (FortiToken / TOTP)
  ✅ Require client certificate (for maximum security)

SSL/TLS Settings:
  Min TLS Version: TLS 1.2
  Cipher Suites:   High security (AES-256-GCM preferred)
```

Create SSL VPN Portal:

```
Name:             full-access
Tunnel Mode:      Enable
Split Tunnelling: Disable (force ALL traffic through VPN for maximum security)
Source IP Pool:   Create new pool: 10.10.10.0/24
DNS Server:       192.168.50.1 (internal DNS)
```

Create SSL VPN Policy:

```
Incoming Interface: ssl.root
Outgoing Interface: [All internal VLANs as needed]
Source:             [SSL-VPN user group] + [VPN IP pool]
Destination:        all
Service:            ALL
Action:             ACCEPT
Security Profiles:  ✅ AntiVirus, ✅ IPS, ✅ Web Filter
```

---

## 10. Advanced Security Features

### Step 10.1 – DoS Protection Policy

Navigate to **Policy & Objects → IPv4 DoS Policy → Create New**:

```
Incoming Interface: WAN1
Source / Destination: all

Anomalies to enable (Action: Block):
  ✅ TCP SYN Flood        Threshold: 2000 packets/s
  ✅ UDP Flood            Threshold: 2000 packets/s
  ✅ ICMP Flood           Threshold: 250 packets/s
  ✅ TCP Port Scan        Threshold: 150 packets/s
  ✅ UDP Port Scan        Threshold: 150 packets/s
  ✅ IP Source Guard
```

### Step 10.2 – Enable ZTNA (Zero Trust Network Access)

FortiOS 7.2+ supports ZTNA as an alternative to traditional VPN. Enable via **Security Fabric → Zero Trust Network Access**:

1. **Deploy FortiClient EMS** for endpoint posture management.
2. Configure **ZTNA server** and **access proxy** on the FortiGate.
3. Create **ZTNA rules** based on user identity + device posture (OS patch level, AV status, etc.).

### Step 10.3 – Enable Security Fabric

Navigate to **Security Fabric → Fabric Connectors**:

1. Enable the **Security Fabric** to allow FortiGate to share threat intelligence with FortiAnalyzer, FortiManager, FortiClient EMS, and FortiSandbox.
2. Add all devices to the Fabric topology.
3. Enable **automated response**: e.g., quarantine an endpoint when a high-severity threat is detected.

### Step 10.4 – Two-Factor Authentication for Admins

1. Go to **System → Administrators → [admin account] → Edit**.
2. Under **Two-factor Authentication**, select **FortiToken Mobile** or **Email**.
3. Activate the token and test login.

### Step 10.5 – Geo-IP Blocking

Block access from countries that have no business reason to connect to your network:

Navigate to **Policy & Objects → Addresses → Create New**:

```
Name:   Block-High-Risk-Countries
Type:   Geography
Country: [Select countries to block]
```

Add this to the inbound WAN policy as a Source Block:

```
Name:               Block-Countries-Inbound
Incoming Interface: WAN1
Outgoing Interface: any
Source:             Block-High-Risk-Countries
Destination:        all
Service:            ALL
Action:             DENY
Log:                Enable
```

> Place this policy **above** the Internet-to-DMZ rules.

### Step 10.6 – Botnet C&C Detection

Navigate to **Security Profiles → AntiVirus → [profile] → Botnet C&C**:

```
Scan Outbound Connections to Botnet Sites: Block
```

Also enable via FortiGuard:

```
config ips global
    set block-malicious-url enable
end
```

---

## 11. Logging & Monitoring

### Step 11.1 – Configure Local Logging

Navigate to **Log & Report → Log Settings**:

```
Local Log:
  ✅ Enable disk logging (on FortiGate 40F, logs are stored in memory/flash)
  
Log Level: Information (or Warning for high-traffic environments)

Log the following:
  ✅ Traffic (Allowed and Denied)
  ✅ Threat (IPS, AntiVirus, Web Filter events)
  ✅ Event (Admin login, system events, VPN)
  ✅ VPN
```

### Step 11.2 – Forward Logs to FortiAnalyzer (Recommended)

Navigate to **Log & Report → Log Settings → Remote Logging**:

```
FortiAnalyzer:
  Status:     Enable
  IP Address: <FortiAnalyzer IP>
  Encryption: Enable (TLS)
  Upload option: Real-time
```

### Step 11.3 – Configure Alert Emails

Navigate to **Log & Report → Alert Email**:

```
SMTP Server:  <your mail server>
From:         fortigate@yourdomain.com
To:           security-team@yourdomain.com

Alert on:
  ✅ Critical events (IPS block, malware detected, admin login failures)
  ✅ System events (HA failover, disk full, high CPU)
```

### Step 11.4 – Enable SNMP Monitoring (SNMPv3)

Navigate to **System → SNMP → Create New**:

```
Name:           SNMPv3-Monitor
SNMP Version:   v3
Username:       snmp-admin
Security Level: AuthPriv
Auth Protocol:  SHA-256
Auth Password:  <strong password>
Priv Protocol:  AES-256
Priv Password:  <strong password>
Queries:        Enable (restrict to Management VLAN host)
```

### Step 11.5 – Dashboard Monitoring

Set up the FortiGate dashboard with key widgets:

- **Security Rating** – Provides a score and recommendations for improving security posture.
- **Threat Map** – Real-time visualization of blocked threats.
- **Top Policy Hits** – Identify misconfigured or overly permissive policies.
- **FortiView** – Drill into traffic by source, destination, application, and threat.

---

## 12. Hardening Checklist

Use this checklist to validate your deployment:

### Network & Segmentation
- [ ] All VLANs created and assigned to correct physical/logical ports
- [ ] No unnecessary inter-VLAN routing (least privilege model)
- [ ] IoT devices isolated from corporate network
- [ ] Guest Wi-Fi isolated with client isolation enabled
- [ ] DMZ servers accessible from internet via VIP/port-forward only
- [ ] Management VLAN accessible only from dedicated admin hosts

### Firewall Policies
- [ ] Default deny-all policy at bottom of policy list
- [ ] All allowed policies use UTM security profiles
- [ ] Policies log all sessions (especially denied traffic)
- [ ] Inbound VIP policies are minimal and scoped to required services only
- [ ] Geo-IP blocking applied for high-risk countries

### Authentication & Access
- [ ] Default admin account renamed or disabled
- [ ] All admin accounts use strong, unique passwords
- [ ] Admin accounts protected with MFA (FortiToken / TOTP)
- [ ] Admin access restricted to Management VLAN (Trusted Hosts)
- [ ] SSL VPN requires MFA
- [ ] No HTTP access to management GUI (HTTPS only)
- [ ] SSH access disabled on external/guest-facing interfaces

### Firmware & Updates
- [ ] Latest stable FortiOS firmware installed
- [ ] FortiGuard subscriptions active (UTM, IPS, Web Filter, App Control, AntiVirus)
- [ ] Automatic FortiGuard update schedule configured
- [ ] Firmware update notifications enabled

### Security Profiles
- [ ] AntiVirus profile applied to all policies (with sandboxing enabled)
- [ ] IPS profile applied to all policies (critical/high signatures set to Block)
- [ ] Web Filter profile applied to internet-facing policies
- [ ] Application Control profile applied
- [ ] DNS Filter enabled with botnet C&C blocking
- [ ] SSL Deep Inspection enabled (CA distributed to endpoints)
- [ ] DoS policy applied on WAN interface
- [ ] Botnet C&C detection enabled

### Logging & Monitoring
- [ ] Logging enabled for all allowed and denied traffic
- [ ] Logs forwarded to FortiAnalyzer or syslog server
- [ ] Alert emails configured for critical events
- [ ] SNMPv3 configured for network monitoring
- [ ] Security Rating reviewed and critical findings addressed

### Physical Security
- [ ] FortiGate physically secured (locked rack/cabinet)
- [ ] Console port access restricted
- [ ] USB ports disabled if not required (System → Settings → USB Management)
- [ ] UPS installed to protect against power disruptions

---

## Additional Resources

- [FortiGate 40F Datasheet](https://www.fortinet.com/content/dam/fortinet/assets/data-sheets/fortigate-40f.pdf)
- [FortiOS 7.4 Administration Guide](https://docs.fortinet.com/document/fortigate/7.4.0/administration-guide)
- [FortiOS CLI Reference](https://docs.fortinet.com/document/fortigate/7.4.0/cli-reference)
- [Fortinet Security Best Practices](https://docs.fortinet.com/document/fortigate/7.4.0/best-practices)
- [FortiGuard Labs Threat Intelligence](https://www.fortiguard.com)
- [CIS Benchmark for Fortinet](https://www.cisecurity.org/benchmark/fortinet)

---

*Last updated: 2026-03-30 | FortiOS 7.4.x*
