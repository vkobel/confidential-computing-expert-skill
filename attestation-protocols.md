# Attestation Protocols & Verification

## RCAR Protocol (Remote Configuration And Reporting)

Standard attestation handshake used by Trustee KBS:

```
Client (CVM)                          KBS (Verifier)
    |                                      |
    |--- POST /kbs/v0/auth (tee_type) ---->|
    |<-- Challenge (nonce) ----------------|
    |                                      |
    |  [Generate TDX quote with nonce      |
    |   in report_data]                    |
    |                                      |
    |--- POST /kbs/v0/attest ------------>|
    |    { evidence: <quote>,              |
    |      tee_pubkey: <ephemeral_key> }   |
    |<-- { token: <attestation_token> } ---|
    |                                      |
    |--- GET /kbs/v0/resource/... -------->|
    |    Authorization: Bearer <token>     |
    |<-- { secret: <JWE_encrypted> } ------|
    |                                      |
    |  [Decrypt with ephemeral key]        |
```

### Key Properties
- **Nonce freshness:** KBS provides nonce, client includes in quote's report_data
- **Key binding:** Client generates ephemeral keypair, public key goes in attestation evidence
- **JWE encryption:** Secrets encrypted to client's ephemeral key — only attested client can decrypt
- **Stateless verification:** Each attestation is independent (no session state)

## Trustee KBS Architecture

```
Trustee KBS
  ├── Attestation Service (AS)
  │     ├── TDX Verifier (DCAP-based)
  │     ├── SEV-SNP Verifier
  │     └── Policy Engine (OPA/Rego)
  ├── Reference Value Provider Service (RVPS)
  │     └── Stores expected measurements
  │         (MRTD, RTMRs, PCRs, binary hashes)
  └── Secret Store
        └── KV store for secrets (seed, config)
```

### Policy Enforcement
RVPS stores reference values. Policy engine compares:
- MRTD matches expected firmware measurement
- RTMR values match expected boot chain
- PCR values match expected configuration
- `td_attributes` debug bit is not set

## PCK Certificate Chain Verification

Intel's hardware root of trust for TDX:

```
Intel Root CA (hardcoded, self-signed)
  └── Intel Intermediate CA (Platform CA)
       └── PCK Certificate (per-platform, per-TCB)
            └── Signs TDX Quote
```

### Verification Steps
1. Parse quote, extract signature and certification data
2. Verify PCK certificate signature chains to Intel Root CA
3. Check certificate revocation (CRL from Intel PCS)
4. Verify TCB level meets minimum (SVN checks)
5. Verify quote signature using PCK public key
6. Extract and validate report body (MRTD, RTMRs, report_data)

### TCB Recovery / SVN Checking
- Each TDX component has a Security Version Number (SVN)
- Quote contains `tee_tcb_svn` — array of component SVNs
- Verifier should enforce minimum SVN to prevent downgrade attacks
- TCB recovery events (microcode updates) change expected SVNs

## HCL Envelope (Azure-Specific)

Azure's mechanism to bind vTPM attestation to TDX hardware:

```
TDX Hardware Quote
  report_data = SHA-256(vTPM Attestation Key public key)
  ↕ (binding)
vTPM Quote
  Signed by Attestation Key (AK)
  Contains PCR values for boot chain
  ↕
HCL Envelope
  Proves AK was generated inside TD
  Binds vTPM identity to TDX identity
```

### Verification
1. Verify TDX quote (PCK chain)
2. Extract `report_data` from TDX quote
3. Verify `report_data` == SHA-256(AK public key)
4. Verify vTPM quote signature with AK
5. Now PCR values from vTPM are hardware-rooted

## Signature Proof Chain Pattern

For signing services, bind the full attestation chain into each signature:

```
SignatureProof {
  message_hash: SHA-256(original_message),
  signature: Ed25519(message, derived_key),
  public_key: derived_public_key,
  timestamp: ISO-8601,

  app_proof: {
    key_derivation: "HKDF-SHA256(seed, context)",
    attestation_binding: SHA-256(public_key) == report_data,
  },

  boot_proof: {
    tee_type: "TDX",
    measurements: { mrtd, rtmr0, rtmr1, rtmr2, rtmr3 },
    pcr_values: { pcr9: kernel_hash, pcr10: app_hash },
  },

  hardware_proof: {
    quote: <base64_tdx_quote>,
    certificate_chain: [pck_cert, intermediate, root],
  }
}
```

### Verification Order (outside-in)
1. Verify Intel PCK chain (hardware authenticity)
2. Verify TDX quote signature (hardware integrity)
3. Verify measurements match expected values (correct software)
4. Verify report_data binds to signing key (key is from TEE)
5. Verify signature on message (data integrity)

## Replay Protection

### Threats
- Attacker replays old valid signature for different context
- Attacker replays old attestation quote after TEE compromised

### Mitigations
- **Timestamps in signed data:** Include ISO-8601 timestamp in signature payload
- **Nonce in attestation:** Fresh nonce in report_data for each attestation session
- **Quote freshness window:** Verifier rejects quotes older than N minutes
- **Monotonic counters:** If available, include counter in signed data

## Nitro Enclaves Attestation

```
Client (Enclave)                        Verifier (or KMS)
    |                                        |
    |  [Generate attestation doc via NSM     |
    |   with user_data = nonce/pubkey hash]  |
    |                                        |
    |--- Attestation document (COSE Sign1) ->|
    |                                        |
    |    [Verify COSE signature              |
    |     Verify cert chain to AWS root CA   |
    |     Check PCR0/1/2 against allowlist   |
    |     Optionally check PCR3 (IAM role)]  |
    |                                        |
    |<-- Secret (encrypted to enclave key) --|
```

### Nitro PCR Pinning Strategy
- **PCR0 + PCR1 + PCR2:** Full code identity (EIF + kernel + app). Pin all three for strict verification.
- **PCR2 only:** App identity. Allows kernel updates without re-pinning.
- **PCR3:** IAM role binding. Use to restrict which AWS accounts/roles can run the enclave.
- **PCR4:** Instance ID. Rarely pinned (changes per instance), but useful for audit trails.

### AWS KMS Integration
KMS natively understands Nitro attestation — you can set KMS key policies requiring specific PCR values. The enclave calls `kms:Decrypt` with its attestation document; KMS verifies internally before releasing the key.

## Common Attestation Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not checking debug attribute | Attestation meaningless (host can read TD) | Check `td_attributes[0] & 0x01 == 0` |
| Trusting MRTD alone | Missing runtime tampering | Also verify RTMRs / PCRs |
| No SVN minimum | Downgrade to vulnerable TCB | Enforce minimum `tee_tcb_svn` |
| Raw SHA-256 for key derivation | No domain separation | Use HKDF with context |
| report_data without purpose prefix | Cross-context key confusion | Tag: `"purpose:" \|\| hash` |
| Skipping CRL check | Revoked platform still trusted | Check Intel CRL |
| Hardcoded PCK root | Root CA rotation breaks verification | Fetch from Intel PCS, pin fingerprint |
| Pinning only PCR0 (Nitro) | Can't distinguish kernel/app changes | Pin PCR0 + PCR1 + PCR2 |
| Not verifying Nitro root CA | Forged attestation docs accepted | Validate cert chain to AWS Nitro root CA |
| Secrets in EIF/container image | Keys extractable from ramdisk cpio | Fetch secrets at runtime via KMS after attestation |
