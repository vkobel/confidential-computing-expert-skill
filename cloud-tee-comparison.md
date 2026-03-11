# Cloud TEE Platform Comparison

## Quick Comparison (as of March 2026)

| Aspect | Azure TDX | Azure SEV-SNP | GCP TDX | AWS SEV-SNP |
|--------|-----------|---------------|---------|-------------|
| **Paravisor** | OpenHCL (open source) | HCL (closed) | None (raw TDX) | None (raw SEV-SNP) |
| **Firmware source** | Open (OpenHCL) | Closed | Closed (Google OVMF) | Open (OVMF) |
| **Binary reproducible** | No (opaque build) | No | Yes (`gce-tcb-verifier`) | Partially |
| **Guest TDX access** | Via vTPM only | N/A | Raw `/dev/tdx_guest` | N/A |
| **RTMR3 extension** | Unknown (via vTPM?) | N/A | Yes (sysfs/ioctl) | N/A |
| **Evidence format** | Composite (TDX+vTPM+HCL) | SNP report | Raw DCAP QuoteV4 | Raw SNP report |
| **Attestation service** | MAA or Trustee | MAA | GCE API | NSM/custom |
| **Rust crate** | `az-tdx-vtpm` (mature) | `sev-snp` | `tdx-guest` | `aws-nitro` |

## The Unverifiable Link (Critical Concept)

Every cloud TEE platform has ONE link in the measurement chain that cannot be independently verified:

- **Azure TDX:** Source is open (OpenHCL) -> **[opaque build]** -> binary -> MRTD
  - You can read the code but cannot prove the running binary was built from that code
- **GCP TDX:** **[closed source]** -> binary -> endorsement -> MRTD
  - You can verify binary->MRTD mapping via `gce-tcb-verifier`, but cannot audit the source
- **AWS SEV-SNP:** Open firmware -> **[closed image tool]** -> modified OS -> MEASUREMENT
  - Firmware is reproducible, but guest image construction is opaque

**Implication:** No single cloud platform provides full end-to-end verifiability. Cross-platform deployment (Phase 1 Azure + Phase 2 GCP) reduces risk because the unverifiable links are in different parts of the chain.

## Architecture Differences

### Paravisor Model (Azure TDX)
```
Hardware TDX
  └─ OpenHCL Paravisor (runs inside TD)
       ├─ vTPM (24 PCRs, granular measurement)
       ├─ HCL Envelope (binds vTPM AK to TDX report_data)
       └─ Guest OS
            └─ Application
```
- vTPM provides 24 PCRs for fine-grained boot measurement
- HCL envelope cryptographically binds vTPM attestation key to TDX hardware report
- Composite evidence = TDX quote + vTPM quote + HCL envelope
- No raw `/dev/tdx_guest` access from guest

### Raw TDX Model (GCP)
```
Hardware TDX
  └─ Guest OS (direct TDX access)
       ├─ /dev/tdx_guest (raw quotes)
       ├─ configfs-tsm (sysfs interface)
       └─ Application
```
- Direct hardware access, standard DCAP QuoteV4
- RTMR3 extensible by guest application
- Simpler evidence format, standard verification tools
- Firmware binary verifiable via Google's `gce-tcb-verifier` endorsement

## Measurement Register Comparison

| Register | TDX | SEV-SNP | vTPM (Azure) |
|----------|-----|---------|--------------|
| Initial image | MRTD | MEASUREMENT | PCR[0-3] |
| Firmware | RTMR[0] | N/A | PCR[0-3] |
| Boot chain | RTMR[1] | N/A | PCR[4-9] |
| OS/App runtime | RTMR[2] | N/A | PCR[10-14] |
| Guest-extensible | RTMR[3] | N/A | PCR[15-23] |
| **Granularity** | 5 registers | 1 register | 24 registers |
| **Runtime extend** | Yes (RTMR3) | No | Yes (any PCR) |

## Quote Verification Options

### Option A: Local DCAP verification (`dcap-qvl` crate)
- **Pro:** No network dependency, full control
- **Con:** C dependency (Intel SGX SDK), needs PCCS endpoint for collateral

### Option B: Manual PCK chain verification
- **Pro:** Pure Rust, no C deps
- **Con:** Must implement certificate chain validation, revocation checks

### Option C: Cloud verification API (e.g., Phala)
```
POST https://cloud-api.phala.network/api/v1/attestations/verify
Body: {"hex": "<tdx_quote_hex>"}
Returns: { verified: bool, parsed fields... }
```
- **Pro:** No C deps, no PCCS, works anywhere
- **Con:** Network dependency, trusts third-party verification

### Option D: Cloud-native (MAA / GCE attestation API)
- **Pro:** Integrated with cloud provider
- **Con:** Platform lock-in, trusts cloud provider

## Platform-Specific Gotchas

### Azure TDX
- No raw `/dev/tdx_guest` — must use `az-tdx-vtpm` crate
- Composite evidence format requires custom parsing (not standard DCAP)
- RTMR3 guest extension availability unknown — verify on real hardware
- `az-tdx-vtpm` wraps vTPM PCR extension, not raw RTMR

### GCP TDX
- Standard `configfs-tsm` interface for quotes
- RTMR3 extensible via `sysfs` or direct `ioctl`
- Quote verification via `gce-tcb-verifier` endorsement chain
- C3 instance type required for TDX

### AWS SEV-SNP
- Single MEASUREMENT register — no runtime extension
- Cannot separate firmware from application measurement
- `report_data` limited but available for binding
- Nitro enclaves are separate from SEV-SNP CVMs

## Ecosystem

- **Trustee KBS:** CNCF project, Rust, supports TDX + SEV-SNP, RCAR protocol
- **Azure MAA:** Microsoft Attestation Service, cloud-native, Azure-only
- **CoCo (Confidential Containers):** Kubernetes + Kata + attestation-agent, abstracts platform differences
- **Constellation/Contrast:** Edgeless Systems tooling for confidential Kubernetes
- **dstack:** Phala's bare-metal TDX framework (good crypto patterns, not for cloud CVMs)

## Strategic Recommendation

| Phase | Platform | Why |
|-------|----------|-----|
| Phase 1 | Azure TDX | Source auditability (OpenHCL), mature Rust crate, Trustee ecosystem |
| Phase 2 | GCP TDX | Deployment verifiability (`gce-tcb-verifier`), raw hardware access, RTMR3 |
| Combined | Both | Different unverifiable links — cross-reference reduces compromise probability |
