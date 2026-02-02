# Palo Alto Networks PA-220 – Configuration Overview

This document describes how the **Palo Alto Networks PA-220 Next-Generation Firewall** is configured and utilized.

The PA-220 firewall operates as the central enforcement point between four security zones:
Internal (192.168.0.0/24), DMZ (10.10.37.0/24), VPN (10.10.52.0/24), and External (172.16.0.0/24).

It integrates with legacy routing (Juniper NetScreen), Active Directory, and Elastic SIEM to simulate a realistic enterprise perimeter and identity-aware security architecture.

<img width="468" height="299" alt="ngfw du" src="https://github.com/user-attachments/assets/293e1cd1-33ae-47f0-bb62-76cbec88a86f" />

---

## Contents:

1. Basic Device Settings
2. Interfaces, Management Profiles, and Zones
3. Virtual Router – Static Routing
4. IPsec Tunnel and IKE Gateway (DMZ ↔ Internal)
5. Policy-Based Forwarding (PBF) for IPsec
6. Source NAT (sNAT) Policy
7. Destination NAT (DNAT) Policy
8. LDAPS Authentication Profile
9. User-ID Agent (Windows-based)
10. User Group Mapping
11. GlobalProtect Portal & Gateway
12. External Dynamic C2 IPs List
13. Security Policy Rules
14. SSL Decryption Policies
15. Log Forwarding to Elastic SIEM

---

## 1. Basic Device Settings

The PA-220 is configured with essential system services to ensure **secure management, accurate timekeeping, application awareness, and cryptographic trust**.

### Configured system parameters:

* **Hostname:** `nilfgard-firewall-01`
* **Domain:** `lab.local`
* **Management interface access:** `HTTPS` & `SSH`
* **Applications & Threats:** Updated to the **latest available App-ID content**
* **Appropriate certificates** configured for Web GUI management

<img width="250" height="180" alt="image" src="https://github.com/user-attachments/assets/2936ee48-5289-4158-b1e9-d1f5620c7991" />
<img width="502" height="204" alt="image" src="https://github.com/user-attachments/assets/33e950ae-ad9c-462f-b5ea-432bfbdc0659" />

### DNS Configuration:

* **Primary DNS:** `192.168.0.69` (AD DC)
* **Domain suffix:** `lab.local`

### NTP Configuration:

* **NTP Server:** `time.google.com`
* **Time zone:** `GMT+1`

<img width="382" height="172" alt="image" src="https://github.com/user-attachments/assets/7cbc3299-07bf-4f87-937b-cad1e97a6d66" />

### Certificates & TLS:

* **Imported certificates:**

  * Internal Root CA (from AD CS)
  * Firewall Web Management certificate
  * GlobalProtect Portal & Gateway certificate
  * SSL Decryption Forward Proxy Trust & Untrust certificates
  * DMZ Web Server certificate for SSL Inbound Inspection

<img width="347" height="205" alt="image" src="https://github.com/user-attachments/assets/1c130c4a-44ba-4e2e-81fe-f36ecbec5bcb" />
<img width="337" height="230" alt="image" src="https://github.com/user-attachments/assets/88cab120-f324-4607-ad3b-a13a064372ab" />

### Service Routes:

* Custom service routes are defined:

  * From `ETH 1/8` (Internal):

    * DNS
    * LDAP
    * Syslog
  * From `ETH 1/2` (External):

    * EDLs
    * NTP
    * Palo Alto Networks services

---

## 2. Interfaces, Management Profiles, and Zones

Physical and logical interfaces are configured to enforce **clear separation of trust levels**.

### Interfaces:

<img width="391" height="208" alt="image" src="https://github.com/user-attachments/assets/271b00c2-f188-4bec-8b69-9839af2e9be8" />
<img width="498" height="128" alt="image" src="https://github.com/user-attachments/assets/fee6a0ef-081a-413f-96cf-2cef94c4afe7" />

### Zones:

<img width="381" height="158" alt="image" src="https://github.com/user-attachments/assets/11bcd31e-5729-447f-9fbb-fbad1cd59f8d" />

### Management Profiles:

* Applied only on the Internal interface
* Allows:

  * Ping (ICMP)
  * HTTPS Web Management
  * SSH
* No management access is exposed on the Untrust or any other zone

---

## 3. Virtual Router – Static Routing

A single Virtual Router is used to manage all routing decisions.

<img width="205" height="143" alt="image" src="https://github.com/user-attachments/assets/d8ba91c3-f052-42fa-ae19-c6a952bd44e3" />

### Static routes:

* **Default route:**

  * Destination: `0.0.0.0/0`
  * Next hop: `172.16.0.1` (ISP router)

* **Internal network route:**

  * Destination: `192.168.0.0/24`
  * Next hop: `10.0.0.2` (NetScreen)
  * This route is used for Internal-bound traffic only when it does not originate from the DMZ zone.
    If it does, the PBF rule overrides it.

<img width="317" height="142" alt="image" src="https://github.com/user-attachments/assets/d66d9465-e212-485c-8d66-f4b23a95f978" />

---

## 4. IPsec Tunnel and IKE Gateway (DMZ ↔ Internal)

An IPsec site-to-site VPN is configured to secure **Internal ↔ DMZ traffic only**.

### IKE Gateway:

* **Name:** `IKE_To_NetScreen`
* **Version:** IKEv1
* **Authentication:** Pre-Shared Key
* **Peer Address:** `10.0.0.2` (NetScreen)
* **Crypto:** PSK / AES-128 / SHA-1 / DH Group 2
* **Local ID / Remote ID:** Not used due to compatibility issues with the peer

<img width="280" height="309" alt="image" src="https://github.com/user-attachments/assets/9314fa28-e69e-43ac-846e-e46b33a0a6c2" />

### IPsec Tunnel:

* **Tunnel interface:** `tunnel.1`
* **IKE Gateway:** `IKE_To_NetScreen`
* **Local Proxy ID:** `10.10.37.0/24` (DMZ zone)
* **Remote Proxy ID:** `192.168.0.0/24` (Internal)
* **Crypto:** AES-128 / SHA-1 / DH Group 2

<img width="542" height="116" alt="image" src="https://github.com/user-attachments/assets/094b51e7-0e74-4387-a2cf-66c2fefee6fe" />

The tunnel encrypts traffic **only when explicitly matched by a PBF policy**.

---

## 5. Policy-Based Forwarding (PBF)

Policy-Based Forwarding is used to **force Internal-bound traffic originating from the DMZ zone into the IPsec tunnel**, regardless of the routing table.

### PBF rule logic:

* **Source:** `DMZ zone`
* **Destination:** `192.168.0.0/24`
* **Action:** Forward
* **Egress interface:** `tunnel.1`

<img width="621" height="95" alt="image" src="https://github.com/user-attachments/assets/22a1f4cd-4e61-446e-88a1-7f778049a85b" />

> **Note:**
> Click [here](https://github.com/Astawowski/HomeLab/blob/main/Architecture/juniper-netscreen-config.md) to see how is the peer (Juniper NetScreen) configured.

---

## 6. Source NAT (sNAT) Policy

Source NAT is configured for Internet-bound traffic. The ISP router also performs sNAT, but it is configured on the NGFW so that the ISP router can correctly route return traffic back to the firewall.

There is no sNAT configured for the DMZ because:

* a) The DMZ Web Server does not initiate outbound Internet traffic
* b) Internet-bound return traffic is handled via DNAT “undo”

### NAT policy:

* **From zone:** Internal
* **To zone:** External
* **Translation:** Dynamic IP and Port
* **Translated source address:** Untrust interface IP

<img width="662" height="117" alt="image" src="https://github.com/user-attachments/assets/4c13336e-4f64-44ec-a4c2-5ffb515f9306" />

---

## 7. Destination NAT (DNAT) Policy

Destination NAT is configured to allow External users to access the DMZ Web Server.

* **From zone:** `External`

* **To zone:** `External`

* **Translation type:** `Static IP-to-IP mapping`

* **Translated destination:** Untrust interface IP `:4433` → DMZ Web Server IP `:443`

* **Note:** A custom service has been defined: `Nilfgard-WebServer` (TCP, destination port: 4433).
  This avoids a conflict with the GlobalProtect Portal, which operates on the same IP address on port 443.

* Due to this DNAT rule, the Web Server is accessible to External users at `172.16.0.49:4433`.

> In a typical production scenario, the Web Server would also be accessible from the Internet, so the ISP router would perform DNAT from its public IP `:443` to `172.16.0.49:4433`.

<img width="648" height="71" alt="image" src="https://github.com/user-attachments/assets/b1b19a6d-ae28-49e4-9a6c-3b5be1503b49" />

---

## 8. LDAPS Authentication Profile

The firewall integrates with Active Directory using **LDAPS** for secure authentication.

### LDAPS Server Profile:

* **Server:** `nilfgard-dc01.nilfgard.forest` (AD DC)
* **Port:** `636`
* **Certificate validation:** Enabled
* **Bind account:** `palo-ngfw-srv` (dedicated service account)

<img width="601" height="270" alt="image" src="https://github.com/user-attachments/assets/196dc8ca-aadd-4ba8-b3f6-35085880154e" />

### LDAPS Authentication Profile

* Utilizes the above LDAPS Server Profile
* Used for GlobalProtect authentication of remote users and for user-group mapping

<img width="442" height="215" alt="image" src="https://github.com/user-attachments/assets/348ff708-9684-47db-937f-53c5fb0b1ac7" />

---

## 9. User-ID Agent (Windows-based)

* A **Windows User-ID Agent** is deployed on a domain controller.
* It uses the previously mentioned `palo-ngfw-srv` service account with dedicated privileges to bind to Active Directory.

### Function:

* Collects IP ↔ username mappings from Windows Event Logs and domain logon events, and sends them to the PA-220.

<img width="551" height="288" alt="image" src="https://github.com/user-attachments/assets/19f8502e-fba2-4ad4-a29d-ca368f57622a" />
<img width="803" height="118" alt="image" src="https://github.com/user-attachments/assets/f21d1715-a61e-484f-8712-a3fbb901d204" />
<img width="684" height="149" alt="image" src="https://github.com/user-attachments/assets/35400e40-a4ca-4dfc-b92f-df01707a04eb" />

This enables:

* User-based security policies
* Identity-aware logging
* Accurate correlation in SIEM

---

## 10. User Group Mapping

The firewall retrieves **Active Directory group membership** using the previously described LDAPS Server Profile.

<img width="574" height="105" alt="image" src="https://github.com/user-attachments/assets/6e1b2f96-11a4-4b7e-b822-b13da9d0f10a" />

<img width="469" height="312" alt="image" src="https://github.com/user-attachments/assets/9d8d25f6-9025-4011-a3a6-f34c4eb48519" />

Security policies can now reference **AD groups instead of individual IP addresses or users**.

---

## 11. GlobalProtect Portal & Gateway

* Remote access VPN is provided using **GlobalProtect**.
* **Both GlobalProtect Portal and Gateway**:

  * are deployed on the same NGFW,
  * are hosted on the same `eth.1/2` (External zone) interface and exposed via its IP address (`172.16.0.49`),
  * use the previously described LDAPS Authentication Profile for user authentication,
  * use the same certificate issued by the Enterprise Root CA.

### GlobalProtect Portal:

<img width="358" height="215" alt="image" src="https://github.com/user-attachments/assets/6bd3730d-5921-47ea-b54f-21c18e9676e6" />

<img width="603" height="271" alt="image" src="https://github.com/user-attachments/assets/0e38be16-803c-4c27-b5f9-69c1280f4501" />

<img width="503" height="153" alt="image" src="https://github.com/user-attachments/assets/293e5ec7-b3ba-4330-a67a-85016c3392c2" />

### GlobalProtect Gateway:

* Terminates IPSec tunnels using the `tunnel.2` interface
* Assigns users IP addresses from the VPN Zone (`10.10.52.0/24`)

<img width="379" height="189" alt="image" src="https://github.com/user-attachments/assets/05410c43-3de7-4f83-9714-89a89d095905" />
<img width="650" height="269" alt="image" src="https://github.com/user-attachments/assets/a9afba46-4c5c-43e8-90a6-377c585757fb" />
<img width="270" height="150" alt="image" src="https://github.com/user-attachments/assets/46622ce1-d2d2-4166-ad44-609d9a724565" />
<img width="672" height="181" alt="image" src="https://github.com/user-attachments/assets/726de113-884e-4fcf-8f8c-0f07cdf90254" />

* Corporate DNS settings (AD DC) are also pushed to connected clients.

This setup provides **secure, trusted remote access from outside the organization**.

<img width="186" height="222" alt="Zrzut ekranu 2026-02-02 183535" src="https://github.com/user-attachments/assets/1b7d1f8c-143c-4bf1-92a1-907a45a616a4" />
<img width="509" height="351" alt="Zrzut ekranu 2026-02-02 183625" src="https://github.com/user-attachments/assets/c1c73564-afa7-4559-b611-5fc94bad2544" />
<img width="667" height="180" alt="image" src="https://github.com/user-attachments/assets/44c13716-8c74-4736-9fca-ae9b113d6072" />

---

## 12. External Dynamic C2 IPs List

An **External Dynamic List (EDL)** is configured to retrieve an up-to-date list of known command-and-control (C2) servers.

<img width="473" height="232" alt="image" src="https://github.com/user-attachments/assets/df031995-43ac-4e42-a4b5-8fdbd4d8fc0c" />

This enables fast and automated blocking of:

* Malware callbacks
* Known attacker infrastructure

---

## 13. Security Policy Rules

The NGFW enforces security policies described in the main project documentation:
[README.md](https://github.com/Astawowski/HomeLab)

The first rule is a custom policy allowing unrestricted and uninspected Internet access for a dedicated AD group.

<img width="901" height="466" alt="image" src="https://github.com/user-attachments/assets/2b1a7ffe-51fb-41df-ba87-51eacc1d3316" />

---

## 14. SSL Decryption Policies

SSL/TLS decryption is enabled to provide deep traffic visibility.

### Decryption types:

* **SSL Forward Proxy:**

  * Used for outbound Internet (HTTPS) traffic
  * Applied only to high-risk Internal users (e.g. `Jason`)
  * Exclusions include:

    * Banking services
    * Healthcare services
    * Certificate-pinned applications
    * Additional sites defined in the Decryption Exclusion List
  * Uses a Root CA–issued Forward Trust certificate and a self-issued Forward Untrust certificate
  * Enterprise Root CA is deployed to endpoints

* **SSL Inbound Inspection:**

  * Used for incoming HTTPS traffic from the Internet to the DMZ Web Server
  * Provides enhanced protection for exposed services
  * Requires importing the Web Server certificate into the NGFW

<img width="796" height="97" alt="image" src="https://github.com/user-attachments/assets/d6f19a6d-7101-4141-8817-8da93a7f6f52" />

<img width="572" height="139" alt="image" src="https://github.com/user-attachments/assets/10d5e599-75ef-4cf6-be8c-1eb0affc2093" />
<img width="582" height="80" alt="image" src="https://github.com/user-attachments/assets/d89feb2e-9220-4d32-8527-82f5a053d69f" />

---

## 15. Log Forwarding to Elastic SIEM

Security-relevant logs—such as those generated by policy rules blocking C2 communication—are forwarded to **Elastic SIEM** via Syslog.

<img width="815" height="158" alt="image" src="https://github.com/user-attachments/assets/a5fce4a2-d966-4285-94b0-98f1fda33d2e" />

<img width="500" height="147" alt="image" src="https://github.com/user-attachments/assets/98f83f71-7d29-4248-8ab8-34aaba41630f" />

<img width="194" height="278" alt="image" src="https://github.com/user-attachments/assets/b5d598cf-7eb7-40ef-a0b2-a9bc6d3c1f1a" />

* On the Elastic side, log collection and transformation are implemented using Fleet (dedicated PaloAlto integration):

<img width="605" height="445" alt="image" src="https://github.com/user-attachments/assets/eb60bc88-a1c0-480c-9aa7-28ef6ab9d2d0" />

* Logs are successfully ingested into SIEM, where custom detection rules generate alerts:

<img width="686" height="150" alt="image" src="https://github.com/user-attachments/assets/66af7613-5f9b-4f8e-8708-7230a3b949a3" />
<img width="890" height="454" alt="image" src="https://github.com/user-attachments/assets/3a5ee810-3ad7-42da-bbc1-1850b233081a" />

---

## Resulting State

The resulting architecture reflects a modern enterprise security design, where a single NGFW acts as the policy decision point, while legacy and edge devices provide transport only.

All traffic paths are explicitly defined, identity-aware, logged, and observable, with encrypted segmentation between trust zones and full SIEM integration for detection and response.
