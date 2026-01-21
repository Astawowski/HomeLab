
# Elastic Stack – Lab Environment Documentation

## 1. Environment Overview

* **Elasticsearch**

  * URL: `https://elastic.lab.local:9200`
  * IP: `192.168.0.19/24`

* **Kibana**

  * URL: `https://kibana.lab.local:5601`
  * IP: `192.168.0.19/24`

> Elasticsearch and Kibana are running on the **same machine**.
> Both DNS names resolve to `192.168.0.19`.

---

## 2. Active Directory Infrastructure

* **Domain Controller**: `DC-01`
* **Hostname**: `nilfgard-dc01.nilfgard.forest`
* **IP Address**: `192.168.0.69/24`
* **Domain**: `nilfgard.forest`

### Services provided by DC-01

* Active Directory Domain Services
* Active Directory DNS

  * Resolves:

    * `*.nilfgard.forest` 
    * `elastic.lab.local`
    * `kibana.lab.local`
* **Active Directory Certificate Services (AD CS)**

  * Enterprise Root CA
  * Web Certificate Enrollment
  * Used to sign CSRs for:

    * Elasticsearch
    * Kibana

---

## 3. Certificate Trust Model

All systems trust the **self-signed Enterprise Root CA**:

```
Subject: CN = NILFGARD-DC01-CA
DC = nilfgard
DC = forest
```

### Certificates

* CSRs and private keys generated for:

  * Elasticsearch
  * Kibana
* CSRs signed via **DC-01 Web Certificate Enrollment Service**
* Resulting certificates deployed on respective services

---

## 4. Elasticsearch JVM Memory Configuration

Elasticsearch heap size set to **4 GB**.

**File:**

```
C:\My_Elastic_Stack\elasticsearch-9.2.4-windows-x86_64\
elasticsearch-9.2.4\config\jvm.options.d\heap.options
```

**Configuration:**

```text
-Xms4g
-Xmx4g
```

---

## 5. TLS Configuration Overview

### Fully Encrypted Connections

* Kibana ↔ Elasticsearch
* Browser ↔ Kibana
* Elasticsearch Node ↔ Elasticsearch Node
  *(not used – single-node environment)*

---

## 6. Elasticsearch Configuration

### File: `elasticsearch.yml`

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

---
## 7. Active Directory (LDAPS) Authentication – License Limitation

> **Important note**
> LDAPS authentication is **configured correctly** but is **not functional** because
> **Elastic Free Self-Managed license does NOT allow LDAP / LDAPS authentication**.

As a result:

* The Active Directory realm is **loaded but ignored**
* Users **cannot authenticate** via AD/LDAPS
* Role mappings **can be created** but **will never be evaluated** on this license

---

## 7.1 Configured Active Directory Realm (Blocked by License)

The following Active Directory realm is configured in `elasticsearch.yml`:

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

* Enable authentication against **Active Directory**
* Use **LDAPS (TCP/636)** with a trusted Enterprise Root CA
* Authenticate users from domain `nilfgard.forest`
* Use a service account (`svc_elastic_ldap`) for directory binding

---

## 7.2 Confirmation in `elasticsearch.log`

Elasticsearch explicitly reports that the realm is skipped:

```text
[2026-01-20T19:35:58,722][WARN ][o.e.x.s.a.RealmsAuthenticator] [ADAMPC]
Authentication failed using realms [reserved/reserved,file/default_file,native/default_native].

Realms [active_directory/nilfgard_ad] were skipped because they are not permitted on the current license
```
---

## 7.3 Role Mapping Configuration (Created via Security API)

Even though LDAPS authentication is blocked, **role mappings can still be created** using the Security API.

### Mapping definition

* **Active Directory group:** `soc-analysts`
* **Elastic role:** `editor`
* **Realm name:** `nilfgard_ad`

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
## 7.4 Active Directory User Example

Below is a real example of a domain user who belongs to the mapped AD group.

### Command executed on Domain Controller

```powershell
Get-ADUser -Filter "UserPrincipalName -eq 'jason.smith@nilfgard.forest'" -Properties MemberOf
```

### Output

```text
DistinguishedName : CN=Jason Smith,CN=Users,DC=nilfgard,DC=forest
Enabled           : True
GivenName         : Jason
MemberOf          : {CN=soc-analysts,CN=Users,DC=nilfgard,DC=forest}
Name              : Jason Smith
ObjectClass       : user
ObjectGUID        : f2e13d82-664f-45a0-86ff-542b51c3d6d6
SamAccountName    : jason.smith
SID               : S-1-5-21-642565244-3206084399-256025613-1104
Surname           : Smith
UserPrincipalName : jason.smith@nilfgard.forest
```

---

## 7.5 Summary

✔ Active Directory realm is **correctly configured**

✔ TLS and certificate trust are **working properly**

✔ Role mapping **exists and is valid**

❌ Authentication via LDAPS is **blocked by license**

> **Conclusion:**
> The limitation is **purely licensing-related**.
> With a **Platinum / Enterprise** license, this setup would work **without any configuration changes**.

---

## 8. Kibana Configuration

### File: `kibana.yml`

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
```
