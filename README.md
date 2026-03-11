# Confidential Computing Expert Skill

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that provides domain expertise for TEE-based systems: Intel TDX, AMD SEV-SNP, attestation protocols, cloud confidential VM deployments, and threat modeling.

## What's included

| File | Topic |
|------|-------|
| `SKILL.md` | Skill router — triggers on TEE/attestation topics, routes to reference files |
| `tee-fundamentals.md` | TDX/SEV-SNP hardware, measurement registers, key derivation, trust model |
| `cloud-tee-comparison.md` | Azure vs GCP vs AWS tradeoffs, unverifiable links, platform gotchas |
| `attestation-protocols.md` | RCAR protocol, Trustee KBS, PCK verification, proof chains |
| `threat-model-patterns.md` | 12 threat patterns, security layers, auditor walkthrough, hardening checklist |

## Install

Clone into your Claude Code skills directory:

```bash
git clone git@github.com:vkobel/confidential-computing-expert-skill.git \
  ~/.claude/skills/confidential-computing-expert
```

The skill activates automatically when Claude Code detects TEE, attestation, or confidential computing topics.
