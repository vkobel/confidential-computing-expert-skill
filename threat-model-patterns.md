# TEE Threat Model Patterns

## Security Layer Framework

Based on Blue Throat Labs TEE Security Handbook classification:

| Layer | Description | Requirements |
|-------|-------------|--------------|
| **Layer 1** | TEE only | Hardware isolation, no attestation |
| **Layer 2** | TEE + correct attestation | RCAR, measurement verification, PCK chain |
| **Layer 3** | TEE + attestation + constant-time crypto | Side-channel resistant operations |
| **Layer 4** | TEE + attestation + constant-time + formal verification | Highest assurance |

Most production systems target Layer 2-3. Layer 4 is research territory.

## Common TEE Threats

### T1: Trusting the Host
**Threat:** Assuming hypervisor/cloud operator cannot access TEE memory
**Reality:** TDX/SEV-SNP encrypt VM memory; host sees ciphertext only
**Mitigation:** This is what TEEs solve. Verify attestation proves code runs inside TEE.
**Residual risk:** Side-channel attacks at hardware level (low for VM isolation)

### T2: Unpinned Measurements
**Threat:** Accepting any MRTD/MEASUREMENT without comparing to known-good value
**Impact:** Attacker runs modified firmware/code, attestation still "passes"
**Mitigation:** RVPS stores expected measurements; policy rejects mismatches
**Practical gap:** On Azure, cannot independently derive MRTD from source (opaque build)

### T3: Single-Register Reliance
**Threat:** Only checking MRTD, ignoring RTMRs/PCRs
**Impact:** Runtime tampering undetected (modified kernel cmdline, injected modules)
**Mitigation:** Verify full measurement set: MRTD + RTMR[0-3] + relevant PCRs

### T4: Secret Leakage via Logs/Errors
**Threat:** Seeds, keys, or attestation tokens appearing in logs or error messages
**Mitigation:** Audit all log statements; use generic error responses; never log secret material

### T5: Host-Supplied Randomness
**Threat:** Host provides weak/predictable random numbers to TEE
**Mitigation:** Use CPU hardware RNG (RDRAND/RDSEED on Intel); verify entropy source

### T6: Bloated TCB
**Threat:** Large trusted computing base increases attack surface
**Mitigation:** Minimal static binary; no unnecessary dependencies; Nix reproducible build

### T7: Unverified Certificate Chains
**Threat:** Accepting TDX quotes without validating PCK chain to Intel Root CA
**Impact:** Forged quotes accepted as genuine
**Mitigation:** Full PCK chain verification + CRL check

### T8: Timing / Side-Channel Oracles
**Threat:** Crypto operations leak information via timing differences
**Impact:** Key recovery possible with enough observations
**Mitigation:** Use constant-time crypto libraries; avoid branching on secret data
**Note:** Verifier code (non-secret operations) has lower side-channel risk

### T9: Replay Attacks
**Threat:** Old valid attestation/signature replayed in new context
**Mitigation:** Timestamps in signed data; nonce-based attestation; freshness windows

### T10: Debug Mode Acceptance
**Threat:** Accepting quotes from debug-mode TDs (host can read TD memory)
**Mitigation:** Explicitly check and reject `td_attributes` debug bit

### T11: Protocol Downgrade
**Threat:** Attacker forces use of older, vulnerable TCB version
**Mitigation:** Enforce minimum SVN (Security Version Number) in policy

### T12: Weak Key Derivation
**Threat:** Using raw hash (SHA-256) instead of proper KDF
**Impact:** No domain separation; keys from different contexts may collide
**Mitigation:** HKDF-SHA256 with context binding (purpose, algorithm, version)

## Trust Assumptions (Document These Explicitly)

Every TEE deployment rests on assumptions. Document yours:

1. **Hardware trust:** Intel/AMD silicon correctly implements isolation
2. **Root CA trust:** Intel Root CA / AMD ARK is uncompromised
3. **Cloud provider firmware:** Provider deploys unmodified firmware (OpenHCL, OVMF)
4. **Attestation service honesty:** KBS operator enforces correct policy
5. **Build system integrity:** Nix build environment is not compromised
6. **Time source:** System clock is reasonably accurate (for timestamps)
7. **Cryptographic assumptions:** Ed25519/P-256/SHA-256/HKDF are secure

## Auditor Verification Walkthrough

What an independent auditor can verify end-to-end:

1. **Reproduce the build** — `nix build` produces identical binary hash
2. **Query live attestation** — `GET /attest` returns TDX quote + measurements
3. **Verify hardware authenticity** — PCK certificate chain to Intel Root CA
4. **Verify firmware binding** — HCL envelope ties vTPM to TDX report (Azure)
5. **Verify boot chain** — PCR[9] matches kernel hash from Nix build
6. **Verify application** — PCR[10] / IMA measurement matches binary hash
7. **Verify signature proof** — AppProof -> BootProof -> TDX quote chain is consistent
8. **Check trust assumptions** — Explicitly list what is trusted vs verified

### What Auditors CANNOT Verify (Per Platform)

| Platform | Unverifiable | Why |
|----------|-------------|-----|
| Azure TDX | Binary matches source | OpenHCL build is opaque |
| GCP TDX | Source code | Google OVMF is closed |
| AWS SEV-SNP | Guest image construction | Image tool is closed |

**This is the fundamental limitation of cloud TEEs.** No platform provides full source-to-measurement verifiability. Cross-platform deployment mitigates by placing the trust gap in different parts of the chain.

## Security Hardening Checklist

- [ ] `random.trust_cpu=y` kernel parameter (trust hardware RNG)
- [ ] NTS time synchronization (authenticated NTP)
- [ ] Debug mode quote rejection
- [ ] Pin container/VM image digests (no `:latest` tags)
- [ ] Minimal base image (no shell, no package manager in production)
- [ ] HKDF for all key derivation
- [ ] report_data purpose tagging
- [ ] SVN minimum enforcement
- [ ] CRL checking for PCK chain
- [ ] Generic error responses (no stack traces, no secret material)
- [ ] Constant-time crypto for secret operations
- [ ] Attestation freshness window
