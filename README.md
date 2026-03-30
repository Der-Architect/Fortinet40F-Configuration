# FortiGate 40F — Maximum Security Configuration Guide with VLAN Segmentation

A step-by-step guide to configure the **Fortinet FortiGate 40F** for maximum security using VLAN segmentation in a small-to-medium business or home-lab environment.

---

## Table of Contents

1. [Required Devices & Hardware](#1-required-devices--hardware)
2. [Network Topology Overview](#2-network-topology-overview)
3. [Step 1 — Initial Setup & Firmware Update](#step-1--initial-setup--firmware-update)
4. [Step 2 — Interface Configuration](#step-2--interface-configuration)
5. [Step 3 — VLAN Segmentation](#step-3--vlan-segmentation)
6. [Step 4 — DHCP Servers per VLAN](#step-4--dhcp-servers-per-vlan)
7. [Step 5 — Firewall Policies (Inter-VLAN & WAN)](#step-5--firewall-policies-inter-vlan--wan)
8. [Step 6 — Security Profiles](#step-6--security-profiles)
9. [Step 7 — DNS & DoT/DoH Filtering](#step-7--dns--dotdoh-filtering)
10. [Step 8 — IPS (Intrusion Prevention System)](#step-8--ips-intrusion-prevention-system)
11. [Step 9 — SSL/TLS Inspection](#step-9--ssltls-inspection)
12. [Step 10 — Two-Factor Authentication & Admin Hardening](#step-10--two-factor-authentication--admin-hardening)
13. [Step 11 — IPsec / SSL VPN](#step-11--ipsec--ssl-vpn)
14. [Step 12 — Logging & SIEM Integration](#step-12--logging--siem-integration)
15. [Step 13 — Scheduled Backups & Maintenance](#step-13--scheduled-backups--maintenance)
16. [Maximum Security Hardening Checklist](#maximum-security-hardening-checklist)

---

## 1. Required Devices & Hardware

| Device | Purpose | Notes |
|---|---|---|
| **FortiGate 40F** | Next-Generation Firewall / UTM | Requires active FortiCare + UTM/UTP subscription for security features |
| **Managed Layer-2/3 Switch** (e.g., Cisco SG350, Ubiquiti UniFi, HP/Aruba) | VLAN trunk distribution | Must support 802.1Q VLAN tagging |
| **Wi-Fi Access Point** (e.g., FortiAP 231F, Ubiquiti UniFi AP) | Wireless segmentation per SSID/VLAN | Recommend FortiAP for native FortiGate integration |
| **ISP Router / Modem** | WAN uplink | Put in bridge/DMZ mode so FortiGate owns the WAN IP |
| **Management Workstation** | Initial configuration | Laptop/PC with Ethernet or browser access |
| **UPS (Uninterruptible Power Supply)** | Power protection | Protects firewall and switch from outages |
| *(Optional)* **FortiAnalyzer / FortiSIEM** | Centralized logging & analytics | Can use FortiCloud as a free alternative |
| *(Optional)* **FortiAuthenticator** | MFA / RADIUS for VPN and admin | Adds strong identity controls |

### FortiGate 40F Specifications (quick reference)

- **Firewall throughput:** 5 Gbps
- **NGFW throughput:** 1 Gbps
- **Threat Protection throughput:** 600 Mbps
- **Interfaces:** 5 × GE RJ45 (WAN1, WAN2, 3 × LAN), 1 × USB, 1 × Console
- **PoE:** None (external PoE switch/injector needed for FortiAP)
- **Max VLANs:** 255

---

## 2. Network Topology Overview

```
Internet
    │
[ISP Modem] ── Bridge/DMZ mode
    │
[WAN1 - FortiGate 40F]
    │
[LAN (port3)] ── Trunk (802.1Q) ──► [Managed Switch]
                                         │
                         ┌───────────────┼────────────────────┐
                         │               │                    │
                    VLAN 10          VLAN 20              VLAN 30
                   (Trusted)        (IoT)               (Guest)
                 192.168.10.0/24  192.168.20.0/24     192.168.30.0/24
                         │               │                    │
                   PCs / Servers    Smart TVs /          Guest Wi-Fi
                   NAS / VoIP       Cameras / Printers   Clients

Optional:
VLAN 40 (Management)  192.168.40.0/24  ← FortiGate GUI, Switch mgmt only
VLAN 50 (DMZ)         192.168.50.0/24  ← Public-facing servers
```

**Traffic rules summary (default-deny between VLANs):**

- Trusted (10) → Internet: ✅ Full access (with inspection)
- IoT (20) → Internet: ✅ Limited outbound only
- IoT (20) → Trusted (10): ❌ Blocked
- Guest (30) → Internet: ✅ HTTP/HTTPS only
- Guest (30) → Trusted/IoT: ❌ Blocked
- Management (40) → FortiGate GUI: ✅ Admin only
- Any → DMZ (50): ✅ Specific published services only

---

## Step 1 — Initial Setup & Firmware Update

### 1.1 Factory Reset (if used device)

Connect to the console port (9600 8N1) and run:

```
execute factoryreset
```

Or hold the **Reset** button for 10 seconds until the power LED blinks.

### 1.2 Access the GUI

1. Connect a PC to **port2** (default LAN) with a static IP `192.168.1.x/24`.
2. Browse to `https://192.168.1.99`.
3. Log in: username `admin`, password *(blank by default — set immediately)*.

### 1.3 Set a Strong Admin Password

```
System → Administrators → admin → Change Password
```

Use at minimum a 20-character passphrase with mixed case, numbers, and symbols.

### 1.4 Update Firmware

```
Dashboard → Status → Firmware → Check for Updates
```

Always run the **latest stable GA (General Availability)** firmware. As of writing, FortiOS **7.4.x** is current GA. Review the release notes before upgrading in production.

```bash
# Alternative via CLI
execute backup full-config flash
execute update-now
```

### 1.5 Set System Hostname, Timezone, and NTP

```
System → Settings
  Hostname:  FGT-40F-PROD
  Timezone:  (your timezone)
  NTP Server: time.fortiguard.net  (or your internal NTP)
```

---

## Step 2 — Interface Configuration

### 2.1 Configure WAN1 (Internet uplink)

```
Network → Interfaces → wan1
  Role:   WAN
  Addressing:  DHCP  (or Static if ISP provides fixed IP)
```

### 2.2 Configure the Internal Trunk Port

Rename **port3** (or your chosen LAN port) to `TRUNK` for clarity:

```
Network → Interfaces → port3
  Alias:  TRUNK
  Role:   LAN
  Type:   Physical
  (Leave IP blank — sub-interfaces/VLANs carry the IPs)
```

Enable **LLDP** and **STP** on the switch side to match.

---

## Step 3 — VLAN Segmentation

Create a VLAN sub-interface for each zone. Navigate to:

```
Network → Interfaces → Create New → Interface
  Type: VLAN
  Interface: port3 (TRUNK)
```

### VLAN 10 — Trusted LAN

| Field | Value |
|---|---|
| Name | `VLAN10-TRUSTED` |
| VLAN ID | `10` |
| Role | `LAN` |
| IP/Netmask | `192.168.10.1/24` |
| Administrative access | HTTPS, SSH, PING *(restrict later to VLAN 40)* |

### VLAN 20 — IoT

| Field | Value |
|---|---|
| Name | `VLAN20-IOT` |
| VLAN ID | `20` |
| Role | `LAN` |
| IP/Netmask | `192.168.20.1/24` |
| Administrative access | PING only |

### VLAN 30 — Guest

| Field | Value |
|---|---|
| Name | `VLAN30-GUEST` |
| VLAN ID | `30` |
| Role | `LAN` |
| IP/Netmask | `192.168.30.1/24` |
| Administrative access | None |

### VLAN 40 — Management *(optional but recommended)*

| Field | Value |
|---|---|
| Name | `VLAN40-MGMT` |
| VLAN ID | `40` |
| Role | `LAN` |
| IP/Netmask | `192.168.40.1/24` |
| Administrative access | HTTPS, SSH, PING |

### VLAN 50 — DMZ *(for public servers)*

| Field | Value |
|---|---|
| Name | `VLAN50-DMZ` |
| VLAN ID | `50` |
| Role | `DMZ` |
| IP/Netmask | `192.168.50.1/24` |
| Administrative access | PING only |

### Managed Switch Configuration

On your managed switch, configure the uplink port to the FortiGate as an **802.1Q trunk** carrying VLANs 10, 20, 30, 40, 50. Configure each access port or Wi-Fi SSID with the appropriate VLAN access tag.

---

## Step 4 — DHCP Servers per VLAN

For each VLAN interface, configure a DHCP scope:

```
Network → Interfaces → [VLAN interface] → DHCP Server → Enable
```

| VLAN | Range | Gateway | DNS |
|---|---|---|---|
| VLAN10-TRUSTED | 192.168.10.100–200 | 192.168.10.1 | 192.168.10.1 (FortiGate DNS proxy) |
| VLAN20-IOT | 192.168.20.100–200 | 192.168.20.1 | 192.168.20.1 |
| VLAN30-GUEST | 192.168.30.100–200 | 192.168.30.1 | 192.168.30.1 |
| VLAN40-MGMT | 192.168.40.100–150 | 192.168.40.1 | 192.168.40.1 |
| VLAN50-DMZ | Static assignments | 192.168.50.1 | 192.168.50.1 |

Enable **DNS proxy** on each VLAN interface so you can apply DNS filtering:

```
Network → DNS → DNS Database → Enable DNS proxy on interfaces
```

---

## Step 5 — Firewall Policies (Inter-VLAN & WAN)

### 5.1 Default Deny All

FortiGate uses implicit deny, but make it explicit:

```
Policy & Objects → Firewall Policy → Create New
  Name:    DENY-ALL
  Srcintf: any
  Dstintf: any
  Action:  DENY
  Log:     All Sessions
  (Place this at the bottom of the policy list)
```

### 5.2 Trusted LAN → WAN (Internet)

```
Name:         TRUSTED-TO-WAN
Srcintf:      VLAN10-TRUSTED
Dstintf:      wan1
Source:       all
Destination:  all
Service:      ALL
Action:       ACCEPT
NAT:          Enable (outbound masquerade)
Security Profiles: (attach in Step 6)
Logging:      All Sessions
```

### 5.3 IoT → WAN (restricted outbound)

```
Name:         IOT-TO-WAN
Srcintf:      VLAN20-IOT
Dstintf:      wan1
Source:       all
Destination:  all
Service:      HTTP, HTTPS, DNS, NTP
Action:       ACCEPT
NAT:          Enable
Security Profiles: (attach in Step 6 — strict profile)
```

### 5.4 Guest → WAN (web only)

```
Name:         GUEST-TO-WAN
Srcintf:      VLAN30-GUEST
Dstintf:      wan1
Source:       all
Destination:  all
Service:      HTTP, HTTPS, DNS
Action:       ACCEPT
NAT:          Enable
Security Profiles: (attach web filter)
```

### 5.5 Block Inter-VLAN by default

Create explicit DENY policies between each sensitive VLAN pair (or rely on the default deny at the bottom). Key blocks:

```
IOT-TO-TRUSTED:  VLAN20-IOT  → VLAN10-TRUSTED  DENY
GUEST-TO-TRUSTED: VLAN30-GUEST → VLAN10-TRUSTED  DENY
GUEST-TO-IOT:    VLAN30-GUEST → VLAN20-IOT      DENY
ANY-TO-MGMT:     any          → VLAN40-MGMT     DENY  (except VLAN10-TRUSTED or VPN)
```

### 5.6 DMZ Inbound (published services)

```
Name:        WAN-TO-DMZ-HTTPS
Srcintf:     wan1
Dstintf:     VLAN50-DMZ
Destination: [VIP - Virtual IP for your server]
Service:     HTTPS
Action:      ACCEPT
Security Profiles: IPS, AV, Web Filter
```

Create a **Virtual IP (VIP)** under:
```
Policy & Objects → Virtual IPs → Create New
  External IP:  [WAN IP]
  Mapped IP:    192.168.50.10  (DMZ server)
  Port forward: 443 → 443
```

---

## Step 6 — Security Profiles

### 6.1 Antivirus Profile

```
Security Profiles → AntiVirus → Create New
  Name:  AV-STRICT
  Scan:  HTTP, HTTPS, FTP, SMTP, POP3, IMAP, SMB
  Action on virus: Block
  Fortiguard:  Enable virus outbreak prevention
  Grayware:    Block
```

### 6.2 Web Filter Profile

```
Security Profiles → Web Filter → Create New
  Name:  WF-TRUSTED
  FortiGuard Categories:
    - Malicious Websites:         Block
    - Phishing:                   Block
    - Spam URLs:                  Block
    - Proxy & Anonymizers:        Block
    - Peer-to-Peer:               Block (or Monitor)
    - Adult Content:              Block (adjust per policy)
  Safe Search:  Enforce
  YouTube Restrict:  Strict
```

For IoT, create **WF-IOT** with even tighter category restrictions (allow only what IoT devices need).

For Guest, create **WF-GUEST** with social media allowed but malicious/phishing blocked.

### 6.3 Application Control Profile

```
Security Profiles → Application Control → Create New
  Name:  APP-TRUSTED
  Categories:
    - Botnet:          Block
    - Anonymous Proxy: Block
    - P2P:             Monitor
    - High Risk:       Block
  Unknown Applications: Monitor
```

### 6.4 DNS Filter Profile

```
Security Profiles → DNS Filter → Create New
  Name:  DNS-STRICT
  FortiGuard DNS Filter:  Enable
  Block botnet C&C:       Enable
  Block malware domains:  Enable
  Block phishing:         Enable
```

Assign **DNS-STRICT** to all VLAN policies.

### 6.5 Email Filter (if mail flows through FortiGate)

```
Security Profiles → Email Filter → Create New
  Name:  EF-STRICT
  Spam Detection:  FortiGuard + Heuristic
  Action:  Tag subject with [SPAM]  or Block
```

### 6.6 File Filter

```
Security Profiles → File Filter → Create New
  Name:  FF-STRICT
  Block high-risk file types: .exe, .bat, .ps1, .vbs, .js, .jar, .hta, .scr
  Protocols:  HTTP, HTTPS, FTP, SMB, Email
```

---

## Step 7 — DNS & DoT/DoH Filtering

Force all clients to use the FortiGate as their DNS resolver to prevent DNS-over-HTTPS (DoH) bypass:

### 7.1 Block external DNS on all VLANs

```
Policy & Objects → Firewall Policy → Create New
  Name:      BLOCK-EXTERNAL-DNS
  Srcintf:   VLAN10-TRUSTED, VLAN20-IOT, VLAN30-GUEST
  Dstintf:   wan1
  Service:   DNS (UDP/TCP 53)
  Destination: all (excluding FortiGate itself)
  Action:    DENY
  (Place ABOVE the general WAN policies)
```

### 7.2 Block DoH (DNS over HTTPS to external resolvers)

Use Application Control to block known DoH providers (Google DNS, Cloudflare 1.1.1.1, etc.) or create a URL filter for known DoH endpoints:

```
Security Profiles → Application Control → APP-TRUSTED
  Add signature: DNS-over-HTTPS → Block
```

### 7.3 Enable FortiGuard DNS over TLS on the FortiGate itself

```
System → Feature Visibility → Enable DNS over TLS
Network → DNS
  Primary DNS:    208.91.112.220  (FortiGuard)
  Secondary DNS:  208.91.112.53
  DNS over TLS:   Enable
```

---

## Step 8 — IPS (Intrusion Prevention System)

```
Security Profiles → Intrusion Prevention → Create New
  Name:  IPS-STRICT
  Signature Database:  Extended (all signatures)
  IPS Signatures:
    - Severity Critical:  Block + Reset
    - Severity High:      Block + Reset
    - Severity Medium:    Block
    - Severity Low:       Monitor
    - Severity Info:      Monitor
  Protocol Decoders:  Enable all relevant
  Botnet C&C:         Block
```

Apply **IPS-STRICT** to:
- All outbound (VLAN→WAN) policies
- All inbound (WAN→DMZ) policies
- All high-risk inter-VLAN paths

Keep the IPS signature database **auto-updated**:

```
System → FortiGuard → Intrusion Prevention → Auto Update: Enabled
  Update interval: Every 1 hour
```

---

## Step 9 — SSL/TLS Inspection

Deep packet inspection breaks SSL to allow AV, IPS, and Web Filter to scan encrypted traffic.

### 9.1 Create SSL/TLS Inspection Profile

```
Security Profiles → SSL/SSH Inspection → Create New
  Name:  SSL-DEEP
  SSL Inspection Mode:  Full SSL Inspection
  CA Certificate:  Fortinet_CA_SSL  (or your own CA)
  Exempt:  Corporate banking sites, health portals (as needed)
  Invalid SSL Certificates:  Block
  Untrusted SSL Certificates: Block
```

### 9.2 Install the FortiGate CA on all clients

Export the CA:
```
Security Profiles → SSL/SSH Inspection → Download Certificate
```

Push via **Group Policy (Windows)**, **MDM (mobile)**, or **manual install** on each device. Without this, browsers will show certificate warnings.

### 9.3 Apply to firewall policies

Attach **SSL-DEEP** to all ACCEPT policies where you want content inspection:
- TRUSTED-TO-WAN
- IOT-TO-WAN
- GUEST-TO-WAN
- WAN-TO-DMZ-*

---

## Step 10 — Two-Factor Authentication & Admin Hardening

### 10.1 Restrict GUI/SSH access to Management VLAN only

```
System → Administrators → admin
  Trusted Hosts:  192.168.40.0/24  (Management VLAN only)
```

Disable admin access on all non-management interfaces:

```
Network → Interfaces → [each interface] → Administrative Access
  Remove HTTPS and SSH from non-MGMT VLANs
```

### 10.2 Enable MFA for admin accounts

```
System → Administrators → admin → Two-factor Authentication
  Type:  FortiToken Mobile  (or email OTP as a minimum)
```

Assign a **FortiToken** or configure **email-based OTP** as a fallback.

### 10.3 Change management ports

```
System → Settings
  HTTPS port:  8443  (change from 443)
  SSH port:    2222  (change from 22)
```

### 10.4 Lockout Policy

```
System → Settings
  Admin Lockout Threshold:  3 failed attempts
  Admin Lockout Duration:   300 seconds (5 minutes)
```

### 10.5 Disable unused services

```
System → Feature Visibility
  Disable:  WAN Optimization (if unused), Endpoint Control (if unused)

System → Settings
  Disable:  USB auto-install
  Disable:  Auto-run on USB
```

### 10.6 Create read-only admin accounts

Avoid sharing the primary admin credentials. Create role-based accounts:

```
System → Administrators → Create New
  Profile:  prof_admin (read-only)
  Trusted Hosts:  192.168.40.0/24
```

---

## Step 11 — IPsec / SSL VPN

### 11.1 SSL VPN (remote user access)

```
VPN → SSL-VPN Settings
  Listen on Interface:  wan1
  Listen on Port:  10443  (non-standard port)
  Restrict Access:  Limit to trusted IPs or countries
  Idle Timeout:  300 seconds
  Authentication:  Require certificate + MFA
  Tunnel Mode:  Full Tunnel (routes all traffic via FortiGate)
```

Create an SSL VPN portal:
```
VPN → SSL-VPN Portals → Create New
  Name:  VPN-USERS
  Tunnel Mode:  Enable
  Split Tunneling:  Disabled  (route all traffic for maximum security)
  DNS Server:  192.168.40.1
```

Create a firewall policy for SSL VPN traffic:
```
Srcintf:  ssl.root
Dstintf:  VLAN10-TRUSTED (or specific resources)
Source:   VPN-USERS group
Action:   ACCEPT
Security Profiles: Apply same AV/IPS/WebFilter as LAN
```

### 11.2 IPsec VPN (site-to-site)

```
VPN → IPsec Wizard
  Template:  Site to Site
  Authentication:  Pre-shared Key (or certificates for higher security)
  IKE Version:  2
  Encryption:  AES-256-GCM
  Diffie-Hellman Group:  20 (ECDH P-384) or 21
  Lifetime:  86400 seconds (phase 1), 3600 (phase 2)
  PFS:  Enable
```

---

## Step 12 — Logging & SIEM Integration

### 12.1 Enable comprehensive logging

```
Log & Report → Log Settings
  Local Logging:  Enable (disk or memory)
  Log Level:  Information
  Log traffic:  All sessions (or at minimum: violations + threats)
```

### 12.2 Forward to FortiAnalyzer / Syslog / SIEM

```
Log & Report → Log Settings → Remote Logging
  Remote Syslog:  Enable
  IP:  [SIEM IP]
  Port:  514 (UDP) or 6514 (TLS)
  Facility:  local7
  Format:  CEF (Common Event Format) or LEEF
```

For **FortiAnalyzer**:
```
Log & Report → FortiAnalyzer/FortiManager
  IP:  [FortiAnalyzer IP]
  Upload Option:  Real-Time
```

### 12.3 Enable Security Rating

```
Security Fabric → Security Rating → Run Assessment
```

Review and act on all **Critical** and **High** findings regularly.

### 12.4 Set up email alerts

```
Log & Report → Alert Email
  SMTP Server:  [your mail server]
  Recipients:   admin@yourdomain.com
  Alerts:  Admin login, VPN login failure, IPS critical event, Virus detected
```

---

## Step 13 — Scheduled Backups & Maintenance

### 13.1 Automated configuration backup

```
System → Maintenance → Backup & Restore
  Schedule:  Daily
  Destination:  FTP/SFT server or FortiCloud
  Encryption:  Enable (AES-256 passphrase)
```

CLI alternative:
```bash
execute backup config ftp <filename> <FTP server> <user> <password>
```

### 13.2 FortiGuard subscription renewal reminders

Ensure the following subscriptions are active:
- **FortiCare** (hardware support)
- **FortiGuard UTM/UTP** bundle (AV, IPS, Web Filter, App Control, DNS Filter)
- **FortiGuard Outbreak Prevention** (real-time threat intel)

Check status:
```
Dashboard → Licenses
```

### 13.3 Firmware maintenance window

Schedule a monthly review of FortiOS release notes and test firmware upgrades in a **staging environment** before production rollout.

---

## Maximum Security Hardening Checklist

Use this checklist to verify your deployment meets maximum security standards:

### Firmware & Software
- [ ] FortiOS firmware is on the latest stable GA release
- [ ] FortiGuard definitions are current (AV, IPS, Web Filter, App Control)
- [ ] All FortiGuard subscriptions are active and not expired

### Access Control
- [ ] Default `admin` password changed to a strong passphrase (20+ characters)
- [ ] MFA (FortiToken or email OTP) enabled for all administrator accounts
- [ ] Admin GUI/SSH access restricted to Management VLAN (VLAN 40) trusted hosts only
- [ ] Management ports changed from defaults (443 → 8443, 22 → 2222)
- [ ] USB auto-install disabled
- [ ] Admin lockout policy configured (3 attempts, 5-minute lockout)
- [ ] Read-only admin accounts used for day-to-day monitoring

### Network Segmentation
- [ ] VLANs created for Trusted, IoT, Guest, Management, and DMZ
- [ ] Managed switch configured with 802.1Q trunking to FortiGate
- [ ] Wi-Fi SSIDs mapped to appropriate VLANs
- [ ] IoT VLAN cannot reach Trusted VLAN
- [ ] Guest VLAN cannot reach Trusted or IoT VLANs
- [ ] Management VLAN access is restricted to admin workstations only

### Firewall Policies
- [ ] Default-deny policy at the bottom of the policy list
- [ ] Outbound policies apply NAT and security profiles
- [ ] Inter-VLAN deny rules explicitly configured
- [ ] Inbound WAN access only allows published services via VIPs/VIPs
- [ ] All policies have logging enabled

### Security Profiles
- [ ] Antivirus profile enabled and attached to all policies
- [ ] IPS (Extended signatures) enabled, Critical/High set to Block
- [ ] Web Filter with malicious/phishing/botnet categories blocked
- [ ] Application Control blocking botnets, anonymous proxies
- [ ] DNS Filter blocking botnet C&C and malicious domains
- [ ] File Filter blocking high-risk file types
- [ ] SSL/TLS Deep Inspection enabled on outbound and inbound policies
- [ ] FortiGate CA certificate deployed to all client devices

### DNS Security
- [ ] External DNS (port 53) blocked on all VLANs (FortiGate is the only DNS resolver)
- [ ] DoH (DNS over HTTPS) to external resolvers blocked
- [ ] FortiGate uses FortiGuard DNS over TLS upstream

### VPN
- [ ] SSL VPN listens on non-standard port (not 10443/443 if possible)
- [ ] SSL VPN uses full-tunnel (no split tunneling)
- [ ] SSL VPN requires certificate + MFA
- [ ] IPsec uses IKEv2, AES-256-GCM, DH Group 20+, PFS enabled

### Logging & Monitoring
- [ ] All traffic logging enabled (at minimum violations and threats)
- [ ] Logs forwarded to external SIEM or FortiAnalyzer
- [ ] Email alerts configured for critical events
- [ ] Security Rating assessment reviewed and critical findings resolved

### Backup & Maintenance
- [ ] Automated encrypted configuration backups scheduled daily
- [ ] Firmware maintenance window defined monthly
- [ ] Incident response plan documented and tested

---

## Quick Reference: Key CLI Commands

```bash
# Show system status
get system status

# Show interface summary
get system interface

# Show active firewall sessions
diagnose sys session list

# Show routing table
get router info routing-table all

# Check FortiGuard connectivity
diagnose debug rating

# Show IPS statistics
diagnose ips session list

# Check CPU and memory
get system performance status

# Backup configuration to console
execute backup config ftp daily-backup.conf <ftp-ip> ftpuser ftppass

# Manually trigger FortiGuard update
execute update-now

# Flush DNS cache
diagnose ip address list
diagnose test app dnsproxy 9
```

---

## References

- [Fortinet Documentation Library](https://docs.fortinet.com/)
- [FortiOS 7.4 Administration Guide](https://docs.fortinet.com/product/fortigate/7.4)
- [FortiGate Best Practices](https://docs.fortinet.com/document/fortigate/7.4.0/best-practices/)
- [FortiGuard Labs Threat Intelligence](https://www.fortiguard.com/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CIS Benchmarks for Network Devices](https://www.cisecurity.org/cis-benchmarks/)

---

*This guide is provided for educational purposes. Always test configuration changes in a lab environment before applying to production. Security requirements vary by organization — adapt policies to your specific threat model and compliance requirements.*
