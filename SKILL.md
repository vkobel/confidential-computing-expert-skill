---
name: confidential-computing-expert
description: Use when working with TEEs (Intel TDX, AMD SEV-SNP), attestation protocols, confidential VMs, measurement chains, or cloud TEE deployments (Azure, GCP, AWS). Also use when reviewing security of TEE-based systems, implementing attestation flows, or answering questions about hardware-rooted trust.
---

# Confidential Computing Expert

Domain expertise for TEE-based systems: hardware isolation, attestation, measurement chains, and cloud confidential VM deployments.

## Reference Routing

| Question domain | Read |
|----------------|------|
| TDX/SEV-SNP hardware, measurement registers, key derivation, trust model | `tee-fundamentals.md` |
| Azure vs GCP vs AWS tradeoffs, platform gotchas, ecosystem tooling | `cloud-tee-comparison.md` |
| RCAR protocol, Trustee KBS, PCK verification, proof chains, HCL envelope | `attestation-protocols.md` |
| Threat modeling, security layers, auditor verification, hardening | `threat-model-patterns.md` |

Load the relevant reference file(s) before answering. For cross-cutting questions, load multiple.

## Key Principles

1. **Every cloud TEE has an unverifiable link** — no platform gives full source-to-measurement verifiability. Know where the gap is.
2. **Measurements are only useful if pinned** — always compare against known-good values from reproducible builds.
3. **Debug mode = no security** — always reject quotes with debug attribute set.
4. **HKDF over raw hashing** — domain separation prevents cross-context key collisions.
5. **Cross-platform reduces risk** — different platforms have gaps in different places.

## Quick Terminology

| Term | Meaning |
|------|---------|
| MRTD | Initial TD measurement (firmware+kernel), locked at boot |
| RTMR[0-3] | Runtime measurement registers, extendable |
| PCK | Intel Provisioning Certification Key (hardware root of trust) |
| RCAR | Remote Configuration And Reporting (KBS attestation protocol) |
| KBS | Key Broker Service (Trustee — verifies attestation, releases secrets) |
| RVPS | Reference Value Provider Service (stores expected measurements) |
| CVM | Confidential VM |
| HCL | Host Compatibility Layer (Azure paravisor binding vTPM to TDX) |
| TCB | Trusted Computing Base (what you must trust) |
| SVN | Security Version Number (for downgrade prevention) |
