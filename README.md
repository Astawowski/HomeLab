# HomeLab
* **My own environment integrating:**
  * Elasticsearch & Kibana
  * Microsoft Active Directory
  * Fleet of Elastic Agents with EDR
  * PaloAlto NGFW PA-220
  * GlobalProtect VPN 
  * Juniper Networks NetScreen 5GT
  * Simple Apache HTTPS Web Server

* **GitHub Links:**
  * [Elasticsearch & Kibana - Deployment & Configuration with AD TLS](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
  * [Elastic Fleet - Deployment & Configuration with AD TLS](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md)
  * [Juniper NetScreen - Deployment & Configuration](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md)
    
<img width="2151" height="990" alt="HomeLAB_hypotetical" src="https://github.com/user-attachments/assets/d1697037-79cf-4c32-a1e8-4e660edcaaed" />


---

## Homelab Architecture Overview

This homelab represents a **segmented enterprise-style network** designed to simulate real-world security, identity, and monitoring scenarios. The environment is split into **Internal**, **DMZ**, **VPN**, and **External** zones, with controlled traffic flows enforced by NGFW firewall and IPSec tunnels.

---

## Contents

1. Security Rules
2. Internal Network (192.168.0.0/24)
3. Internal Edge Routing – Juniper NetScreen 5GT
4. NG Firewall (PA-220) – Security Enforcement Point
5. DMZ Network (10.10.37.0/24)
6. Remote Access VPN (GlobalProtect)
7. External Network & Internet Access
8. Actual Real Life Photo

---

## 1. Security Rules:

* **From Internal:**
  * Internal → External ✅Allowed  ⚠️Inspected
  * Internal → DMZ      🔐↔️🔐Tunneled
  * Internal → GP VPN   ✅Allowed

* **From GlobalProtect VPN Zone:**
  * GP VPN → DMZ        ✅Allowed
  * GP VPN → Internal   ✅Allowed
  * GP VPN → External   ⚠️Does not apply (split-tunneling configured)
   
* **From DMZ Zone:**
  * DMZ → Internal      🔐↔️🔐Tunneled
  * DMZ → GP VPN        ✅Allowed
  * DMZ → External      🚫Blocked
   
* **From External:**
  * External → DMZ      ✅Allowed  ⚠️Only specific services ⚠️Inspected
  * External → Internal 🚫Blocked
  * External → GP VPN   🚫Blocked

---

## 2. Internal Network (192.168.0.0/24)

The **Internal zone** hosts core identity and monitoring services:

* **Active Directory Domain Controller (AD DC-01 – 192.168.0.69)**
  Provides Authentication, Enterprise Root CA, DNS, IIS (Web certificate Enrollment) and user directory services.

* **Workstations**

  * `Workstation01 (192.168.0.99)` – domain-joined client
  * `AdamPC (192.168.0.19)` – Elastic Stack node (non-domain)

* **Elastic Stack**

  * Used for **logging, monitoring, and security analytics**
  * Collects data from:
    * internal network systems by using **Elastic Agents Fleet**: `AD DC`, `AD Workstation` - Elastic EDR, `Elastic Fleet Server` on Elasticsearch node
    * NGFW via Elastic Agent PaloAlto integration.
  * Fires alerts should events violate detection SIEM rules.

* All internal devices communicate freely within the zone, with VPN users, **via IPSec tunnel** to DMZ and have inspected external (e.g. Internet) traffic.
* They rely on AD for identity services.
* Every system on the network trusts AD Root CA and every service have certificate issued by it.

Click [here](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md) to see how is `Elasticsearch` and `Kibana` configured with `Active Directory`. 

Click [here](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md) to see how is `Elastic Agents Fleet` configured with `Active Directory`.

---

## 3. Internal Edge Routing – Juniper NetScreen 5GT

The **Juniper NetScreen 5GT** acts as an **internal edge router**, separating the Internal network from the NGFW.

* Internal interface: `192.168.0.1`
* Transit interface toward NGFW: `10.0.0.2/24`

Its key role is to:

* Route internal traffic
* Participate in a **site-to-site IPSec VPN** with the NG Firewall for Internal ↔ DMZ traffic **only**.
* This device utilizes a **policy-based IPSec tunnel**.

Click [here](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md) to see how is Juniper NetScreen configured.

---

## 4. NG Firewall (PA-220) – Security Enforcement Point

The **Next-Generation Firewall** is the **central security control point** in the lab. It enforces security policy rules from the beggining of this document.

Interfaces and zones:

* **Transit/Internal side:** `10.0.0.1/24` and `192.168.0.0/24`
* **DMZ:** `10.10.37.1/24`
* **External:** `172.16.0.49/24`
* **VPN zone:** `10.10.52.0/24` (Provided via GlobalProtect)

## Secure Internet Access

* This device provides **secure Internet access** for Internal User/Systems by performing sNAT, blocking malicious IPs (External AbuseCH list) and enforcing various PaloAlto Security profiles.
* It also performs **SSL Forward Proxy Decryption** based on AD User. (Some users are more risky and require full visibility into their encrypted traffic).
* Every threat blocked is **reported to Elastic SIEM** via Log Forwarding.

## Securing DMZ Web Server

* This device **protects Internal Web Server** in DMZ from External Users by enforcing PaloAlto Anti-Vulnurability, Anti-Virus and File upload blocking.
* It also allows only specific service access (TCP 443 - HTTPS)
* Performs **SSL Inbound Inspection Decryption** so as to have full visibility into incoming encrypted traffic.

## IPSec Site-to-Site VPN

* Tunnel between **Juniper NetScreen ↔ NG Firewall**
* **Strictly limited to DMZ ↔ Internal traffic**
* This way we simulate a scenario in which there are multiple not fully trusted devices between Juniper NetScreen and NG Firewall. We achieve perfect secrecy and authentication.
* External/Internet/GP-VPN-Users-bound traffic is explicitly excluded from the tunnel

This device utilizes a **policy-based IPSec tunnel with security policy enforcement**. (utilizes Proxy ID)

## Global Protect VPN

* This device hosts a **GlobalProtect Portal & Gateway**, allowing remote users (from External Network `172.16.0.49/24`) to access enterprise network securely via VPN Users Zone.
* Internet-bound traffic is not tunneled via GlobalProtect VPN.

## Active Directory Integration

* This device authenticates GlobalProtect VPN Users using **LDAPS**.
* It also gathers **Username-IP and Username-Group mappings** from AD DC. (also using LDAPS)

---

## 5. DMZ Network (10.10.37.0/24)

The **DMZ zone** hosts exposed services:

* **Web Server (10.10.37.45)**

Key characteristics:

* Accessible from the Internal network **only via the IPSec tunnel**
* Accessible from the Internet **and fully inspected** (SSL Inbound Inspection Decryption)
* Accessible freely for GP VPN Users.
---

## 6. Remote Access VPN (GlobalProtect)

Remote users connect using **GlobalProtect VPN**, terminating on the NG Firewall.

* VPN address pool: `10.10.52.0/24`
* Users originate from the **External network (172.16.0.0/24)**
* They can freely access Internal Resources and DMZ.

---

## 7. External Network & Internet Access

* **Router ISP (172.16.0.1)** provides upstream Internet access
* External users and VPN clients originate here
* External users can only access DMZ
* Traffic from the Internet to DMZ is fully inspected by the NG Firewall
  

---

## 8. Actual Real Life Photo

![homelab_inreallife_photo](https://github.com/user-attachments/assets/7f9bd5e1-687e-4f28-be21-d5e0abb3afc3)


