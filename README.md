# QUIC Buffer Overflow Analysis

### **[Visual Problem/Solution Flow Diagram](quic-fixes-diagram.md)**

## Problem
Receptor QUIC connections fail with `CRYPTO_BUFFER_EXCEEDED` when mesh-CA.crt exceeds 16,384 bytes.

## Root Cause
**File:** `aap-containerized-installer/roles/receptor/templates/mesh-CA.crt.j2`

**Current template:**
```jinja
{{ _internal_ca_cert.content | b64decode }}
{%- if custom_ca_cert is defined %}
{{ _custom_ca_cert.content | b64decode }}
{%- endif %}
```

**Issue:** Simple concatenation creates large CA bundles (154+ certificates, ~18,342 bytes) that exceed QUIC's 16,384-byte limit.

## Solution Options

### Option 1: Simple Fix (Recommended)
**Remove custom CA concatenation entirely:**
```jinja
{{ _internal_ca_cert.content | b64decode }}
```

**Rationale:** 
- Receptor mesh only needs internal CA for authentication
- Custom CAs properly handled in system trust store (`/etc/pki/ca-trust/source/anchors/`)
- KCS 7129200 proves this works (customers successfully use workaround)

### Option 2: Smart Filtering
**Add certificate processing before concatenation:**
- Filter expired certificates
- Remove duplicates by SHA-256 fingerprint  
- Size validation before template rendering

## Customer Impact
- **Current:** Multiple support cases (04227540, 04226082, 04230597, etc.)
- **Workaround:** Manual editing of mesh-CA.crt (tedious per KCS 7129200)
- **Fix impact:** Eliminates QUIC buffer overflow for new deployments

## Files to Modify
- **Template:** `aap-containerized-installer/roles/receptor/templates/mesh-CA.crt.j2`
- **Tasks:** `aap-containerized-installer/roles/receptor/tasks/tls.yml` (lines 78-94)

## References
- **KCS:** https://access.redhat.com/solutions/7129200
- **Epic:** [AAP-51480](https://issues.redhat.com/browse/AAP-51480)
- **Related Story:** [AAP-51479](https://issues.redhat.com/browse/AAP-51479)
