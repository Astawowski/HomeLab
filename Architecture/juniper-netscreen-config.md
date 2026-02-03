# Juniper Networks NetScreen 5GT - Configuration Overview

This document describes how the **Juniper Networks NetScreen 5GT** firewall is configured and used within my home lab environment. The goal of this setup is to demonstrate **network segmentation**, **site-to-site IPsec VPN connectivity**, and **clear separation of responsibilities** between legacy perimeter devices and a modern NGFW.

<img width="435" height="132" alt="juniper-diagram" src="https://github.com/user-attachments/assets/136b5131-1844-4b37-879d-a775e30db7ff" />

The NetScreen 5GT is deployed as a **perimeter gateway** for the Internal network. It provides:

* Basic network segmentation
* DHCP services for the Internal LAN
* **Site-to-site IPsec VPN connectivity** using AutoKey IKE

Although the device is physically connected directly to a **Palo Alto Networks PA-220 NGFW**, an **IPsec site-to-site VPN tunnel** is intentionally deployed between the two devices.
This design enforces **confidentiality and perfect forward secrecy**, while simulating a real-world scenario where traffic traverses **untrusted or intermediate networks** before reaching the DMZ.

> **Important design decision:**
> The NetScreen 5GT **does not perform traffic inspection, filtering, or security enforcement**.
> All advanced security controls (inspection, policy enforcement, logging) are handled exclusively by the **Palo Alto NGFW**.

---

## Contents

1. Base System Configuration
2. DNS Configuration
3. NTP Configuration
4. Interface Configuration
5. DHCP Server Configuration
6. Destination Static Routing
7. AutoKey IKE Gateway Configuration
8. VPN Tunnel Configuration
9. Security & VPN Tunnel Policies
10. Resulting State

---

## 1. Base System Configuration

The NetScreen 5GT is configured with essential system parameters to ensure **stable, predictable, and manageable operation** within the lab.

### Configured system parameters

* **Hostname:** `nilfgard-netscreen`
* **Domain:** `lab.local`
* **Management access:** `HTTP`, `Telnet`
* **Administrator account:** `root`, secured with a strong password

---

## 2. DNS Configuration

DNS is configured to allow the NetScreen device to resolve both **internal and external hostnames**, which is required for services such as:

* NTP synchronization
* VPN peer resolution using FQDNs

### DNS settings

* **Primary DNS:** `192.168.0.69` (Active Directory Domain Controller)
* **Domain suffix:** `lab.local`

<img width="370" height="82" alt="image" src="https://github.com/user-attachments/assets/2152e68a-dfed-4ebf-9753-2842e917e9ca" />

---

## 3. NTP Configuration

Network Time Protocol (NTP) is configured to maintain **accurate system time**, which is critical for:

* Successful VPN tunnel establishment
* Correct IKE Phase 1 and Phase 2 lifetimes
* Log accuracy and troubleshooting

### NTP settings

* **NTP server:** `time.google.com`
* **Time zone:** `GMT+1`

<img width="411" height="218" alt="image" src="https://github.com/user-attachments/assets/93a7dca5-38cc-4099-bd88-8d02f4ccd483" />

---

## 4. Interface Configuration

Two physical interfaces are configured to clearly separate **trusted internal traffic** from **untrusted external traffic**.

### 4.1 Trust Interface

* **Interface:** `trust.1`
* **Zone:** `Trust`
* **IP address:** `192.168.0.1/24`
* **Purpose:** Internal LAN connectivity

### 4.2 Untrust Interface

* **Interface:** `untrust`
* **Zone:** `Untrust`
* **IP address:** `10.0.0.2/24`
* **Purpose:** Connectivity toward Palo Alto NGFW / DMZ / Internet

<img width="436" height="137" alt="image" src="https://github.com/user-attachments/assets/d2b4961b-6dd0-40ae-bc9c-046747eef03c" />

---

## 5. DHCP Server Configuration

The NetScreen 5GT acts as a **DHCP server** on the Trust interface, providing automatic IP configuration for Internal hosts.
This simplifies endpoint management and ensures consistent network settings.

### DHCP settings
  * **Interface:** `trust` (`192.168.0.1/24`)
  * **Mode:** `Server`
  * **Address pool:** `192.168.0.2 - 192.168.0.15`
  * **Reserved address:**
    * `192.168.0.19` (Elastic Stack host, bound to MAC address)
  * **Default gateway:** `192.168.0.1`
  * **Subnet mask:** `255.255.255.0`
  * **DNS server:** `192.168.0.69` (AD Domain Controller)

<img width="307" height="72" alt="image" src="https://github.com/user-attachments/assets/2b416390-8828-4e38-8f8e-76b7db0638ae" />
<img width="327" height="198" alt="image" src="https://github.com/user-attachments/assets/400db44c-1415-4dda-b470-a5f6320f2093" />
<img width="332" height="47" alt="image" src="https://github.com/user-attachments/assets/e709300c-4e45-4d59-b4f2-99ace4abb751" />

---

## 6. Destination Static Routing

Static routing defines how traffic leaves the Internal network.

### Configured static routes

* **Default route:**

  * Destination: `0.0.0.0/0`
  * Gateway: `10.0.0.1` (Palo Alto NGFW)

This ensures that:

* Internet-bound traffic is forwarded directly to the NGFW
* Only DMZ-bound traffic is subject to IPsec tunneling, based on policy (see Section 9)

<img width="442" height="161" alt="image" src="https://github.com/user-attachments/assets/b00aebd5-8951-45d2-a2b7-887b23a5d025" />

---

## 7. AutoKey IKE Gateway Configuration

An AutoKey IKE gateway is configured to define **IKE Phase 1 parameters** for the site-to-site VPN.

### IKE Gateway settings

* **Gateway name:** `Gateway_ToPalo`
* **Remote gateway FQDN:** `nilfgard-firewall-01.lab.local`
* **Peer ID / Local ID:** Not used (compatibility reasons)
* **Mode:** `IKEv1`
* **Authentication:** `Pre-Shared Key`
* **Outgoing interface:** `untrust`
* **Phase 1 proposal:** `PSK-DH2-AES128-SHA1`

<img width="420" height="141" alt="image" src="https://github.com/user-attachments/assets/d9776035-aded-4f57-bdb2-436cbcc138da" />
<img width="290" height="140" alt="image" src="https://github.com/user-attachments/assets/14f7ff72-3844-4d44-aee7-2ce515df05c4" />

---

## 8. VPN Tunnel Configuration

An IPsec VPN tunnel is created and bound to the AutoKey IKE gateway.

### VPN tunnel settings

* **Tunnel name:** `IPsec_Tunnel_ToPalo`
* **IKE gateway:** `Gateway_ToPalo`
* **Phase 2 proposal:** `ESP-DH2-AES128-SHA1`
* **Local proxy ID:** `192.168.0.0/24`
* **Remote proxy ID:** `10.10.37.0/24`

The tunnel provides **encrypted site-to-site connectivity** **only** between:

* Internal Network
* DMZ Network

<img width="667" height="83" alt="image" src="https://github.com/user-attachments/assets/2fa6976a-d1a8-48fd-86e8-1be10c1f896e" />
<img width="303" height="228" alt="image" src="https://github.com/user-attachments/assets/dcb7babf-e58c-45e5-ba8f-287c0bfc834d" />

---

## 9. Security & VPN Tunnel Policies

Security policies explicitly control which traffic is tunneled and which is forwarded normally.

> **Key principle:**
> Only traffic between the **Internal network and the DMZ** is encrypted via IPsec.
> All other traffic is forwarded in cleartext to the Palo Alto NGFW, where security enforcement takes place.

### VPN policies

* **Policy ID 3** - Internal → DMZ (Tunnel)

  * From zone: `Trust`
  * To zone: `Untrust`
  * Destination: `10.10.37.0/24`
  * Action: `Tunnel`
  * VPN: `IPsec_Tunnel_ToPalo`

* **Policy ID 2** - Internal → Any (Permit)

  * Catch-all for non-DMZ traffic

* **Policy ID 4** - DMZ → Internal (Tunnel)

  * Source: `10.10.37.0/24`
  * Action: `Tunnel`

* **Policy ID 5** - Any → Internal (Permit)

  * Catch-all for non-DMZ inbound traffic

<img width="590" height="173" alt="image" src="https://github.com/user-attachments/assets/0d4f674a-dc25-47df-8903-36178b2eae99" />

> **Note:**
> See the Palo Alto NGFW peer configuration here:
>  [https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md](https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md)

---

## 10. Resulting State

### After completing the configuration

* DNS and NTP operate correctly
* Static routing functions as expected
* IPsec VPN tunnel between Internal ↔ DMZ is established
* Traffic flows securely and predictably
* Logs are generated for troubleshooting and auditing

### Verified traffic flows

**Internal → DMZ** <img width="424" height="158" alt="image" src="https://github.com/user-attachments/assets/575f4a1c-3fb2-4a76-8aa9-e0558ae4c56a" />

**Internal → Any** <img width="440" height="140" alt="image" src="https://github.com/user-attachments/assets/4588f1f4-146b-44d8-948d-74b595e947b9" />

**DMZ → Internal** <img width="403" height="140" alt="image" src="https://github.com/user-attachments/assets/b402d25c-47f8-459c-9d5a-2e1bff771b26" />

**Any → Internal** <img width="382" height="137" alt="image" src="https://github.com/user-attachments/assets/829cb0e9-0e89-4ad3-8568-47b0a8430f40" />
