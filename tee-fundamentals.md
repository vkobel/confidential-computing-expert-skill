# TEE Fundamentals

## Hardware Platforms

### Intel TDX (Trust Domain Extensions)
- **Isolation unit:** VM-level (Trust Domain)
- **Measurement registers:**
  - **MRTD** (SHA-384): Initial TD contents (firmware + kernel + initrd), locked before first instruction
  - **RTMR[0]**: Firmware measurements (BIOS, UEFI variables)
  - **RTMR[1]**: OS loader measurements (kernel, initrd, cmdline)
  - **RTMR[2]**: OS runtime measurements (IMA, application binaries)
  - **RTMR[3]**: Guest-extensible (application can extend at runtime)
- **Quote format:** DCAP QuoteV4 (SGX-compatible infrastructure)
- **Root of trust:** Intel PCK (Provisioning Certification Key) certificate chain
  - PCK Cert (per-platform) -> Intermediate CA -> Intel Root CA
- **Key registers in TD Report:**
  - `td_attributes`: Bit 0 = debug mode (MUST reject if set in production)
  - `report_data` (64 bytes): Guest-supplied binding field (e.g., hash of public key)
  - `mr_config_id` / `mr_owner`: Additional identity fields

### AMD SEV-SNP (Secure Encrypted Virtualization - Secure Nested Paging)
- **Isolation unit:** VM-level
- **Measurement:** Single `MEASUREMENT` register (SHA-256/384), fixed at launch
- **No runtime extension** — cannot extend measurements after boot
- **Quote format:** AMD SEV Attestation Report
- **Root of trust:** AMD Root Key (ARK) -> SEV Signing Key (ASK) -> VCEK (per-chip)
- **Key fields:** `report_data` (64 bytes), `host_data` (32 bytes)

### AWS Nitro Enclaves
- **Isolation unit:** Enclave VM (carved from parent EC2 instance)
- **Not silicon-TEE:** Hypervisor-based isolation (Nitro Hypervisor), not CPU memory encryption
- **Measurement registers (PCRs):**
  - **PCR0**: Enclave Image File (EIF) hash
  - **PCR1**: Linux kernel and bootstrap hash
  - **PCR2**: Application code hash
  - **PCR3**: IAM role of parent instance
  - **PCR4**: Instance ID of parent instance
- **No runtime extension** — PCRs are fixed at enclave boot
- **Attestation:** NSM (Nitro Security Module) device at `/dev/nsm`
- **Root of trust:** AWS Nitro Attestation PKI (root CA published by AWS)
- **Evidence format:** CBOR-encoded attestation document (COSE Sign1)
- **Key fields:** `user_data` (512 bytes), `nonce` (512 bytes), `public_key` (1024 bytes)
- **No persistent storage or network** — all I/O via vsock to parent instance
- **Critical difference from CVMs:** Parent instance is NOT in the TCB but controls all enclave I/O

### Nitro vs CVM Comparison
| Aspect | Nitro Enclaves | TDX/SEV-SNP CVMs |
|--------|---------------|------------------|
| Isolation | Hypervisor-based | CPU memory encryption |
| Network | None (vsock only) | Full network stack |
| Storage | None (stateless) | Full disk (encrypted) |
| Measurement | 5 PCRs (fixed) | MRTD + 4 RTMRs (TDX) or 1 MEASUREMENT (SNP) |
| Runtime extend | No | Yes (RTMR3 on TDX) |
| I/O trust | Parent controls all I/O | Guest has direct I/O |
| Hardware root | Nitro Hypervisor | CPU silicon |

## Measurement Chain (Generic Pattern)

```
Layer 0: CPU Hardware
  MRTD/MEASUREMENT locked at boot
  ↓
Layer 1: Firmware/Paravisor (if present)
  May add vTPM, envelope binding
  ↓
Layer 2: Boot Chain
  Kernel, initrd, cmdline measured into RTMR[1] or PCR[4-9]
  ↓
Layer 3: Application
  Binary measured into RTMR[2] or PCR[10] (IMA)
  Runtime state bound via report_data
  ↓
Layer 4: Attestation Service (KBS/MAA)
  Verifies composite evidence against policy
  ↓
Layer 5: External Verifier
  Reproduces build, validates certificate chain, checks measurements
```

## Trust Model

**Inside TCB (trusted):**
- CPU silicon (TDX/SEV-SNP hardware)
- Guest VM contents (kernel + application)
- Attestation verification service
- Reproducible build system

**Outside TCB (untrusted):**
- Cloud provider operators
- Host OS / hypervisor
- Kubernetes control plane
- Container registries, CI/CD pipelines

## Key Derivation Best Practices

- **Use HKDF-SHA256** (not raw SHA-256) for domain separation
- **Context binding:** Include purpose, algorithm, version in derivation
  ```
  HKDF-Expand(seed, [b"signing-svc", b"ed25519", b"v1"])
  ```
- **Report data tagging:** Prefix hashes with purpose string
  ```
  SHA-512("signing-svc-ephemeral:" || pubkey)
  ```
  Prevents cross-context collisions if same key used elsewhere

## Debug Mode Rejection

Always check TDX debug attribute before trusting a quote:
```
if td_attributes[0] & 0x01 != 0 {
    reject("debug mode TDX report")
}
```
Debug mode means the host can read TD memory — attestation is meaningless.

## Attestation Freshness

Two approaches:
1. **Per-request quotes:** Generate fresh quote with nonce in report_data (expensive, ~100ms)
2. **Cached quote + signed timestamp:** Cache quote at boot, sign responses with timestamp (cheaper, slight staleness)

Pattern from production systems: cache quote, include nonce in signed response, verifier checks both quote validity and response freshness.
