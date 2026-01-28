**This document describes how the **Juniper Networks NetScreen 5GT** firewall is configured and utilized in the lab environment.**

---

# Juniper Networks NetScreen 5GT – Configuration Overview

The device is deployed as a **perimeter security gateway**, providing basic network segmentation and **site-to-site IPsec VPN connectivity** using AutoKey IKE.
Although in my environment the device is directly connected to the NGFW, an IPsec site-to-site VPN tunnel was intentionally deployed to enforce confidentiality and perfect secrecy when connecting to DMZ Zone, simulating a scenario in which communication passes through multiple untrusted intermediate devices.

---

## 1. Base System Configuration

The NetScreen 5GT has been configured with essential system services required for stable and predictable operation in the lab.

### Configured system parameters:

* Hostname: `nilfgard-netscreen`
* Domain: `lab.local`
* Management access: `HTTP` and `Telnet`
* Admin account `root` secured with strong password

---

## 2. DNS Configuration

DNS is configured to allow the netscreen to resolve internal and external hostnames (e.g. NTP servers, VPN peers using FQDN).

### DNS settings:

* **Primary DNS:** `192.168.0.69` (domain controller)
* **Secondary DNS:** `8.8.8.8`
* **Domain suffix:** `lab.local`
  
DNS resolution is required for:

* NTP synchronization
* FQDN-based VPN peers
* Management and troubleshooting

<img width="374" height="131" alt="image" src="https://github.com/user-attachments/assets/adfc3aba-6cba-4f04-b5c3-d129ab029cad" />

---

## 3. NTP Configuration

Network Time Protocol (NTP) is configured to ensure **accurate system time**, which is critical for:

* VPN tunnel establishment
* IKE phase 1/2 lifetimes
* Log correlation and troubleshooting

### NTP settings:

* **NTP Server:** `time.google.com`
* **Time zone:** `GMT+1`
  
<img width="411" height="218" alt="image" src="https://github.com/user-attachments/assets/93a7dca5-38cc-4099-bd88-8d02f4ccd483" />

---

## 4. Interface Configuration

Two physical interfaces are configured to separate trusted internal traffic from untrusted external traffic.

### 4.1 Trust Interface

* **Interface:** `trust.1`
* **Zone:** `Trust`
* **IP Address:** `192.168.0.1/24`
* **Purpose:** Internal LAN connectivity

### 4.2 Untrust Interface

* **Interface:** `untrust`
* **Zone:** `Untrust`
* **IP Address:** `10.0.0.2/24`
* **Purpose:** Internet / WAN connectivity / Connected to PaloAlto NGFW

<img width="436" height="137" alt="image" src="https://github.com/user-attachments/assets/d2b4961b-6dd0-40ae-bc9c-046747eef03c" />


---

## 5. Destination Static Routing

Static routes are configured to define how traffic is forwarded outside the local networks.

### Static routes:

* **Default route:**
  * Destination: `0.0.0.0/0`
  * Gateway: `10.0.0.1` (points to PaloAlto NGFW)
    
These route ensure:
* Internet-bound traffic exits via Untrust and is directed through VPN tunnel via Policy-tunneling.

<img width="442" height="161" alt="image" src="https://github.com/user-attachments/assets/b00aebd5-8951-45d2-a2b7-887b23a5d025" />


---

## 6. AutoKey IKE Gateway Configuration

An AutoKey IKE gateway is configured to establish **Phase 1 (IKE)** parameters for the IPsec VPN.

### Example IKE Gateway settings:

* **Gateway Name:** `Gateway_ToPalo`
* **Remote Gateway FQDN:** `nilfgard-firewall-01.lab.local`
* **Peer ID:**  `nilfgard-firewall-01.lab.local`
* **Mode:** `IKEv1`
* **Authentication:** `Preshared Key` (Netscreen does not support certificates for this purpose)
* **Local ID:**	`10.0.0.2`
* **Outgoing Interface:**	`untrust`
* **Phase 1 Proposal:** `PSK-DH2-AES128-SHA1`
  
<img width="455" height="148" alt="image" src="https://github.com/user-attachments/assets/3001d90b-7ba6-41d5-8027-3f35936eef42" />
<img width="270" height="144" alt="image" src="https://github.com/user-attachments/assets/484e714b-cf01-4a68-9bfe-80cf8b9e0d82" />


---

## 7. VPN Tunnel Configuration

An IPsec VPN tunnel is created and bound to the AutoKey IKE gateway.

### VPN Tunnel settings:

* **Tunnel Name:** `IPsec_Tunnel_ToPalo`
* **IKE Gateway:** `Gateway_ToPalo`
* **Phase 2 Proposal:** `ESP-DH2-AES128-SHA1`

The tunnel provides **encrypted site-to-site connectivity** between the Internal Network and external networks (e.g.: DMZ Zone, External).

<img width="667" height="83" alt="image" src="https://github.com/user-attachments/assets/2fa6976a-d1a8-48fd-86e8-1be10c1f896e" />
<img width="509" height="145" alt="image" src="https://github.com/user-attachments/assets/21929a0d-61f9-4ce7-a494-ef0563165b5e" />

---

## 8. VPN Tunnel Policy

A security policy is configured to tunnel (and allow) traffic between the internal network and external networks (e.g.: DMZ Zone, External).

### VPN policy:

* **From Zone:** `Trust`
* **To Zone:** `Untrust`
* **Service:** `ANY`
* **Action:** `Tunnel`
* **VPN Tunnel:** `IPsec_Tunnel_ToPalo`

<img width="519" height="65" alt="image" src="https://github.com/user-attachments/assets/bf01013c-d1e5-40ee-ae4b-6e49e6e58faa" />
<img width="380" height="207" alt="image" src="https://github.com/user-attachments/assets/3e6f76a7-0025-4174-a24c-0f7e1e39dd72" />


---

## 9. Resulting State

After completing the configuration:

* Interfaces are up and correctly zoned
* DNS and NTP are functional
* Static routing is operational
* IPsec VPN tunnel establishes successfully
* Traffic flows securely between sites
* Logs are generated for audit and troubleshooting

---
