# AAP Deployment Types and Custom CA Support

## Quick Overview

```mermaid
graph LR
    A[AAP Deployments] --> B[File-Based<br/>⚠️ QUIC Issue]
    A --> C[Secret-Based<br/>✅ Safe]
    
    B --> B1[RPM-A/B<br/>VM installer]
    B --> B2[CONT-A/B<br/>Container installer]
    
    C --> C1[OCP-A/B<br/>AAP Operator]
    C --> C2[AWX/Hub/EDA<br/>Component Operators]
    
    B1 --> D[mesh-CA.crt file<br/>Can accumulate bundles]
    B2 --> D
    
    C1 --> E[K8s Secret<br/>Single CA only]
    C2 --> E
    
    D --> F[⚠️ Risk: CRYPTO_BUFFER_EXCEEDED]
    E --> G[✅ Protected by architecture]
    
    classDef problem fill:#000000,stroke:#ff0000,stroke-width:3px,color:#ffffff
    classDef safe fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class B,B1,B2,D,F problem
    class C,C1,C2,E,G safe
```

## Deployment Type Comparison and Scope

| Deployment | Installer | CA Format | Custom CA | QUIC Risk | Fix Needed |
|-----------|-----------|-----------|-----------|-----------|------------|
| **RPM-A** (Standard VM) | automation-platform-collection | File | mesh_ca_certfile, custom_ca_cert bundle | ⚠️ YES | ✅ YES |
| **RPM-B** (Enterprise VM) | automation-platform-collection | File | mesh_ca_certfile, custom_ca_cert bundle | ⚠️ YES | ✅ YES |
| **CONT-A** (Standard Container) | aap-containerized-installer | File | custom_ca_cert bundle | ⚠️ YES | ✅ YES |
| **CONT-B** (Enterprise Container) | aap-containerized-installer | File | custom_ca_cert bundle | ⚠️ YES | ✅ YES |
| **OCP-A** (Standard OpenShift) | AAP Operator | K8s Secret | Single CA Secret | ✅ NO | ❌ NO |
| **OCP-B** (Enterprise OpenShift/SaaS) | AAP Operator | K8s Secret | Single CA Secret | ✅ NO | ❌ NO |
| **AWX Operator** (Kubernetes) | AWX Operator | K8s Secret | Single CA Secret | ✅ NO | ❌ NO |

## Repository and Code References

| Deployment | Component | Repository | Code Links |
|-----------|-----------|-----------|------------|
| **RPM-A/B** | VM Installer | [automation-platform-collection](https://github.com/ansible/automation-platform-collection) | [tls_ca.yml](https://github.com/ansible/automation-platform-collection/blob/main/roles/receptor/tasks/tls_ca.yml), [receptor.conf.j2](https://github.com/ansible/automation-platform-collection/blob/main/roles/receptor/templates/receptor.conf.j2) |
| **CONT-A/B** | Container Installer | [aap-containerized-installer](https://github.com/ansible/aap-containerized-installer) | [mesh-CA.crt.j2](https://github.com/ansible/aap-containerized-installer/blob/main/roles/receptor/templates/mesh-CA.crt.j2), [tls.yml:78-94](https://github.com/ansible/aap-containerized-installer/blob/main/roles/receptor/tasks/tls.yml#L78-L94) |
| **OCP-A/B** | AAP Operator | Operator Hub subscription | OpenShift Operator Marketplace |
| **AWX Operator** | K8s Operator | [awx-operator](https://github.com/ansible/awx-operator) | [custom-receptor-certs.md](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/advanced-configuration/custom-receptor-certs.md), [receptor_ca_secret.yaml.j2](https://github.com/ansible/awx-operator/blob/devel/roles/installer/templates/secrets/receptor_ca_secret.yaml.j2) |
| **All** | Core Receptor | [receptor](https://github.com/ansible/receptor) | [netceptor.go:1105-1107](https://github.com/ansible/receptor/blob/devel/pkg/netceptor/netceptor.go#L1105-L1107) |
| **All** | AWX Controller | [awx](https://github.com/ansible/awx) | [receptor.py:785-792](https://github.com/ansible/awx/blob/devel/awx/main/tasks/receptor.py#L785-L792) |

---

## Detailed Architecture Diagrams

### Traditional VM/Bare Metal (RPM-A/B)

```mermaid
graph TD
    A[automation-platform-collection<br/>Ansible installer] --> B{Custom CA Options}
    
    B --> B1[mesh_ca_certfile/keyfile<br/>Customer provides CA + key]
    B --> B2[custom_ca_cert<br/>Customer provides CA bundle]
    B --> B3[Default<br/>Installer generates CA]
    
    B1 --> C1[Installer uses customer CA<br/>to sign all node certs]
    C1 --> D1[mesh-CA.crt:<br/>Single customer CA<br/>~1.5KB]
    
    B2 --> C2[Installer concatenates<br/>internal + custom bundle]
    C2 --> D2[mesh-CA.crt:<br/>Internal + 154 custom CAs<br/>~18KB ⚠️]
    
    B3 --> C3[Installer generates<br/>self-signed CA]
    C3 --> D3[mesh-CA.crt:<br/>Single internal CA<br/>~1.5KB]
    
    D1 --> E[✅ QUIC Compatible]
    D2 --> F[❌ CRYPTO_BUFFER_EXCEEDED]
    D3 --> E
    
    classDef problem fill:#000000,stroke:#ff0000,stroke-width:3px,color:#ffffff
    classDef safe fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class D2,F problem
    class D1,D3,E safe
```

### Containerized/Podman (CONT-A/B)

```mermaid
graph TD
    A[aap-containerized-installer<br/>Podman-based] --> B{Custom CA Options}
    
    B --> B1[custom_ca_cert<br/>CA bundle provided]
    B --> B2[Default<br/>No custom_ca_cert]
    
    B1 --> C1[Template concatenates<br/>internal + custom CAs]
    C1 --> D1[mesh-CA.crt:<br/>Internal + custom bundle<br/>154+ certs, ~18KB ⚠️]
    
    B2 --> C2[Template uses<br/>internal CA only]
    C2 --> D2[mesh-CA.crt:<br/>Single internal CA<br/>~1.5KB]
    
    D1 --> E1[Custom CAs also in:<br/>/etc/pki/ca-trust/source/anchors/]
    E1 --> F[❌ CRYPTO_BUFFER_EXCEEDED<br/>Unnecessary duplication]
    
    D2 --> G[✅ QUIC Compatible]
    
    classDef problem fill:#000000,stroke:#ff0000,stroke-width:3px,color:#ffffff
    classDef safe fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class D1,E1,F problem
    class D2,G safe
```

### Kubernetes/OpenShift Operators (OCP-A/B)

```mermaid
graph TD
    A[AWX/AAP Operator<br/>K8s-based] --> B{Receptor CA Method}
    
    B --> B1[Auto-generated<br/>Default]
    B --> B2[Custom CA<br/>K8s Secret]
    
    B1 --> C1[Operator creates<br/>receptor-ca Secret]
    C1 --> D1[Secret type: kubernetes.io/tls<br/>tls.crt + tls.key<br/>Single CA only]
    
    B2 --> C2[Customer creates Secret:<br/>kubectl create secret tls]
    C2 --> D2[Secret type: kubernetes.io/tls<br/>tls.crt + tls.key<br/>Single CA only]
    
    D1 --> E[Operator signs node certs<br/>with CA from Secret]
    D2 --> E
    
    E --> F[Execution nodes receive<br/>certificates signed by CA]
    
    F --> G[✅ No mesh-CA.crt file<br/>✅ K8s Secret format prevents bundles<br/>✅ QUIC buffer overflow impossible]
    
    classDef safe fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class D1,D2,E,F,G safe
```

---

## Deployment Type Details and Code References

| Deployment | Repository | CA Storage | Custom CA | Problem | Code Links |
|-----------|-----------|-----------|-----------|---------|------------|
| **RPM-A/B**<br/>(VM) | [automation-platform-collection](https://github.com/ansible/automation-platform-collection) | File: `mesh-CA.crt` | `mesh_ca_certfile`<br/>`custom_ca_cert` | custom_ca_cert bundles concatenated (154+ certs) | [tls_ca.yml](https://github.com/ansible/automation-platform-collection/blob/main/roles/receptor/tasks/tls_ca.yml)<br/>[receptor.conf.j2](https://github.com/ansible/automation-platform-collection/blob/main/roles/receptor/templates/receptor.conf.j2) |
| **CONT-A/B**<br/>(Container) | [aap-containerized-installer](https://github.com/ansible/aap-containerized-installer) | File: `mesh-CA.crt` | `custom_ca_cert` | custom_ca_cert bundles concatenated (154+ certs) | [mesh-CA.crt.j2](https://github.com/ansible/aap-containerized-installer/blob/main/roles/receptor/templates/mesh-CA.crt.j2)<br/>[tls.yml:78-94](https://github.com/ansible/aap-containerized-installer/blob/main/roles/receptor/tasks/tls.yml#L78-L94) |
| **OCP-A/B**<br/>(OpenShift) | AAP Operator | Secret: `receptor-ca` | K8s TLS Secret | None - Secret prevents bundles | Operator Hub |
| **AWX Operator**<br/>(K8s) | [awx-operator](https://github.com/ansible/awx-operator) | Secret: `receptor-ca` | K8s TLS Secret | None - Secret prevents bundles | [custom-receptor-certs.md](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/advanced-configuration/custom-receptor-certs.md)<br/>[receptor_ca_secret.yaml.j2](https://github.com/ansible/awx-operator/blob/devel/roles/installer/templates/secrets/receptor_ca_secret.yaml.j2) |
| **Core Receptor** | [receptor](https://github.com/ansible/receptor) | N/A | N/A | ReceptorVerifyFunc duplication | [netceptor.go:1105-1107](https://github.com/ansible/receptor/blob/devel/pkg/netceptor/netceptor.go#L1105-L1107) |

## Findings

### Summary:

**File-Based Deployments (Affected):**
- RPM-A/B, CONT-A/B use file-based mesh-CA.crt
- custom_ca_cert bundles concatenated into file
- Large files (154+ certs, 18KB+) exceed QUIC 16KB limit

**Secret-Based Deployments (Safe):**
- OCP-A/B, AWX/Hub/EDA operators use K8s Secrets
- Secret format stores single cert/key pair only
- Cannot accumulate large CA bundles architecturally

### Solution:

**Installer fix (file-based deployments):**
- **Target:** automation-platform-collection, aap-containerized-installer
- **Change:** Remove custom_ca_cert concatenation from mesh-CA.crt.j2
- **Scope:** RPM and containerized installation methods only

**Operator deployments (no fix needed):**
- **Already protected** by K8s Secret architecture
- **Cannot accumulate** large CA bundles
- **No code changes** required
