## 9. Fleet & Elastic Agent Management

Fleet is configured and operational in the lab environment, providing **centralized management of Elastic Agents** via Kibana.

All Fleet-related communication is secured with **TLS**, using certificates issued by the internal **Active Directory Certificate Services (AD CS)**.

---

## 9.1 Fleet Overview

Fleet components deployed in the lab:

* **Fleet Server**
* **2 Elastic Agents (including Fleet Server)**
* **Kibana Fleet UI**
* **Elasticsearch backend**

Fleet is used to:
* Enroll and manage Elastic Agents
* Apply agent policies centrally
* Monitor agent health and activity

---

## 9.2 Fleet Server Configuration

* **Running machine**
  * `192.168.0.19/24`
  * This is the same machine where Elasticsearch and Kibana work.

* **Fleet Server URL**
  * `https://fleet-server-01.lab.local:8220`
  * This is the address agents use to enroll.

* **DNS**
  * `fleet-server-01.lab.local`
  * Added to AD DNS so it resolves to `192.168.0.19/24`
<img width="1359" height="533" alt="image" src="https://github.com/user-attachments/assets/ff585375-837f-425c-ad14-1a5d213d3b82" />


---

## 9.3 Certificate & Trust Model (Fleet)

* Fleet Server private key and CSR generated locally
* CSR signed via **AD IIS Web Certificate Enrollment**
* Certificate issued by **Enterprise Root CA**
* All domain systems trust the Root CA

---

## 9.4 Elastic Agents

### Enrolled Agents

* **nilfgard-dc01**
  * Role: Active Directory Domain Controller
  * IP: `192.168.0.69/24`

* **AdamPC**
  * Role: Fleet Server host
  * IP: `192.168.0.19/24`
<img width="1521" height="553" alt="image" src="https://github.com/user-attachments/assets/d7b40c81-b176-4ee4-b902-3af0d008cc6a" />


### Custom Certificates set
<img width="918" height="1096" alt="image" src="https://github.com/user-attachments/assets/e3c58bf6-38fe-4c25-8006-6d6ca5020c91" />


### AD DC Agent Policy example
<img width="1282" height="447" alt="image" src="https://github.com/user-attachments/assets/3eb6bd40-bf18-4abb-9fc5-54e2e62b9f45" />

---

## 9.4 Logs incoming
  * Now logs are properly incoming into elasticsearch.
<img width="1287" height="574" alt="image" src="https://github.com/user-attachments/assets/6ad8323b-c3e3-42e4-8f2a-27ada833f570" />


---
