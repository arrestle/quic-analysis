# Receptor Internal CA and Self-Signing Process

## How AAP Installer Creates Internal PKI

```mermaid
graph TD
    A[AAP Installation Starts] --> B[Common TLS Setup]
    
    B --> B1[Generate Internal Root CA]
    B1 --> B2[Create CA Private Key<br/>_ca_tls_dir/ca.key]
    B2 --> B3[Create Self-Signed CA Certificate<br/>_ca_tls_dir/ca.cert<br/>provider: selfsigned]
    
    B3 --> C[Component Certificate Generation]
    
    %% All components use the same internal CA
    C --> C1[Receptor Certificate]
    C --> C2[Controller Certificate]
    C --> C3[Hub Certificate]
    C --> C4[EDA Certificate]
    C --> C5[Gateway Certificate]
    C --> C6[PostgreSQL Certificate]
    C --> C7[Redis Certificate]
    
    %% Receptor Certificate Details
    C1 --> C1A[Generate receptor.key]
    C1A --> C1B[Create CSR with Node ID OID<br/>1.3.6.1.4.1.2312.19.1]
    C1B --> C1C[Sign with Internal CA<br/>provider: ownca<br/>ownca_path: ca.cert]
    C1C --> C1D[receptor.crt<br/>Valid for Receptor Node ID]
    
    %% Controller Certificate Details
    C2 --> C2A[Generate tower.key]
    C2A --> C2B[Create CSR with hostname]
    C2B --> C2C[Sign with Internal CA<br/>provider: ownca<br/>ownca_path: ca.cert]
    C2C --> C2D[tower.cert<br/>Valid for Controller hostname]
    
    %% Similar pattern for other components
    C3 --> C3D[pulp.cert<br/>Signed by Internal CA]
    C4 --> C4D[eda.cert<br/>Signed by Internal CA]
    C5 --> C5D[gateway.cert<br/>Signed by Internal CA]
    C6 --> C6D[postgresql server.cert<br/>Signed by Internal CA]
    C7 --> C7D[redis server.cert<br/>Signed by Internal CA]
    
    %% Configuration Phase
    C1D --> D[Receptor Configuration]
    C2D --> E[Controller Configuration]
    C3D --> F[Hub Configuration]
    C4D --> G[EDA Configuration]
    C5D --> H[Gateway Configuration]
    C6D --> I[PostgreSQL Configuration]
    C7D --> J[Redis Configuration]
    
    %% Receptor configuration - Two connection types
    D --> D1[Configure mesh-CA.crt<br/>clientcas: mesh-CA.crt<br/>rootcas: mesh-CA.crt]
    D1 --> D2A[Network Connections<br/>tcp-peers/tcp-listeners<br/>Uses TLS + mesh-CA.crt]
    D1 --> D2B[Local Control Service<br/>Unix Socket<br/>No TLS validation]
    
    D2A --> D3[Mesh Node Authentication<br/>Validates peer certificates<br/>against mesh-CA.crt]
    D2B --> D4[AWX Control Connection<br/>/var/run/awx-receptor/receptor.sock<br/>No certificate validation]
    
    %% Component configurations
    E --> E1[Controller → Receptor<br/>Unix socket local only]
    F --> F1[Hub → Database<br/>TLS with internal CA]
    G --> G1[EDA → Database<br/>TLS with internal CA]
    H --> H1[Gateway → Services<br/>TLS with internal CA]
    I --> I1[PostgreSQL accepts TLS<br/>Internal CA certs]
    J --> J1[Redis accepts TLS<br/>Internal CA certs]
    
    %% Runtime behavior
    D3 --> K[Receptor Mesh Network<br/>TLS validated via mesh-CA.crt]
    D4 --> L[Local Control Commands<br/>No certificate validation]
    E1 --> L
    F1 --> M[Component Communication<br/>Internal TLS]
    G1 --> M
    H1 --> M
    I1 --> M
    J1 --> M
    
    K --> N[Network: Uses mesh-CA.crt<br/>Only needs internal CA]
    L --> O[Local: Unix socket<br/>No mesh-CA.crt needed]
    M --> P[Internal: TLS with internal CA<br/>No mesh-CA.crt needed]
    
    N --> Q[Self-Contained PKI<br/>All network auth via internal CA]
    O --> Q
    P --> Q
    
    %% Styling - White text on black background
    classDef ca fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef cert fill:#000000,stroke:#0066cc,stroke-width:3px,color:#ffffff
    classDef config fill:#000000,stroke:#ff8800,stroke-width:3px,color:#ffffff
    classDef runtime fill:#000000,stroke:#8800ff,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class B,B1,B2,B3 ca
    class C,C1,C2,C3,C4,C5,C6,C7,C1A,C1B,C1C,C1D,C2A,C2B,C2C,C2D,C3D,C4D,C5D,C6D,C7D cert
    class D,D1,D2A,D2B,D3,D4,E,F,G,H,I,J,E1,F1,G1,H1,I1,J1 config
    class K,L,M,N,O,P,Q runtime
```

## Key Points

### Internal CA Design
1. **Single Root CA** signs all component certificates
   - **Code:** `aap-containerized-installer/roles/common/tasks/tls.yml` (provider: selfsigned)
2. **Self-contained PKI** - no external dependencies
   - **Code:** All components use `provider: ownca` with same internal CA
3. **Consistent trust model** across all AAP components

### Connection Types and Certificate Validation

#### **Network Connections (TLS Validated):**
- **Receptor mesh** (node-to-node): Uses tcp-peers/tcp-listeners with TLS
- **Validates certificates** against mesh-CA.crt during QUIC handshakes
- **Code:** `receptor/pkg/netceptor/tlsconfig.go` lines 208-217 (RootCAs), 125-132 (ClientCAs)
- **All mesh nodes** present certificates signed by internal CA
- **Only internal CA needed** in mesh-CA.crt for validation

#### **Unix Socket Connections (No TLS):**
- **AWX → Receptor** control service: `/var/run/awx-receptor/receptor.sock`
- **Code:** `awx/awx/main/tasks/receptor.py` line 165, `receptorctl/socket_interface.py` lines 96-104
- **Local process communication** (no network layer)
- **No certificate validation** occurs (Unix sockets don't use TLS)
- **mesh-CA.crt not used** for local control service connections

#### **Internal Component Connections (TLS with Internal CA):**
- **Database connections:** TLS enabled (`sslmode: require`) but uses internal CA certs
- **Code:** `awx/tools/docker-compose/ansible/roles/sources/templates/database.py.j2`
- **Gateway Redis:** Separate `redis_ca.cert` file, not mesh-CA.crt
- **Code:** `aap-gateway/aap_gateway_api/defaults.py` lines 58-62
- **No mesh-CA.crt involvement** - separate from Receptor mesh

### Certificate Characteristics
- **Receptor certificates** include Node ID OIDs (1.3.6.1.4.1.2312.19.1) for mesh authentication
- **Component certificates** include hostnames/SANs for service authentication
- **All certificates signed by same internal CA** for unified trust

### Why This Works
- **Mutual trust** - all components trust the same root CA
- **Secure communication** - proper TLS encryption and authentication
- **No external dependencies** - works in isolated environments
- **Proper separation** - Unix sockets for local, TLS for network

### The Problem
**Code:** `aap-containerized-installer/roles/receptor/templates/mesh-CA.crt.j2`

When `custom_ca_cert` is provided, installer concatenates it with internal CA in `mesh-CA.crt`, creating unnecessary bloat. Since:
- **All mesh nodes** use internal CA-signed certificates (installer generates these)
- **Local connections** don't validate mesh-CA.crt (Unix sockets - no TLS)
- **Only network mesh connections** use mesh-CA.crt for validation
- **Custom CAs provide no value** for Receptor mesh authentication

**Evidence:** KCS 7129200 workaround (keep only first certificate) works without functionality loss.

### The Solution
**Template change:** `aap-containerized-installer/roles/receptor/templates/mesh-CA.crt.j2`

Remove custom CA concatenation - Receptor mesh only needs internal CA for node authentication. Custom CAs remain available in system trust store (`/etc/pki/ca-trust/source/anchors/`) for external integrations.

**Code reference:** `automation-platform-collection/roles/certificate_authority/tasks/add_cacert.yml` (system trust installation)
