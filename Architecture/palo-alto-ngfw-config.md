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
* **Applications & Threats:** Updated to **latest available App-ID content**
* Apropriate certificate set for Web GUI Management.

<img width="250" height="180" alt="image" src="https://github.com/user-attachments/assets/2936ee48-5289-4158-b1e9-d1f5620c7991" />
<img width="502" height="204" alt="image" src="https://github.com/user-attachments/assets/33e950ae-ad9c-462f-b5ea-432bfbdc0659" />


### DNS Configuration:

* **Primary DNS:** `192.168.0.69` (AD DC)
* **Domain suffix:** `lab.local`

### NTP Configuration:

* **NTP Server:** `time.google.com`
* **Time zone:** `GMT+1`
* Accurate time is required for:
 
<img width="382" height="172" alt="image" src="https://github.com/user-attachments/assets/7cbc3299-07bf-4f87-937b-cad1e97a6d66" />

### Certificates & TLS:

* **Imported certificates:**

  * Internal Root CA (from AD CS)
  * Firewall Web management certificate
  * GlobalProtect Portal & Gateway certificate
  * SSL Decryption Forward Proxy Trust & UnTrust certificate
  * DMZ Webserver certificate for SSL Inbound Inspection
    
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
    * PaloAlto Networks Services

---

## 2. Interfaces, Management Profiles, and Zones

Physical and logical interfaces are configured to enforce **clear separation of trust levels**.

### Interfaces:

<img width="391" height="208" alt="image" src="https://github.com/user-attachments/assets/271b00c2-f188-4bec-8b69-9839af2e9be8" />
<img width="498" height="128" alt="image" src="https://github.com/user-attachments/assets/fee6a0ef-081a-413f-96cf-2cef94c4afe7" />

### Zones:

<img width="381" height="158" alt="image" src="https://github.com/user-attachments/assets/11bcd31e-5729-447f-9fbb-fbad1cd59f8d" />

### Management Profiles:

* Applied only on Internal interface
* Allow:
  * Ping (ICMP)
  * HTTPS Web Management
  * SSH
* No management exposed on Untrust or any other zone

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
  * This route is used for Internal-bound traffic but only when it doesn't originate from DMZ Zone. If it does, PBF rule overrides it.

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
* **Local ID/Remote ID:** Not used due to compatibility issues with peer

<img width="280" height="309" alt="image" src="https://github.com/user-attachments/assets/9314fa28-e69e-43ac-846e-e46b33a0a6c2" />


### IPsec Tunnel:

* **Tunnel interface:** `tunnel.1`
* **IKE Gateway:** `IKE_To_NetScreen`
* **Local Proxy ID:** `10.10.37.0/24` (DMZ Zone)
* **Remote Proxy ID:** `192.168.0.0/24` (Internal)
* **Crypto:** AES-128 / SHA-1 / DH Group 2

<img width="542" height="116" alt="image" src="https://github.com/user-attachments/assets/094b51e7-0e74-4387-a2cf-66c2fefee6fe" />

The tunnel encrypts traffic **only when explicitly matched by PBF policy**.

---

## 5. Policy-Based Forwarding (PBF)

Policy-Based Forwarding is used to **force Internal-bound traffic originating from DMZ Zone into the IPsec tunnel**, regardless of routing table.

### PBF rule logic:

* **Source:** `DMZ Zone`
* **Destination:** `192.168.0.0/24`
* **Action:** Forward
* **Egress interface:** `tunnel.1`

<img width="621" height="95" alt="image" src="https://github.com/user-attachments/assets/22a1f4cd-4e61-446e-88a1-7f778049a85b" />

---

## 6. Source NAT (sNAT) Policy

Source NAT is configured for Internet-bound traffic. ISP Router does sNAT too, but here it is done so that ISP Router can route it back to NGFW. 
There is no sNAT for DMZ because: 
 * a) DMZ Web Server do not initiate traffic to Internet,
 * b) Internet-bound returning traffic goes through DNAT "undo".

### NAT policy:

* **From zone:** Internal
* **To zone:** External
* **Translation:** Dynamic IP and Port
* **Translated source address:** Untrust interface IP

<img width="662" height="117" alt="image" src="https://github.com/user-attachments/assets/4c13336e-4f64-44ec-a4c2-5ffb515f9306" />

---

## 7. Destination NAT (DNAT) Policy

Destination NAT is configures so as to allow External users to access DMZ Web Server. In normal scenario, ISP Router would also perform DNAT from its Public IP.

* **From zone:** `External`
* **To zone:** `External`
* **Translation type:** `Static IP-IP mapping`
* **Translating destination IP:** Untrust interface IP -> DMZ Web Server IP:HTTPS

<img width="692" height="74" alt="image" src="https://github.com/user-attachments/assets/84f122f9-7a37-4cd7-af2d-c483d0c96cc8" />

---

## 8. LDAPS Authentication Profile

The firewall integrates with Active Directory using **LDAPS** for secure authentication.

### LDAPS Server profile:

* **Server:** `nilfgard-dc01.nilfgard.forest` (AD DC)
* **Port:** `636`
* **Certificate validation:** Enabled
* **Bind account:** `palo-ngfw-srv` dedicated service account 

<img width="601" height="270" alt="image" src="https://github.com/user-attachments/assets/196dc8ca-aadd-4ba8-b3f6-35085880154e" />

### LDAPS Authentication Profile

* Utilizes above LDAPS Server profile, used for GlobalProtect authentication of remote users and user-group mapping.

<img width="442" height="215" alt="image" src="https://github.com/user-attachments/assets/348ff708-9684-47db-937f-53c5fb0b1ac7" />

---

## 9. User-ID Agent (Windows-based)

* A **Windows User-ID Agent** is deployed on a domain controller.
* It uses previously mentioned `palo-ngfw-srv` service account with special priviledges to bind to AD.

### Function:

* Collects IP ↔ username mappings from Event logs and Domain logons and sends them to PA-220.

<img width="551" height="288" alt="image" src="https://github.com/user-attachments/assets/19f8502e-fba2-4ad4-a29d-ca368f57622a" />
<img width="803" height="118" alt="image" src="https://github.com/user-attachments/assets/f21d1715-a61e-484f-8712-a3fbb901d204" />

This allows:

* User-based security policies
* Identity-aware logging
* Correlation in SIEM

---

## 10. User Group Mapping

The firewall retrieves **Active Directory group membership** using previously mentioned LDAPS Server profile.

<img width="574" height="105" alt="image" src="https://github.com/user-attachments/assets/6e1b2f96-11a4-4b7e-b822-b13da9d0f10a" />

Policies can now reference **groups instead of IPs/Users**.

---

## 11. GlobalProtect Portal & Gateway

* Remote access VPN is provided using **GlobalProtect**.
* **Both GlobalProtect Portal & Gateway**:
  * are deployed on the same NGFW,
  * are deployed on the same `eth.1/2` (external zone) interface and are available at its IP (`172.16.0.49`),
  * use described before LDAPS Authentication profile to authenticate users,
  * use the same certificate issued by Enterprise Root CA.

### GlobalProtect Portal:

<img width="358" height="215" alt="image" src="https://github.com/user-attachments/assets/6bd3730d-5921-47ea-b54f-21c18e9676e6" />

<img width="603" height="271" alt="image" src="https://github.com/user-attachments/assets/0e38be16-803c-4c27-b5f9-69c1280f4501" />

<img width="503" height="153" alt="image" src="https://github.com/user-attachments/assets/293e5ec7-b3ba-4330-a67a-85016c3392c2" />




### GlobalProtect Gateway:
  * IPSec Tunnels traffic via tunnel.2 interface
  * Assigns users with VPN Zone IP (10.10.52.0/24)

<img width="379" height="189" alt="image" src="https://github.com/user-attachments/assets/05410c43-3de7-4f83-9714-89a89d095905" />
<img width="650" height="269" alt="image" src="https://github.com/user-attachments/assets/a9afba46-4c5c-43e8-90a6-377c585757fb" />
<img width="270" height="150" alt="image" src="https://github.com/user-attachments/assets/46622ce1-d2d2-4166-ad44-609d9a724565" />
<img width="672" height="181" alt="image" src="https://github.com/user-attachments/assets/726de113-884e-4fcf-8f8c-0f07cdf90254" />

* Also sets corporate DNS (AD DC).

Provides **secure remote trusted access from outside**.

---

## 12. External Dynamic C2 IPs List

An External Dynamic List (EDL) is configured to get up-to-date list of command-and-control servers.

<img width="473" height="232" alt="image" src="https://github.com/user-attachments/assets/df031995-43ac-4e42-a4b5-8fdbd4d8fc0c" />

Enables fast blocking of:

* Malware callbacks
* Known attacker infrastructure

---

## 13. Security Policy Rules

This NG Firewall enforces security policies described in main document [README.md](https://github.com/Astawowski/HomeLab)

<img width="897" height="466" alt="image" src="https://github.com/user-attachments/assets/48bc4646-c446-4f9f-89b9-b4e78a6a5f58" />

---

## 13. SSL Decryption Policies

SSL/TLS decryption is enabled for visibility.

### Decryption types:

* **SSL Forward Proxy:**
* For outbound internet (HTTPS) traffic
* Only for High risk Internal users `Jason`
* Exclusions:

  * Banking
  * Health services
  * Certificate-pinned apps
  * ... many other sites defined in Decryption Exclusion List

* Using Root CA issued Forward Trust certificate and self-issued Forward UnTrust certificate.
* Enterprise Root CA deployed to endpoints.

* **SSL Inbound Inspection:**
* For incoming HTTPS traffic from Internet to DMZ Web Server.
* Allows us to best protect our exposed Web Server.
* For this to be possible, Web Server certificate has been imported to NGFW.

<img width="780" height="99" alt="image" src="https://github.com/user-attachments/assets/7579a60c-f143-44d2-a685-7187352fc34f" />

---

## 14. Log Forwarding to Elastic SIEM

Alarming Logs that were generated when matched e.g. Security Policy Rule preventing C2 communication are forwarded to **Elastic SIEM** via Syslog.

<img width="815" height="158" alt="image" src="https://github.com/user-attachments/assets/a5fce4a2-d966-4285-94b0-98f1fda33d2e" />
<img width="500" height="147" alt="image" src="https://github.com/user-attachments/assets/98f83f71-7d29-4248-8ab8-34aaba41630f" />
<img width="194" height="278" alt="image" src="https://github.com/user-attachments/assets/b5d598cf-7eb7-40ef-a0b2-a9bc6d3c1f1a" />


---

## Resulting State

* PA-220 acts as **single security brain**
* Legacy devices provide connectivity only
* Traffic is segmented, encrypted, inspected, and logged
* Identity and user context is fully integrated
* Remote users are securely onboarded
* Full visibility is available in Elastic SIEM
