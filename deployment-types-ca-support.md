# AAP Deployment Types and Custom CA Support

## Overview of AAP Deployment Methods and Certificate Handling

```mermaid
graph TD
    A[AAP Deployment Options] --> B[Traditional VM/Bare Metal]
    A --> C[Containerized/Podman]
    A --> D[Kubernetes/OpenShift]
    A --> E[Cloud Managed Services]
    
    %% Traditional VM/Bare Metal
    B --> B1[automation-platform-collection<br/>Ansible-based installer]
    B1 --> B2{Custom CA Options}
    B2 --> B2A[mesh_ca_certfile/keyfile<br/>Customer CA signs node certs]
    B2 --> B2B[custom_ca_cert<br/>System trust store]
    B2A --> B3[✅ Supported<br/>Single customer CA]
    B2B --> B4[✅ Supported<br/>CA bundle for external systems]
    
    %% Containerized
    C --> C1[aap-containerized-installer<br/>Podman-based]
    C1 --> C2{Custom CA Options}
    C2 --> C2A[Internal CA only<br/>Default behavior]
    C2 --> C2B[custom_ca_cert<br/>System trust + mesh-CA.crt]
    C2A --> C3[✅ Default<br/>Self-signed internal CA]
    C2B --> C4[⚠️ QUIC Issue<br/>Large bundles cause overflow]
    
    %% Kubernetes/OpenShift
    D --> D1[AWX Operator<br/>Upstream K8s]
    D --> D2[AAP on OpenShift<br/>Enterprise K8s]
    
    D1 --> D1A{Certificate Method}
    D1A --> D1A1[Cert-manager<br/>K8s native certs]
    D1A --> D1A2[Custom certificates<br/>via Secrets]
    D1A1 --> D1B[✅ Supported<br/>K8s cert infrastructure]
    D1A2 --> D1C[✅ Supported<br/>Secret-based certs]
    
    D2 --> D2A{Certificate Method}
    D2A --> D2A1[OpenShift certs<br/>Service CA]
    D2A --> D2A2[Custom CA<br/>via ConfigMap/Secret]
    D2A1 --> D2B[✅ Supported<br/>OpenShift PKI]
    D2A2 --> D2C[❓ Unknown<br/>Needs investigation]
    
    %% Cloud Services
    E --> E1[AWS Managed]
    E --> E2[Azure Managed]
    E --> E3[Google Cloud]
    
    E1 --> E1A[AWS Certificate Manager<br/>ACM integration]
    E1A --> E1B[❓ Unknown<br/>No AAP-specific operator found]
    
    E2 --> E2A[Azure Key Vault<br/>Certificate integration]
    E2A --> E2B[❓ Unknown<br/>No AAP-specific operator found]
    
    E3 --> E3A[Google Certificate Authority<br/>Service integration]
    E3A --> E3B[❓ Unknown<br/>No AAP-specific operator found]
    
    %% Receptor Mesh Impact
    B3 --> F[Receptor mesh-CA.crt]
    B4 --> F
    C3 --> F
    C4 --> F
    D1B --> G[K8s Secret/ConfigMap<br/>Not mesh-CA.crt file]
    D1C --> G
    D2B --> G
    D2C --> G
    
    F --> F1{mesh-CA.crt Size}
    F1 -->|< 16KB| F2[✅ QUIC Works]
    F1 -->|> 16KB| F3[❌ CRYPTO_BUFFER_EXCEEDED]
    
    G --> G1[K8s Certificate Management<br/>Different architecture]
    
    %% Styling
    classDef supported fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef problem fill:#000000,stroke:#ff0000,stroke-width:3px,color:#ffffff
    classDef unknown fill:#000000,stroke:#ffaa00,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class B3,B4,C3,D1B,D1C,D2B supported
    class C4,F3 problem
    class D2C,E1B,E2B,E3B unknown
```

## Deployment Type Analysis

### ✅ Traditional VM/Bare Metal (automation-platform-collection)

**Installer:** Ansible-based automation-platform-collection
**Receptor Config:** File-based `/etc/receptor/tls/ca/mesh-CA.crt`

**Custom CA Support:**
1. **mesh_ca_certfile/mesh_ca_keyfile:**
   - Customer provides CA certificate and private key
   - Installer uses it to sign all Receptor node certificates
   - Result: mesh-CA.crt contains customer's single CA
   - **Use case:** Enterprise security requirements for signed certificates

2. **custom_ca_cert:**
   - Customer provides CA bundle for system trust
   - Installed in `/etc/pki/ca-trust/source/anchors/`
   - **Problem:** Currently also concatenated into mesh-CA.crt (causes QUIC overflow)
   - **Use case:** External PostgreSQL cert auth, system integrations

**Code Reference:** `automation-platform-collection/roles/receptor/tasks/tls_ca.yml`

### ✅ Containerized/Podman (aap-containerized-installer)

**Installer:** Podman-based containerized deployment
**Receptor Config:** Mounted into containers

**Custom CA Support:**
1. **Internal CA (default):**
   - Installer generates self-signed CA
   - Signs all component certificates
   - Result: Minimal mesh-CA.crt (~1,500 bytes)

2. **custom_ca_cert:**
   - Installed in system trust store
   - **Currently concatenated into mesh-CA.crt** ← QUIC PROBLEM
   - Large bundles (154+ certs) cause buffer overflow

**Code Reference:** `aap-containerized-installer/roles/receptor/templates/mesh-CA.crt.j2`

### ✅ Kubernetes/OpenShift (AWX Operator)

**Installer:** Operator-based deployment
**Receptor Config:** ConfigMaps/Secrets in K8s

**Custom CA Support:**
1. **Cert-manager integration:**
   - Kubernetes-native certificate management
   - Automated certificate lifecycle
   - Different architecture than file-based

2. **Custom certificates via Secrets:**
   - Customer provides certificates as K8s Secrets
   - Mounted into pods
   - No file-based mesh-CA.crt

**Note:** AWX operator is upstream project. AAP on OpenShift deployment specifics need further investigation.

**Code Reference:** [ansible/awx-operator](https://github.com/ansible/awx-operator)

### ❓ Cloud Managed Services (AWS/Azure/GCP)

**Status:** No AAP-specific operators found in analyzed repositories

**Likely Scenarios:**
- AAP deployed via marketplace offerings
- Uses cloud provider certificate management (ACM, Key Vault, CA Service)
- May use containerized or VM-based installers with cloud integration
- Custom CA handling depends on underlying deployment method

**Needs Investigation:** 
- AWS Marketplace AAP offering
- Azure Marketplace AAP offering  
- Google Cloud Marketplace AAP offering

## Key Findings

### Common Pattern:
**All deployment methods that use file-based Receptor configuration** (VM, containerized) reference `mesh-CA.crt` and can encounter the QUIC buffer overflow issue with large CA bundles.

### Kubernetes Exception:
**Operator-based deployments** use different certificate architecture (Secrets/ConfigMaps) and may not have the same mesh-CA.crt file-based approach.

### The Problem Scope:
**QUIC buffer overflow primarily affects:**
- ✅ Traditional VM installations using custom_ca_cert
- ✅ Containerized installations using custom_ca_cert
- ❓ OpenShift deployments (needs investigation)
- ❓ Cloud marketplace offerings (needs investigation)

### The Solution:
**Installer fix applies to:**
- automation-platform-collection (VM installer)
- aap-containerized-installer (Podman installer)

**May not apply to:**
- Kubernetes operator deployments (different architecture)
- Cloud-managed offerings (vendor-specific)
