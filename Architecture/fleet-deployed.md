# Fleet & Elastic Agent Management

This document describes how **Elastic Fleet** is deployed and used in my lab environment. Fleet provides **centralized lifecycle management of Elastic Agents**, enabling consistent configuration, secure communication, and real-time visibility across endpoints.

All Fleet-related communication is secured using **TLS**, with certificates issued by the internal **Active Directory Certificate Services (AD CS)**, ensuring trust consistency across the entire domain.

<img width="417" height="350" alt="fleet-deployed-diagram" src="https://github.com/user-attachments/assets/fe127cf7-068c-4619-b29e-b77d1779a2b6" />

---

## Contents

1. Fleet Overview
2. Fleet Server Configuration
3. Certificate & Trust Model
4. Elastic Agents
5. Log Ingestion
6. Alert Detection

---

## 1. Fleet Overview

The following Fleet components are deployed in the lab:

* **Fleet Server**
* **Three Elastic Agents** (including the Fleet Server host)
* **Kibana Fleet UI**
* **Elasticsearch backend**

Fleet is used to:

* Enroll and centrally manage Elastic Agents
* Apply and maintain agent policies
* Monitor agent health, status, and activity
* Provide a foundation for endpoint detection and response (EDR)

This setup reflects a **realistic enterprise Fleet deployment**, with separation of roles and minimal privileges applied where appropriate.

---

## 2. Fleet Server Configuration

**Fleet Server Host**

* IP address: `192.168.0.19/24`
* This machine also hosts **Elasticsearch** and **Kibana**

**Fleet Server URL**

* `https://fleet-server-01.lab.local:8220`
* Used by Elastic Agents for enrollment and ongoing communication

**DNS Configuration**

* Hostname: `fleet-server-01.lab.local`
* Registered in **Active Directory DNS**
* Resolves internally to `192.168.0.19`

<img width="680" height="267" alt="image" src="https://github.com/user-attachments/assets/ff585375-837f-425c-ad14-1a5d213d3b82" />

This approach ensures name-based TLS validation and mirrors production best practices.

---

## 3. Certificate & Trust Model (Fleet)

Fleet Server communication is secured using internally issued certificates:

* Fleet Server private key and CSR generated locally
* CSR submitted via **AD IIS Web Certificate Enrollment**
* Certificate issued by the **Enterprise Root CA**
* All domain-joined systems inherently trust the Root CA

This guarantees:

* Mutual trust between agents and Fleet Server
* Encrypted communication
* No reliance on external public CAs

**Custom certificate and private key configuration:**

<img width="441" height="524" alt="image" src="https://github.com/user-attachments/assets/bfe84476-ed8e-4a90-8786-6a1ad63384f2" />

---

## 4. Elastic Agents

### Enrolled Hosts

#### **nilfgard-dc01**

* Role: Active Directory Domain Controller
* IP: `192.168.0.69/24`
* Lightweight integrations only (metrics, authentication logs)

> The agent policy is intentionally minimal to avoid unnecessary load on a critical infrastructure component.

---

#### **Workstation01**

* Role: Domain-joined user workstation
* IP: `192.168.0.99/24`
* Full **Elastic Defend (EDR)** integration enabled

---

#### **AdamPC**

* Role: Fleet Server, Elasticsearch & Kibana host
* IP: `192.168.0.19/24`
* Basic metrics collection only, transformation & parsing of NGFW logs

<img width="798" height="504" alt="image" src="https://github.com/user-attachments/assets/f8babc75-fd01-4c0e-8f12-2622948eb709" />

---

### Agent Policy Examples

**Active Directory Domain Controller Policy:**

Metrics, authentication logs, and essential system telemetry only.

<img width="641" height="224" alt="image" src="https://github.com/user-attachments/assets/3eb6bd40-bf18-4abb-9fc5-54e2e62b9f45" />




**Workstation Policy:**

System metrics combined with full endpoint protection and behavioral detection.

<img width="850" height="246" alt="image" src="https://github.com/user-attachments/assets/729fb7ca-32ce-430f-a79b-04a67565fff3" />

---

## 5. Log Ingestion

All enrolled agents successfully forward logs and metrics to Elasticsearch via Fleet.

* Logs are properly indexed
* Data is visible in Kibana
* Ingestion pipelines function as expected

<img width="644" height="287" alt="image" src="https://github.com/user-attachments/assets/6ad8323b-c3e3-42e4-8f2a-27ada833f570" />

---

## 6. Alert Detection

With detection rules enabled, **security alerts are generated and visible in Kibana**.

Examples:

* Privilege escalation event:

  * Domain user added to the **Domain Admins** group

<img width="449" height="400" alt="image" src="https://github.com/user-attachments/assets/011b6829-4bd7-40af-a9a6-3292ae3a635b" />

* Malware-related activity:

  * User `jason.smith` downloaded and attempted to unzip a malicious archive

<img width="619" height="548" alt="image" src="https://github.com/user-attachments/assets/8f926fa4-f991-4baa-9a5a-7114b09f4837" />

These examples demonstrate **end-to-end visibility**, from agent telemetry to actionable security alerts.

---

> **Note:**
> Elastic also ingests logs from the Palo Alto NGFW and triggers custom detection rules.
> See details here:
> [https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md](https://github.com/Astawowski/HomeLab/blob/main/Architecture/palo-alto-ngfw-config.md)

> **Note:**
> TLS and Active Directory integration for Elasticsearch and Kibana is documented here:
> [https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md](https://github.com/Astawowski/HomeLab/blob/main/Architecture/elasticstack-tls-ad-setup.md)
