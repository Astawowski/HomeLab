**This document descibes how fleet is deployed and utilized in my environment.**

---

## Fleet & Elastic Agent Management

Fleet is configured and operational in the lab environment, providing **centralized management of Elastic Agents** via Kibana.

All Fleet-related communication is secured with **TLS**, using certificates issued by the internal **Active Directory Certificate Services (AD CS)**.

<img width="417" height="350" alt="fleet-deployed-diagram" src="https://github.com/user-attachments/assets/fe127cf7-068c-4619-b29e-b77d1779a2b6" />


---

## Contents:

1. Fleet Overview
2. Fleet Server Configuration
3. Certificate & Trust Model (Fleet)
4. Elastic Agents
5. Logs incoming
6. Alerts detection

   
---

## 1. Fleet Overview

Fleet components deployed in the lab:

* **Fleet Server**
* **3 Elastic Agents (including Fleet Server)**
* **Kibana Fleet UI**
* **Elasticsearch backend**

Fleet is used to:
* Enroll and manage Elastic Agents
* Apply agent policies centrally
* Monitor agent health and activity

---

## 2. Fleet Server Configuration

* **Running machine**
  * `192.168.0.19/24`
  * This is the same machine where Elasticsearch and Kibana work.

* **Fleet Server URL**
  * `https://fleet-server-01.lab.local:8220`
  * This is the address agents use to enroll.

* **DNS**
  * `fleet-server-01.lab.local`
  * Added to AD DNS so it resolves to `192.168.0.19/24`
<img width="680" height="267" alt="image" src="https://github.com/user-attachments/assets/ff585375-837f-425c-ad14-1a5d213d3b82" />


---

## 3. Certificate & Trust Model (Fleet)

* Fleet Server private key and CSR generated locally
* CSR signed via **AD IIS Web Certificate Enrollment**
* Certificate issued by **Enterprise Root CA**
* All domain systems trust the Root CA

* Custom certificates and key set:
<img width="441" height="524" alt="image" src="https://github.com/user-attachments/assets/bfe84476-ed8e-4a90-8786-6a1ad63384f2" />


---

## 4. Elastic Agents

### Enrolled Agents

* **nilfgard-dc01**
  * Role: Active Directory Domain Controller
  * IP: `192.168.0.69/24`
  * Some basic integrations applied, so as not to overwhelm a critical component DC definitely is.
 
* **Workstation01**
  * Role: Ordinary User Workstation (Computer belongs to `nilfgard.forest` domain)
  * IP: `192.168.0.99/24`
  * Full EDR (Elastic Defend integration) deployed.

* **AdamPC**
  * Role: Fleet Server host
  * IP: `192.168.0.19/24`
  * Only basic metrics gathered.
<img width="798" height="504" alt="image" src="https://github.com/user-attachments/assets/f8babc75-fd01-4c0e-8f12-2622948eb709" />


### AD DC Agent Policy example
* Metrics, authentication logs, etc.
<img width="641" height="224" alt="image" src="https://github.com/user-attachments/assets/3eb6bd40-bf18-4abb-9fc5-54e2e62b9f45" />

### Workstation Agent Policy example
* Metrics and full EDR.
<img width="850" height="246" alt="image" src="https://github.com/user-attachments/assets/729fb7ca-32ce-430f-a79b-04a67565fff3" />

---

## 5. Logs incoming
  * Now logs are properly ingested into elasticsearch.
<img width="644" height="287" alt="image" src="https://github.com/user-attachments/assets/6ad8323b-c3e3-42e4-8f2a-27ada833f570" />

---

## 6. Alerts detection
* **When rules have been enabled, alerts are also appearing.**
  * Added domain user to `Domain Admins` group:
<img width="449" height="400" alt="image" src="https://github.com/user-attachments/assets/011b6829-4bd7-40af-a9a6-3292ae3a635b" />

  * User `jason.smith` downloaded and attempted to unzip a malware:
<img width="619" height="548" alt="image" src="https://github.com/user-attachments/assets/8f926fa4-f991-4baa-9a5a-7114b09f4837" />



---

> **Note:**
> Click [here](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md) to see how was `Elasticsearch` and `Kibana` configured with `Active Directory`. 
