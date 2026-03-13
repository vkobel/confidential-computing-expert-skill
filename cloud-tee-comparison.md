# Cloud TEE Platform Comparison

## Quick Comparison (as of March 2026)

| Aspect | Azure TDX | Azure SEV-SNP | GCP TDX | AWS SEV-SNP | AWS Nitro Enclaves |
|--------|-----------|---------------|---------|-------------|-------------------|
| **Isolation type** | CPU (TDX) | CPU (SEV-SNP) | CPU (TDX) | CPU (SEV-SNP) | Hypervisor |
| **Paravisor** | OpenHCL (open source) | HCL (closed) | None (raw TDX) | None (raw SEV-SNP) | Nitro Hypervisor |
| **Firmware source** | Open (OpenHCL) | Closed | Closed (Google OVMF) | Open (OVMF) | N/A |
| **Binary reproducible** | No (opaque build) | No | Yes (`gce-tcb-verifier`) | Partially | Yes (EIF is deterministic) |
| **Guest TDX access** | Via vTPM only | N/A | Raw `/dev/tdx_guest` | N/A | N/A |
| **RTMR3 extension** | Unknown (via vTPM?) | N/A | Yes (sysfs/ioctl) | N/A | No (PCRs fixed at boot) |
| **Evidence format** | Composite (TDX+vTPM+HCL) | SNP report | Raw DCAP QuoteV4 | Raw SNP report | CBOR/COSE Sign1 |
| **Attestation service** | MAA or Trustee | MAA | GCE API | NSM/custom | NSM (`/dev/nsm`) |
| **Rust crate** | `az-tdx-vtpm` (mature) | `sev-snp` | `tdx-guest` | `aws-nitro` | `aws-nitro-enclaves-nsm-api` |

**Important:** Nitro Enclaves PCRs are NOT the same as vTPM PCRs (Azure) or TDX RTMRs. Nitro PCR0-4 have Nitro-specific semantics (see table above). Do not confuse them.

## The Unverifiable Link (Critical Concept)

Every cloud TEE platform has ONE link in the measurement chain that cannot be independently verified:

- **Azure TDX:** Source is open (OpenHCL) -> **[opaque build]** -> binary -> MRTD
  - You can read the code but cannot prove the running binary was built from that code
- **GCP TDX:** **[closed source]** -> binary -> endorsement -> MRTD
  - You can verify binary->MRTD mapping via `gce-tcb-verifier`, but cannot audit the source
- **AWS SEV-SNP:** Open firmware -> **[closed image tool]** -> modified OS -> MEASUREMENT
  - Firmware is reproducible, but guest image construction is opaque
- **AWS Nitro Enclaves:** Docker image -> EIF (deterministic) -> PCR0. **Smallest gap** — but isolation is hypervisor-based, not silicon

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

| Register | TDX | SEV-SNP | vTPM (Azure) | Nitro Enclaves |
|----------|-----|---------|--------------|----------------|
| Initial image | MRTD | MEASUREMENT | PCR[0-3] | PCR0 (EIF hash) |
| Firmware/kernel | RTMR[0] | N/A | PCR[0-3] | PCR1 (kernel+bootstrap) |
| Boot chain | RTMR[1] | N/A | PCR[4-9] | — |
| OS/App runtime | RTMR[2] | N/A | PCR[10-14] | PCR2 (application) |
| Guest-extensible | RTMR[3] | N/A | PCR[15-23] | — |
| Identity/policy | — | — | — | PCR3 (IAM role), PCR4 (instance ID) |
| **Granularity** | 5 registers | 1 register | 24 registers | 5 registers |
| **Runtime extend** | Yes (RTMR3) | No | Yes (any PCR) | No |

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
- Nitro Enclaves are separate from SEV-SNP CVMs (different isolation model)

### AWS Nitro Enclaves
- No network, no persistent storage — all I/O via vsock to parent EC2 instance
- Parent instance is **outside TCB** but controls all enclave inputs
- Attestation via `/dev/nsm` — returns CBOR-encoded COSE Sign1 document
- PCR0 = EIF hash, PCR1 = kernel, PCR2 = application, PCR3 = IAM role, PCR4 = instance ID
- Pin PCR0+PCR1+PCR2 for code identity; PCR3 for IAM policy binding
- Verify attestation certificate chain to AWS Nitro root CA (published by AWS)
- `user_data` field (512 bytes) in attestation request — use for nonce/key binding
- EIF is deterministic from Dockerfile — reproducible builds possible
- **vsock security:** Authenticate + sequence-number all messages; implement read timeouts; never trust host-supplied time or randomness

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
