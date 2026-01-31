# Palo Alto Networks PA-220 – Configuration Overview

This document describes how the **Palo Alto Networks PA-220 Next-Generation Firewall** is configured and utilized as the **central security enforcement point** in the lab environment.

The firewall provides **traffic inspection, segmentation, VPN termination, identity awareness, decryption, and logging**, while legacy and edge devices are intentionally kept minimal to simulate real-world, multi-layered enterprise networks.

<img width="468" height="299" alt="ngfw du" src="https://github.com/user-attachments/assets/293e1cd1-33ae-47f0-bb62-76cbec88a86f" />




---

## Contents:

1. Basic Device Settings
2. Interfaces, Management Profiles, and Zones
3. Virtual Router – Static Routing
4. IPsec Tunnel and IKE Gateway (DMZ ↔ Internal)
5. Policy-Based Forwarding (PBF) for IPsec
6. Source NAT (sNAT) Policy
7. LDAPS Authentication Profile
8. User-ID Agent (Windows-based)
9. User Group Mapping
10. GlobalProtect Portal & Gateway
11. External Dynamic C2 IPs List
12. Security Policy Rules
13. SSL Decryption Policies
14. Log Forwarding to Elastic SIEM

---

## 1. Basic Device Settings

The PA-220 is configured with essential system services to ensure **secure management, accurate timekeeping, application awareness, and cryptographic trust**.

### Configured system parameters:

* **Hostname:** `nilfgard-firewall-01`
* **Domain:** `lab.local`
* **Management interface access:** `HTTPS` & `SSH` 
* **Applications & Threats:** Updated to **latest available App-ID content**

<img width="250" height="180" alt="image" src="https://github.com/user-attachments/assets/2936ee48-5289-4158-b1e9-d1f5620c7991" />
<img width="502" height="204" alt="image" src="https://github.com/user-attachments/assets/33e950ae-ad9c-462f-b5ea-432bfbdc0659" />


### DNS Configuration:

* **Primary DNS:** `192.168.0.69` (AD DC)
* **Domain suffix:** `lab.local`

### NTP Configuration:

* **NTP Server:** `time.google.com`
* **Time zone:** `GMT+1`
* Accurate time is required for:

  * Certificate validation
  * VPN lifetimes
  * Log correlation in SIEM
 
<img width="382" height="172" alt="image" src="https://github.com/user-attachments/assets/7cbc3299-07bf-4f87-937b-cad1e97a6d66" />

### Certificates & TLS:

* **Imported certificates:**

  * Internal Root CA (from AD CS)
  * Firewall management certificate
  * GlobalProtect Portal & Gateway certificate
  * SSL Decryption Forward Proxy Trust & UnTrust certificate
  * DMZ Webserver certificate for SSL Inbound Inspection
    
<img width="425" height="201" alt="image" src="https://github.com/user-attachments/assets/d2c996b3-655d-4ad3-b1e3-d5758ed707af" />

### Service Routes:

* Custom service routes are defined:
  * From `ETH 1/8` (Internal):
    * DNS
    * LDAP
    * Syslog
  * From `ETH 1/2` (External):
    * EDLs
    * NTP
    * PaloAlto Networks Services

---

## 2. Interfaces, Management Profiles, and Zones

Physical and logical interfaces are configured to enforce **clear separation of trust levels**.

### Interfaces & Zones:

| Interface   | IP Address     | Zone      | Purpose                 |
| ----------- | -------------- | --------- | ----------------------- |
| ethernet1/1 | 10.0.0.1/24    | Transit   | Connection to NetScreen |
| ethernet1/2 | 10.10.37.1/24  | DMZ       | DMZ services            |
| ethernet1/3 | 172.16.0.49/24 | Untrust   | ISP / Internet          |
| tunnel.1    | —              | VPN-IPSec | Site-to-site VPN        |
| tunnel.10   | —              | VPN-GP    | GlobalProtect           |

### Management Profiles:

* Applied per interface
* Allow:

  * Ping (ICMP)
  * HTTPS (only where required)
* No management exposed on Untrust unless explicitly needed

---

## 3. Virtual Router – Static Routing

A single Virtual Router is used to manage all routing decisions.

### Static routes:

* **Default route:**

  * Destination: `0.0.0.0/0`
  * Next hop: `172.16.0.1` (ISP router)

* **Internal network route:**

  * Destination: `192.168.0.0/24`
  * Next hop: `10.0.0.2` (NetScreen)

* **VPN-learned routes:**

  * DMZ subnet reachable via IPsec tunnel

Routing ensures:

* Internet traffic exits directly
* Internal traffic is reachable via NetScreen
* DMZ traffic follows VPN or policy rules

---

## 4. IPsec Tunnel and IKE Gateway (DMZ ↔ Internal)

An IPsec site-to-site VPN is configured to secure **Internal ↔ DMZ traffic only**.

### IKE Gateway:

* **Name:** `IKE_To_NetScreen`
* **Version:** IKEv1
* **Authentication:** Pre-Shared Key
* **Peer Address:** `10.0.0.2`
* **Crypto:** AES-128 / SHA-1 / DH Group 2

### IPsec Tunnel:

* **Tunnel interface:** `tunnel.1`
* **Local Proxy ID:** `10.10.37.0/24`
* **Remote Proxy ID:** `192.168.0.0/24`
* **Perfect Forward Secrecy:** Enabled

The tunnel encrypts traffic **only when explicitly matched by policy**.

---

## 5. Policy-Based Forwarding (PBF)

Policy-Based Forwarding is used to **force DMZ-bound traffic into the IPsec tunnel**, regardless of routing table.

### PBF rule logic:

* **Source:** `192.168.0.0/24`
* **Destination:** `10.10.37.0/24`
* **Action:** Forward
* **Egress interface:** `tunnel.1`
* **Fallback:** Disabled

This guarantees:

* DMZ traffic is always encrypted
* All other traffic follows normal routing

---

## 6. Source NAT (sNAT) Policy

Source NAT is configured for Internet-bound traffic.

### NAT policy:

* **From zone:** Trust / VPN / DMZ
* **To zone:** Untrust
* **Translation:** Dynamic IP and Port
* **Translated address:** Untrust interface IP

Internal and VPN hosts access the Internet using firewall’s public identity.

---

## 7. LDAPS Authentication Profile

The firewall integrates with Active Directory using **LDAPS** for secure authentication.

### LDAPS profile:

* **Server:** `192.168.0.69`
* **Port:** `636`
* **Certificate validation:** Enabled
* **Bind account:** Dedicated service account
* **Base DN:** `DC=lab,DC=local`

This enables:

* User authentication
* Group resolution
* GlobalProtect authentication

---

## 8. User-ID Agent (Windows-based)

A **Windows User-ID Agent** is deployed on a domain-joined server.

### Function:

* Collects IP ↔ username mappings from:

  * Event logs
  * Domain logons
* Sends mappings securely to PA-220

This allows:

* User-based security policies
* Identity-aware logging
* Correlation in SIEM

---

## 9. User Group Mapping

The firewall retrieves **Active Directory group membership**.

### Configuration:

* LDAP group mapping enabled
* Security groups imported (e.g.):

  * `IT-Admins`
  * `VPN-Users`
  * `SOC-Analysts`

Policies can reference **groups instead of IPs**.

---

## 10. GlobalProtect Portal & Gateway

Remote access VPN is provided using **GlobalProtect**.

### Portal:

* Handles client configuration
* Uses internal certificate
* Authenticates users via AD

### Gateway:

* Assigns IPs from `10.10.52.0/24`
* Enforces security policies
* Applies User-ID automatically

Provides **secure remote trusted access from outside**.

---

## 11. External Dynamic C2 IPs List

An External Dynamic List (EDL) is configured to track **command-and-control infrastructure**.

### EDL source:

* HTTP-based threat feed
* Auto-refreshed periodically
* Used in security policies

Enables fast blocking of:

* Malware callbacks
* Known attacker infrastructure

---

## 12. Security Policy Rules

Security policies define **who can talk to whom, and how**.

### Example rules:

* Internal → DMZ: Permit (via IPsec)
* VPN Users → Internal: Permit (group-based)
* DMZ → Internal: Deny (default)
* Any → Internet: Permit + inspection
* C2 EDL → Any: Deny + log

All policies use:

* App-ID
* User-ID
* Logging at session end

---

## 13. SSL Decryption Policies

SSL/TLS decryption is enabled for visibility.

### Decryption scope:

* Internal users → Internet
* Exclusions:

  * Banking
  * Health services
  * Certificate-pinned apps

Certificates:

* Enterprise Root CA deployed to endpoints

This allows:

* Malware detection
* Credential theft visibility
* Full App-ID accuracy

---

## 14. Log Forwarding to Elastic SIEM

All logs are forwarded to **Elastic SIEM** via Syslog.

### Forwarded log types:

* Traffic
* Threat
* System
* Authentication
* VPN

### Benefits:

* Centralized visibility
* Correlation with endpoint logs
* Incident response readiness
* Compliance reporting

---

## Resulting State

* PA-220 acts as **single security brain**
* Legacy devices provide connectivity only
* Traffic is segmented, encrypted, inspected, and logged
* Identity and user context is fully integrated
* Remote users are securely onboarded
* Full visibility is available in Elastic SIEM

---

If you want, next we can:

* **polish language to “official Palo Alto style”**
* **shorten for README.md**
* **split into multiple articles**
* **add a “Why this design” section**
* **map to NIS2 / DORA / TIBER-EU**

Just say the word.
