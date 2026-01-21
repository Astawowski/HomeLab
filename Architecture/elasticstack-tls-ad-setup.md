
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

    * `nilfgard.forest`
    * `elastic.lab.local`
    * `kibana.lab.local`
    * `*.lab.local`
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

> **Note:**
> LDAPS authentication is configured but **not functional** because
> **Elastic Free Self-Managed license does NOT allow LDAP/LDAPS authentication**.

### Configured (but blocked by license)

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

### Confirmation from `elasticsearch.log`

```text
[2026-01-20T19:35:58,722][WARN ][o.e.x.s.a.RealmsAuthenticator] [ADAMPC]
Authentication failed using realms [reserved/reserved,file/default_file,native/default_native].

Realms [active_directory/nilfgard_ad] were skipped because they are not permitted on the current license
```

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
