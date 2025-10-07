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
    
    %% Receptor mesh configuration
    D --> D1[Configure mesh-CA.crt<br/>clientcas: mesh-CA.crt<br/>rootcas: mesh-CA.crt]
    D1 --> D2[All Receptor nodes trust<br/>same internal CA]
    D2 --> D3[Mutual TLS authentication<br/>between Receptor nodes]
    
    %% Component configurations
    E --> E1[Controller trusts internal CA<br/>for internal communications]
    F --> F1[Hub trusts internal CA<br/>for internal communications]
    G --> G1[EDA trusts internal CA<br/>for internal communications]
    H --> H1[Gateway trusts internal CA<br/>for internal communications]
    I --> I1[PostgreSQL uses internal CA<br/>for TLS connections]
    J --> J1[Redis uses internal CA<br/>for TLS connections]
    
    %% Runtime behavior
    D3 --> K[Receptor Mesh Operation]
    E1 --> L[Controller Operation]
    F1 --> L
    G1 --> L
    H1 --> L
    I1 --> L
    J1 --> L
    
    K --> M[All mesh communication<br/>authenticated via internal CA]
    L --> N[All component communication<br/>authenticated via internal CA]
    
    M --> O[Self-Contained PKI<br/>No external dependencies]
    N --> O
    
    %% Styling - White text on black background
    classDef ca fill:#000000,stroke:#00aa00,stroke-width:3px,color:#ffffff
    classDef cert fill:#000000,stroke:#0066cc,stroke-width:3px,color:#ffffff
    classDef config fill:#000000,stroke:#ff8800,stroke-width:3px,color:#ffffff
    classDef runtime fill:#000000,stroke:#8800ff,stroke-width:3px,color:#ffffff
    classDef default fill:#000000,color:#ffffff
    
    class B,B1,B2,B3 ca
    class C,C1,C2,C3,C4,C5,C6,C7,C1A,C1B,C1C,C1D,C2A,C2B,C2C,C2D,C3D,C4D,C5D,C6D,C7D cert
    class D,D1,D2,D3,E,F,G,H,I,J,E1,F1,G1,H1,I1,J1 config
    class K,L,M,N,O runtime
```

## Key Points

### Internal CA Design
1. **Single Root CA** signs all component certificates
2. **Self-contained PKI** - no external dependencies
3. **Consistent trust model** across all AAP components

### Certificate Characteristics
- **Receptor certificates** include Node ID OIDs for mesh authentication
- **Component certificates** include hostnames/SANs for service authentication
- **All certificates signed by same internal CA** for unified trust

### Why This Works
- **Mutual trust** - all components trust the same root CA
- **Secure communication** - proper TLS encryption and authentication
- **No external dependencies** - works in isolated environments
- **Compliance ready** - proper certificate hierarchy

### The Problem
When `custom_ca_cert` is provided, installer concatenates it with internal CA in `mesh-CA.crt`, creating unnecessary bloat for Receptor mesh authentication that only needs the internal CA.

### The Solution
Keep internal CA separate for Receptor mesh, use custom CAs only where actually needed (system trust store, external integrations).
