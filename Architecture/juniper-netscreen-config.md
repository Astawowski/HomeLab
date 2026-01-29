**This document describes how the **Juniper Networks NetScreen 5GT** firewall is configured and utilized in the lab environment.**

--- 

# Juniper Networks NetScreen 5GT – Configuration Overview

<img width="435" height="132" alt="juniper-diagram" src="https://github.com/user-attachments/assets/136b5131-1844-4b37-879d-a775e30db7ff" />

The device is deployed as a **perimeter security gateway**, providing basic network segmentation and **site-to-site IPsec VPN connectivity** using AutoKey IKE. It "hosts" `Internal` network, providing DHCP Server.
Although in my environment the device is directly connected to the NGFW, an IPsec site-to-site VPN tunnel was intentionally deployed to enforce confidentiality and perfect secrecy when connecting to DMZ Zone, simulating a scenario in which communication passes through multiple untrusted intermediate devices.

---

## Contents:

1. Base System Configuration
2. DNS Configuration
3. NTP Configuration
4. Interface Configuration
5. DHCP Server Configuration
6. Destination Static Routing
7. AutoKey IKE Gateway Configuration
8. VPN Tunnel Configuration
9. Security & VPN Tunnel Policy
10. Resulting State

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
* **Purpose:** Internet / DMZ Zone / Connected to PaloAlto NGFW

<img width="436" height="137" alt="image" src="https://github.com/user-attachments/assets/d2b4961b-6dd0-40ae-bc9c-046747eef03c" />


---

### 5. DHCP Server Configuration

Dynamic Host Configuration Protocol (DHCP) is configured on the **trust interface (Internal)** to provide automatic IP addressing and basic network parameters to hosts in the internal network. This ensures consistent connectivity, simplified management, and reduced risk of misconfiguration.

**DHCP settings:**

* **Interface:** `trust` (192.168.0.1/24)
* **DHCP mode:** `Server`
* **Address pool:** `192.168.0.2 – 192.168.0.15`
* **Reserved address:**
  * `192.168.0.19` (Elastic Stack host, bound to MAC address)
* **Default gateway:** `192.168.0.1` (NetScreen itself)
* **Subnet mask:** `255.255.255.0`
* **DNS server:** `192.168.0.69` (AD Domain Controller)


<img width="307" height="72" alt="image" src="https://github.com/user-attachments/assets/2b416390-8828-4e38-8f8e-76b7db0638ae" />
<img width="327" height="198" alt="image" src="https://github.com/user-attachments/assets/400db44c-1415-4dda-b470-a5f6320f2093" />
<img width="332" height="47" alt="image" src="https://github.com/user-attachments/assets/e709300c-4e45-4d59-b4f2-99ace4abb751" />


---

## 6. Destination Static Routing

Static routes are configured to define how traffic is forwarded outside the local networks.

### Static routes:

* **Default route:**
  * Destination: `0.0.0.0/0`
  * Gateway: `10.0.0.1` (points to PaloAlto NGFW)
    
These route ensure:
* Internet-bound and DMZ-bound traffic exits via Untrust and is directed directly to NGFW.
* VPN Tunneling applies only to DMZ-bound traffic, thanks to proper Policies. (More in section 8.)

<img width="442" height="161" alt="image" src="https://github.com/user-attachments/assets/b00aebd5-8951-45d2-a2b7-887b23a5d025" />


---

## 7. AutoKey IKE Gateway Configuration

An AutoKey IKE gateway is configured to establish **Phase 1 (IKE)** parameters for the IPsec VPN.

### IKE Gateway settings:

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

## 8. VPN Tunnel Configuration

An IPsec VPN tunnel is created and bound to the AutoKey IKE gateway.

### VPN Tunnel settings:

* **Tunnel Name:** `IPsec_Tunnel_ToPalo`
* **IKE Gateway:** `Gateway_ToPalo`
* **Phase 2 Proposal:** `ESP-DH2-AES128-SHA1`

The tunnel provides **encrypted site-to-site connectivity** between the Internal Network and DMZ Zone **only**.

<img width="667" height="83" alt="image" src="https://github.com/user-attachments/assets/2fa6976a-d1a8-48fd-86e8-1be10c1f896e" />
<img width="509" height="145" alt="image" src="https://github.com/user-attachments/assets/21929a0d-61f9-4ce7-a494-ef0563165b5e" />

---

## 9. Security & VPN Tunnel Policy

Security Policies are configured to tunnel traffic between the internal network and DMZ Zone **only**.
Traffic from Internal to other destinations bypasses VPN Tunneling.

### VPN policies:

* **ID.2:**
  Design to direct DMZ-bound outgoing traffic through IPSec Tunnel.
  * From Zone: `Trust`
  * To Zone: `Untrust`
  * Destination IP: `10.10.37.0/24` (DMZ Zone)
  * Action: `Tunnel`
  * VPN Tunnel: `IPsec_Tunnel_ToPalo`

* **ID.3:**
  Design to 'catch' every outbound traffic that does not match rule ID.2, that is, is not DMZ-bound.
  * From Zone: `Trust`
  * To Zone: `Untrust`
  * Destination IP: `Any`
  * Action: `Permit`

* **ID.4:**
  Design to accept tunneled traffic from DMZ Zone to Internal Network.
  * From Zone: `Untrust`
  * To Zone: `Trust`
  * Source IP: `10.10.37.0/24` (DMZ Zone)
  * Action: `Tunnel`
  * VPN Tunnel: `IPsec_Tunnel_ToPalo`

* **ID.5:**
  Design to accept not tunneled traffic from External Network to Internal Network. (Palo Alto NGFW performs "Firewalling")
  * From Zone: `Untrust`
  *  To Zone: `Trust`
  * Source IP: `Any`
  * Action: `Permit`

<img width="590" height="173" alt="image" src="https://github.com/user-attachments/assets/0d4f674a-dc25-47df-8903-36178b2eae99" />

---

## 10. Resulting State

After completing the configuration:

* DNS and NTP are functional
* Static routing is operational
* IPsec VPN tunnel for Internal <-> DMZ Zone establishes successfully
* Traffic flows securely between sites
* Logs are generated for audit and troubleshooting


*  Traffic Internal -> DMZ Zone:
<img width="625" height="147" alt="image" src="https://github.com/user-attachments/assets/8e972317-cdfc-41af-9b1e-8560188bc46b" />


*  Traffic Internal -> Internet:
<img width="615" height="151" alt="image" src="https://github.com/user-attachments/assets/a0e116d7-9a9b-471f-93ce-2e98f9e99819" />


*  Traffic DMZ Zone -> Internal:
<img width="614" height="148" alt="image" src="https://github.com/user-attachments/assets/f41c1357-aea9-4c0b-9733-1fd46db42b72" />

---
