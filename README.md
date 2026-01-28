# HomeLab
* **My own environment integrating:**
  * Elasticsearch & Kibana [See how it's deployed with AD](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
  * Microsoft Active Directory [See how it's configured with Elasticstack](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
  * Fleet of Elastic Agents with EDR [See how fleet is deployed](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md)
  * PaloAlto NGFW PA-220
  * GlobalProtect VPN (in progress...)
  * Juniper Networks NetScreen 5GT [See how NetScreen is configured](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md)
  * Example Web Server (in progress...)

<img width="2151" height="990" alt="HomeLAB_hypotetical" src="https://github.com/user-attachments/assets/d1697037-79cf-4c32-a1e8-4e660edcaaed" />


---

## Homelab Architecture Overview

This homelab represents a **segmented enterprise-style network** designed to simulate real-world security, identity, and monitoring scenarios. The environment is split into **Internal**, **DMZ**, **VPN**, and **External** zones, with controlled traffic flows enforced by firewalls and IPSec tunnels.

## Security Rules:

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

## 1. Internal Network (192.168.0.0/24)

The **Internal zone** hosts core identity and monitoring services:

* **Active Directory Domain Controller (AD DC-01 – 192.168.0.69)**
  Provides Authentication, Enterprise Root CA, DNS, IIS (Web certificate Enrollment) and user directory services.

* **Workstations**

  * `Workstation01 (192.168.0.99)` – domain-joined client
  * `AdamPC (192.168.0.19)` – Elastic Stack node

* **Elastic Stack**

  * Used for **logging, monitoring, and security analytics**
  * Collects data from:
    * internal systems by using Elastic Agents Fleet: AD DC, EDR host
    * collects logs from NGFW and normalizes them to ECS using Logstash
    * Elastic Agent on DMZ Web Server.
  * Fires alerts should events violate security rules.

All internal devices communicate freely within the zone, with VPN users, DMZ Server and have inspected external (e.g. Internet) traffic. They rely on AD for identity services.

---

## 2. Internal Edge Routing – Juniper NetScreen 5GT

The **Juniper NetScreen 5GT** acts as an **internal edge router/firewall**, separating the Internal network from the NGFW.

* Internal interface: `192.168.0.1`
* Transit interface toward NGFW: `10.0.0.2/24`

Its key role is to:

* Route internal traffic
* Participate in a **site-to-site IPSec VPN** with the NG Firewall for Internal ↔ DMZ traffic **only**.
* This device utilizes a **policy-based IPSec tunnel**.

---

## 3. NG Firewall (PA-220) – Security Enforcement Point

The **Next-Generation Firewall** is the **central security control point** in the lab.

Interfaces and zones:

* **Transit/Internal VPN side:** `10.0.0.1/24`
* **DMZ:** `10.10.37.1/24`
* **External:** `172.16.0.49/24`
* **VPN zone:** `10.10.52.0/24` (Provided via GlobalProtect)

### IPSec Site-to-Site VPN

* Tunnel between **Juniper NetScreen ↔ NG Firewall**
* **Strictly limited to DMZ ↔ Internal traffic**
* External/Internet/GP-VPN-Users-bound traffic is explicitly excluded from the tunnel

This device utilizes a **route-based IPSec tunnel with security policy enforcement**. (utilizes Proxy ID)

---

## 4. DMZ Network (10.10.37.0/24)

The **DMZ zone** hosts exposed services:

* **Web Server (10.10.37.45)**

Key characteristics:

* Accessible from the Internal network **only via the IPSec tunnel**
* Accessible from the Internet **and fully inspected**
* Accessible freely for GP VPN Users.
---

## 5. Remote Access VPN (GlobalProtect)

Remote users connect using **GlobalProtect VPN**, terminating on the NG Firewall.

* VPN address pool: `10.10.52.0/24`
* Users originate from the **External network (172.16.0.0/24)**
* They can freely access Internal Resources and DMZ.

---

## 6. External Network & Internet Access

* **Router ISP (172.16.0.1)** provides upstream Internet access
* External users and VPN clients originate here
* External users can only access DMZ
* Traffic from the Internet to DMZ is fully inspected by the NG Firewall
  

---


![homelab_inreallife_photo](https://github.com/user-attachments/assets/7f9bd5e1-687e-4f28-be21-d5e0abb3afc3)


