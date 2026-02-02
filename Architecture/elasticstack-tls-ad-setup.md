## Elastic Stack - Lab Environment Documentation

This document describes how the **core Elastic Stack components (Elasticsearch and Kibana)** are deployed, configured, and integrated with **Microsoft Active Directory** in my lab environment.

The focus is on implementing **end-to-end TLS encryption** using certificates issued by **Active Directory Certificate Services (Enterprise Root CA)**, ensuring a realistic enterprise-grade trust model.

<img width="400" height="383" alt="elasticstack-tls-ad-setup-diagram" src="https://github.com/user-attachments/assets/8aa258f7-d354-4893-9ec1-7055cc290b9f" />

---

## Contents

1. Environment Overview
2. Active Directory Infrastructure
3. Certificate Trust Model
4. Elasticsearch JVM Memory Configuration
5. TLS Configuration Overview
6. Elasticsearch Configuration
7. Kibana Configuration
8. Active Directory (LDAPS) Authentication

---

## 1. Environment Overview

### Elasticsearch

* URL: `https://elastic.lab.local:9200`
* IP address: `192.168.0.19/24`

### Kibana

* URL: `https://kibana.lab.local:5601`
* IP address: `192.168.0.19/24`

Elasticsearch and Kibana are running on the **same host**.
Both DNS names resolve to `192.168.0.19`.

---

## 2. Active Directory Infrastructure

### Domain Controller

* Hostname: `DC-01`
* FQDN: `nilfgard-dc01.nilfgard.forest`
* IP address: `192.168.0.69/24`
* Domain: `nilfgard.forest`

### Services provided by DC-01

* Active Directory Domain Services
* Active Directory DNS
  Responsible for resolving:

  * `*.nilfgard.forest`
  * `*.lab.local`, including:

    * `elastic.lab.local`
    * `kibana.lab.local`
* Active Directory Certificate Services (AD CS)

  * Enterprise Root Certification Authority
* Internet Information Services (IIS)

  * Web Certificate Enrollment over HTTPS
  * Used to sign certificate signing requests (CSRs) for:

    * Elasticsearch
    * Kibana

<img width="528" height="281" alt="image" src="https://github.com/user-attachments/assets/b696dd7a-42b4-4f93-ad83-b46cabe84b2b" />
<img width="510" height="407" alt="image" src="https://github.com/user-attachments/assets/11020fe0-89f1-4a2f-8209-05b9e1d8b65b" />

---

## 3. Certificate Trust Model

All systems in the environment trust the **self-signed Enterprise Root CA** issued by Active Directory:

```
Subject: CN = NILFGARD-DC01-CA
DC = nilfgard
DC = forest
```

### Certificates

* Private keys and CSRs were generated for:

  * Elasticsearch
  * Kibana
* CSRs were signed using the **DC-01 Web Certificate Enrollment Service**
* Issued certificates were deployed on the corresponding services

---

## 4. Elasticsearch JVM Memory Configuration

The Elasticsearch JVM heap size is configured to **4 GB**.

### File location

```
C:\My_Elastic_Stack\elasticsearch-9.2.4-windows-x86_64\
elasticsearch-9.2.4\config\jvm.options.d\heap.options
```

### Configuration

```text
-Xms4g
-Xmx4g
```

---

## 5. TLS Configuration Overview

All communication channels are fully encrypted using TLS:

* Kibana to Elasticsearch
* Browser to Kibana
* Elasticsearch node to Elasticsearch node
  (not applicable in this lab due to a single-node deployment)

---

## 6. Elasticsearch Configuration

### Configuration file: `elasticsearch.yml`

TLS is enabled using certificates issued by the AD Enterprise Root CA.

```yaml
# Network configuration
network.host: 192.168.0.19
http.port: 9200
http.host: 192.168.0.19

# Enable security
xpack.security.enabled: true

# ================= HTTP TLS =================
xpack.security.http.ssl:
  enabled: true
  verification_mode: full
  certificate: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.pem
  key: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.key
  certificate_authorities:
    - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem

# ================= Transport TLS =================
xpack.security.transport.ssl:
  enabled: true
  verification_mode: full
  certificate: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.pem
  key: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.key
  certificate_authorities:
    - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem

# Cluster bootstrap (single node)
cluster.initial_master_nodes: ["ADAMPC"]
```

<img width="324" height="285" alt="image" src="https://github.com/user-attachments/assets/506c31a6-c409-4aa8-a5b1-e3165f02e68e" />

---

## 7. Kibana Configuration

### Configuration file: `kibana.yml`

```yaml
server.port: 5601
server.host: "192.168.0.19"
server.name: "kibana.lab.local"

# ================= SSL (Browser ↔ Kibana) =================
server.ssl.enabled: true
server.ssl.certificate: C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/kibana.pem
server.ssl.key: C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/kibana.key

# ================= Kibana ↔ Elasticsearch =================
elasticsearch.hosts:
  - "https://elastic.lab.local:9200"
elasticsearch.ssl.verificationMode: full
elasticsearch.ssl.certificateAuthorities:
  - "C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/nilfgard-root-ca.pem"
elasticsearch.username: kibana_system
elasticsearch.password: "-u1VQ1C9dln1ma1*5ibV"

xpack.encryptedSavedObjects.encryptionKey: f852ed738125aabec389b0a9620a9902
xpack.reporting.encryptionKey: e3cc4a66aaffa67061244442d909b2f1
xpack.security.encryptionKey: 4d16bedd9eeaad675bd1cb4e6356f7ab
```

<img width="331" height="373" alt="image" src="https://github.com/user-attachments/assets/4a1cf237-a219-4071-a708-73f897b26a3a" />

---

## 8. Active Directory (LDAPS) Authentication - License Limitation

### Important note

LDAPS authentication is **configured correctly** but is **not functional** because the **Elastic Free Self-Managed license does not allow LDAP or LDAPS authentication**.

As a result:

* The Active Directory realm is loaded but ignored
* Users cannot authenticate using AD credentials
* Role mappings can be created but are never evaluated

---

## 8.1 Configured Active Directory Realm (Blocked by License)

The following realm is defined in `elasticsearch.yml`:

```yaml
xpack:
  security:
    authc:
      realms:
        active_directory:
          nilfgard_ad:
            order: 0
            domain_name: nilfgard.forest
            url: ldaps://nilfgard-dc01.nilfgard.forest:636
            bind_dn: svc_elastic_ldap@nilfgard.forest
            ssl:
              certificate_authorities:
                - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem
```

### Purpose of this configuration

* Enable authentication against Active Directory
* Use LDAPS (TCP/636) secured by a trusted Enterprise Root CA
* Authenticate users from the `nilfgard.forest` domain
* Use a dedicated service account for directory binding

---

## 8.2 Confirmation in `elasticsearch.log`

Elasticsearch explicitly reports that the Active Directory realm is skipped:

```text
[2026-01-20T19:35:58,722][WARN ][o.e.x.s.a.RealmsAuthenticator] [ADAMPC]
Authentication failed using realms [reserved/reserved,file/default_file,native/default_native].

Realms [active_directory/nilfgard_ad] were skipped because they are not permitted on the current license
```

---

## 8.3 Role Mapping Configuration (Created via Security API)

Even though LDAPS authentication is blocked, role mappings can still be created using the Security API.

### Mapping definition

* Active Directory group: `soc-analysts`
* Elastic role: `editor`
* Realm name: `nilfgard_ad`

### API call (executed via Burp Suite)

```http
POST /_security/role_mapping/ldap-soc-analyst
Content-Type: application/json
Accept: application/json
```

```json
{
  "enabled": true,
  "roles": ["editor"],
  "rules": {
    "all": [
      { "field": { "realm.name": "nilfgard_ad" } },
      { "field": { "groups": "cn=soc-analysts,dc=nilfgard,dc=forest" } }
    ]
  },
  "metadata": {
    "version": 1
  }
}
```

<img width="1365" height="1067" alt="Screenshot 2026-01-21 174158" src="https://github.com/user-attachments/assets/0af345fd-1b79-444e-9f21-120be4a258c8" />
<img width="1870" height="705" alt="Screenshot 2026-01-21 174434" src="https://github.com/user-attachments/assets/71416e73-1b2c-4fc3-b26a-f74d903104ae" />

---

## 8.4 Active Directory User Example

The example below shows a domain user `jason.smith` who is a member of the `soc-analysts` Active Directory group.
With an appropriate Elastic license, this user would be able to authenticate to Kibana using AD credentials and receive the `Editor` role.

<img width="1081" height="287" alt="image" src="https://github.com/user-attachments/assets/a73449b9-ac2c-46c3-bb48-2fb39c7522b1" />

---

## 8.5 Manual User Creation

Due to the license limitation, the only available option for custom user-to-role mapping is **manual user creation in Kibana**.

<img width="310" height="431" alt="image" src="https://github.com/user-attachments/assets/75183215-b421-4d22-9fec-7539f6bfd159" />

---

## 8.6 Summary

* Active Directory realm is correctly configured
* TLS and certificate trust are functioning as expected
* Role mappings are valid and correctly defined
* LDAPS authentication is blocked solely due to licensing limitations

### Conclusion

The limitation is **purely license-related**.
With a **Platinum or Enterprise** Elastic license, this configuration would work **without any changes**.

### Additional note

This environment also uses **Elastic Agent Fleet**.
Configuration details are available here:
[https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md](https://github.com/Astawowski/HomeLab/blob/main/Architecture/fleet-deployed.md)
