# HomeLab - Enterprise-Style Security & Networking Environment

This repository documents my **personal HomeLab**, designed to simulate a **real-world enterprise security architecture**.
The lab integrates identity services, network security, endpoint protection, VPN access, and centralized logging/SIEM - all implemented, configured, and documented hands-on.

## Core Technologies

**The environment integrates:**

* Elasticsearch & Kibana (with SIEM)
* Microsoft Active Directory (AD DS + Enterprise CA)
* Elastic Agent Fleet with EDR
* Palo Alto Networks NGFW PA-220
* GlobalProtect Remote Access VPN
* Juniper Networks NetScreen 5GT
* Apache HTTPS Web Server (DMZ)

## Detailed Configuration Guides

**Read more about individual components and deployments:**

* [Elasticsearch & Kibana - Deployment & Configuration with AD TLS](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
* [Elastic Fleet - Deployment & Configuration with AD TLS](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md)
* [Juniper NetScreen - Deployment & Configuration](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md)
* [Palo Alto NGFW & GlobalProtect - Deployment & Configuration](https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md)

<img width="2151" height="990" alt="HomeLAB_hypotetical" src="https://github.com/user-attachments/assets/d1697037-79cf-4c32-a1e8-4e660edcaaed" />

---

## Homelab Architecture Overview

This homelab represents a **segmented, enterprise-style network** built to simulate realistic security, identity, and monitoring scenarios.

The environment is divided into the following zones:

* **Internal**
* **DMZ**
* **VPN**
* **External**

Traffic flows are **strictly controlled and inspected** using a Next-Generation Firewall and **IPSec tunnels**, closely mirroring real corporate network designs.

---

## Contents

1. Security Rules
2. Internal Network (192.168.0.0/24)
3. Internal Edge Routing - Juniper NetScreen 5GT
4. NG Firewall (PA-220) - Security Enforcement Point
5. DMZ Network (10.10.37.0/24)
6. External Network & Internet Access
7. Actual Real-Life Photo

---

## 1. Security Rules

### From Internal Zone

* Internal → External ✅ Allowed, ⚠️ Inspected
* Internal → DMZ 🔐↔️🔐 IPSec-tunneled
* Internal → GP VPN ✅ Allowed

### From GlobalProtect VPN Zone

* GP VPN → DMZ ✅ Allowed
* GP VPN → Internal ✅ Allowed
* GP VPN → External ⚠️ Not applicable (split tunneling enabled)

### From DMZ Zone

* DMZ → Internal 🔐↔️🔐 IPSec-tunneled
* DMZ → GP VPN ✅ Allowed
* DMZ → External 🚫 Blocked

### From External Zone

* External → DMZ ✅ Allowed (⚠️ only specific services, ⚠️ inspected, ⚠️ DNAT)
* External → Internal 🚫 Blocked
* External → GP VPN 🚫 Blocked

---

## 2. Internal Network (192.168.0.0/24)

The **Internal zone** hosts core identity, endpoint, and monitoring services.

### Active Directory

* **AD DC-01 (192.168.0.69)**
  Provides:

  * Authentication & authorization
  * Enterprise Root Certification Authority
  * DNS
  * IIS (Web Certificate Enrollment)

### Workstations

* `Workstation01 (192.168.0.99)` - domain-joined client
* `AdamPC (192.168.0.19)` - Elastic Stack node (non-domain)

### Elastic Stack

* Centralized **logging, monitoring, and security analytics**
* Data sources:

  * Internal systems via **Elastic Agent Fleet**

    * AD DC
    * Domain workstations (Elastic EDR)
    * Fleet Server running on Elasticsearch node
  * Palo Alto NGFW logs via Elastic Agent integration
* SIEM detection rules trigger alerts on suspicious or malicious activity

### Key Characteristics

* Free communication inside the Internal zone
* Controlled access to DMZ **via IPSec tunnel**
* Internet access is inspected and filtered
* All systems trust the **Enterprise Root CA**
* All services use **certificates issued by AD CS**

**Read more**:
* [Elasticsearch & Kibana with Active Directory](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
* [Elastic Agent Fleet with Active Directory](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md)

---

## 3. Internal Edge Routing - Juniper NetScreen 5GT

The **Juniper NetScreen 5GT** functions as an **internal edge router**, separating the Internal network from the NGFW.

### Interfaces

* Internal: `192.168.0.1`
* Transit toward NGFW: `10.0.0.2/24`

### Responsibilities

* Routes internal traffic
* Participates in a **site-to-site IPSec VPN** with the NGFW
* IPSec tunnel is **strictly limited to Internal ↔ DMZ traffic**
* Uses a **policy-based IPSec configuration**

**Read more**:
[Juniper NetScreen configuration details](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md)

---

## 4. NG Firewall (PA-220) - Central Security Enforcement

The **Palo Alto Networks PA-220 NGFW** is the **primary security control point** of the entire lab.

### Interfaces & Zones

* Internal/Transit: `10.0.0.1/24`
* DMZ: `10.10.37.1/24`
* External: `172.16.0.49/24`
* VPN (GlobalProtect): `10.10.52.0/24`

---

### Secure Internet Access

* Source NAT for internal users
* Malicious IP blocking (External Abuse lists)
* Palo Alto security profiles (AV, Anti-Spyware, Vulnerability Protection)
* **SSL Forward Proxy Decryption**, user-based via AD
* All security events forwarded to **Elastic SIEM**

---

### Securing the DMZ Web Server

* DNAT for external access
* Only HTTPS (TCP/443) allowed
* Anti-Virus, Anti-Vulnerability, and file upload protection
* **SSL Inbound Inspection Decryption** for full traffic visibility

---

### IPSec Site-to-Site VPN

* Tunnel: **Juniper NetScreen ↔ Palo Alto NGFW**
* Limited strictly to **Internal ↔ DMZ traffic**
* Simulates untrusted intermediate network segments
* Ensures confidentiality, integrity, and authentication
* Policy-based IPSec using **Proxy IDs**

---

### GlobalProtect VPN

* Portal & Gateway hosted on the NGFW
* Remote users connect from the External network
* VPN address pool: `10.10.52.0/24`
* Split tunneling enabled (Internet traffic not tunneled)
* VPN users have access to:

  * Internal network
  * DMZ services

---

### Active Directory Integration

* GlobalProtect authentication via **LDAPS**
* User-to-IP and user-to-group mappings retrieved from AD

**Read more**:
[Palo Alto NGFW & GlobalProtect configuration](https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md)

---

## 5. DMZ Network (10.10.37.0/24)

The **DMZ zone** hosts externally exposed services.

### Web Server

* **HTTPS Web Server - 10.10.37.45**

### Characteristics

* Certificate issued by Enterprise Root CA
* Accessible:

  * From Internal network **only via IPSec**
  * From Internet with **full inspection**
  * Freely by GlobalProtect VPN users

---

## 6. External Network & Internet Access

* ISP Router: `172.16.0.1`
* Source of:

  * External users
  * VPN client connections
* External users:

  * Can access DMZ only
  * Never reach Internal network directly
* All inbound traffic is inspected by the NGFW

---

## 7. Actual Real-Life Photo

![homelab\_inreallife\_photo](https://github.com/user-attachments/assets/7f9bd5e1-687e-4f28-be21-d5e0abb3afc3)
