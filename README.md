# Fortinet40F-Configuration

A comprehensive, step-by-step guide for configuring the **Fortinet FortiGate 40F** for **maximum security** with **VLAN segmentation**.

## Contents

| File | Description |
|---|---|
| [docs/fortigate-40f-max-security-guide.md](docs/fortigate-40f-max-security-guide.md) | Full configuration guide – hardware list, VLAN design, firewall policies, security profiles, VPN, logging, and hardening checklist |

## What's Covered

- **Required devices & hardware** – FortiGate 40F, managed switch, access point, FortiAnalyzer, FortiToken, and more
- **VLAN Segmentation** – Trusted, IoT, Guest, DMZ, and Management VLANs with a least-privilege inter-VLAN policy
- **Initial setup** – Firmware update, secure management access, MFA for admins
- **Firewall policies** – Zone-based rules with explicit deny-all at the bottom
- **Security profiles** – AntiVirus, IPS, Web Filter, Application Control, DNS Filter
- **SSL/TLS Deep Inspection** – Decrypting and inspecting encrypted traffic
- **VPN** – IPsec site-to-site (IKEv2/AES-256) and SSL VPN with MFA
- **Advanced features** – DoS protection, Geo-IP blocking, Botnet C&C detection, Zero Trust (ZTNA)
- **Logging & monitoring** – FortiAnalyzer, SNMP v3, alert emails, FortiView dashboard
- **Hardening checklist** – Final validation checklist before going live

## Quick Start

1. Read the [full guide](docs/fortigate-40f-max-security-guide.md)
2. Follow sections in order – each section builds on the previous
3. Use the [Hardening Checklist](docs/fortigate-40f-max-security-guide.md#12-hardening-checklist) to validate your deployment

> **FortiOS Version**: Guide targets FortiOS 7.4.x
