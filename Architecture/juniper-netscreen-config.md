**This document describes how the **Juniper Networks NetScreen 5GT** firewall is configured and utilized in the lab environment.**

---

# Juniper Networks NetScreen 5GT – Configuration Overview

The device is deployed as a **perimeter security gateway**, providing basic network segmentation and **site-to-site IPsec VPN connectivity** using AutoKey IKE.
Although in my environment the device is directly connected to the NGFW, an IPsec site-to-site VPN tunnel was intentionally deployed to enforce confidentiality and perfect secrecy, simulating a scenario in which communication passes through multiple untrusted intermediate devices.

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

* **Primary DNS:** `192.168.0.69` *
* **Secondary DNS:** `8.8.8.8`
* **Domain suffix:** `lab.local`

DNS resolution is required for:

* NTP synchronization
* FQDN-based VPN peers
* Management and troubleshooting

---

## 3. NTP Configuration

Network Time Protocol (NTP) is configured to ensure **accurate system time**, which is critical for:

* VPN tunnel establishment
* IKE phase 1/2 lifetimes
* Log correlation and troubleshooting

### Example NTP settings:

* **NTP Server:** `192.168.0.20`
* **Time zone:** `UTC+1`
* **Sync interval:** default

---

## 4. Interface Configuration

Two physical interfaces are configured to separate trusted internal traffic from untrusted external traffic.

### 4.1 Trust Interface

* **Interface:** `ethernet0/0`
* **Zone:** `Trust`
* **IP Address:** `192.168.10.1/24`
* **Purpose:** Internal LAN connectivity

### 4.2 Untrust Interface

* **Interface:** `ethernet0/1`
* **Zone:** `Untrust`
* **IP Address:** `203.0.113.10/29`
* **Purpose:** Internet / WAN connectivity

---

## 5. Destination Static Routing

Static routes are configured to define how traffic is forwarded outside the local networks.

### Example static routes:

* **Default route:**

  * Destination: `0.0.0.0/0`
  * Gateway: `203.0.113.1`
* **Remote VPN network:**

  * Destination: `10.50.0.0/24`
  * Gateway: `tunnel.1`

These routes ensure:

* Internet-bound traffic exits via Untrust
* VPN traffic is forwarded into the correct tunnel interface

---

## 6. AutoKey IKE Gateway Configuration

An AutoKey IKE gateway is configured to establish **Phase 1 (IKE)** parameters for the IPsec VPN.

### Example IKE Gateway settings:

* **Gateway Name:** `IKE-GW-LAB`
* **Outgoing Interface:** `Untrust`
* **Remote Gateway IP:** `198.51.100.25`
* **Authentication Method:** Pre-Shared Key
* **Pre-Shared Key:** `VeryStrongSharedKey123!`
* **Encryption:** AES-256
* **Authentication:** SHA-256
* **DH Group:** Group 14
* **Lifetime:** 28800 seconds

---

## 7. VPN Tunnel Configuration

An IPsec VPN tunnel is created and bound to the AutoKey IKE gateway.

### Example VPN Tunnel settings:

* **Tunnel Name:** `VPN-LAB-01`
* **IKE Gateway:** `IKE-GW-LAB`
* **Protocol:** ESP
* **Encryption:** AES-256
* **Authentication:** SHA-256
* **Perfect Forward Secrecy:** Enabled (Group 14)
* **Lifetime:** 3600 seconds

The tunnel provides **encrypted site-to-site connectivity** between the local lab network and the remote network.

---

## 8. VPN Tunnel Policy

A security policy is configured to allow traffic between the internal network and the remote VPN network through the tunnel.

### Example VPN policy:

* **From Zone:** `Trust`
* **To Zone:** `Untrust`
* **Source Address:** `192.168.10.0/24`
* **Destination Address:** `10.50.0.0/24`
* **Service:** `ANY`
* **Action:** `Permit`
* **VPN Tunnel:** `VPN-LAB-01`
* **Logging:** Enabled

This policy ensures:

* Only defined networks can use the VPN
* Traffic is explicitly permitted and logged
* The tunnel is used automatically for matching traffic

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

If you want, next we can:

* 🔧 add **actual NetScreen CLI commands** for each section
* 📐 redraw this as a **clean architecture diagram** (same style as your Fleet one)
* 🔐 extend it with **NAT-Traversal, DPD, or backup VPN peers**
