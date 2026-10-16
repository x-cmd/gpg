---
name: 8-sigstore-cosign-and-double-signing
description: Sigstore ecosystem walkthrough (Cosign CLI, Fulcio CA, Rekor transparency log), typical sign-and-verify flow, OIDC identity contract, adoption state, and limitations. Comparison with GPG (mechanism, credential burden, trust root, audit, ecosystem fit, revocation, trust substrate). Pre-institutional trust distinction. Double-signing strategy and when it's worth it.
type: reference + comparison
---

# Core Content

core_features:

- Sigstore: CNCF-graduated project (originally Google, 2021+) providing keyless software signing — short-lived OIDC-bound certificates, public append-only log
- Cosign: the user-facing CLI; signs/verifies container images, blobs, attestations; `--keyless` is the 2026 default
- Fulcio: certificate authority issuing ~10-min X.509 certificates binding OIDC identity to a fresh public key per signing event
- Rekor: append-only Merkle tree of every signing event with inclusion proofs verifiable against published signed checkpoints
- Supporting tools: gitsign (Git commit signing), policy-controller / kyverno (Kubernetes admission), SLSA provenance integration, TUF integration
- Typical sign-and-verify flow (8-step signing, 6-step verifying)
- OIDC identity contract: publisher commits to *who* signs (e.g. https://github.com/org/repo/.github/workflows/file@refs/tags/v1.2.3), verifier pins to that identity — inverse of GPG's "which key signs" contract
- Sigstore vs GPG seven-dimension comparison table (mechanism, credential burden, trust root, audit, ecosystem fit, revocation, trust substrate)
- Trust root analysis for both chains (GPG = math + key + channel; Sigstore = math + certificate + log entry)
- Pre-institutional trust: GPG works for anonymous OSS contributors and pre-incorporation projects; Sigstore requires institutional anchor (GitHub org, etc.)
- Double-signing strategy: when worth it (artifact consumed by both ecosystems), when overkill (single ecosystem), hybrid pipeline notes (GPG first, then Sigstore on post-GPG artifact)
- 2026 adoption: Kubernetes upstream, GitHub CLI, npm, several Linux distros, Sigstore project itself

## Key Information

highlights:

- Sigstore's contract is "who signs" (OIDC identity), GPG's is "which key signs" (key bytes)
- Cosign signs with a fresh ephemeral keypair; private key destroyed after each signing event — no long-term secret to protect
- Rekor inclusion proofs publicly auditable against signed checkpoints
- GPG fingerprint is, in many jurisdictions, admissible evidence of authorship with direct (not mediated) cryptographic link
- Sigstore's institutional prerequisite makes it less suitable for individuals producing OSS without a corporate / org identity
- This isn't "GPG > Sigstore" generally — Sigstore's anchoring is a feature in commercial contexts
- Double-signing order matters: GPG first, then Sigstore on post-GPG artifact (so Rekor covers final state)
- Rekor grows forever; full nodes hold the entire Merkle tree

## Use Cases

use_cases:

- Choosing between GPG, Sigstore, or double-signing for a new release pipeline
- Justifying the choice to a security team or procurement review
- Deciding whether double-signing overhead is worth it for a specific artifact and audience
- Onboarding a team member to the trade-offs between the two paradigms
- Understanding Sigstore internals (Cosign / Fulcio / Rekor) for integration work

## Related Resources

official:
  website: <https://www.sigstore.dev/>
  repo: <https://github.com/sigstore>
related:
  cosign: <https://github.com/sigstore/cosign>
  fulcio: <https://github.com/sigstore/Fulcio>
  rekor: <https://github.com/sigstore/Rekor>
  gitsign: <https://github.com/sigstore/gitsign>
  slsa: <https://slsa.dev/>
  cncf: <https://www.cncf.io/projects/sigstore/>
  rfc4880: <https://datatracker.ietf.org/doc/html/rfc4880>

## Summary

Sigstore and Cosign are the most visible pieces of the 2020s shift from "long-term private key signing" to "short-lived certificate + transparency log" signing, anchored to existing institutional identities (GitHub, Google). Three core components: Cosign (CLI; signs/verifies; `--keyless` is default), Fulcio (CA issuing ~10-min X.509 certificates binding OIDC identity to a fresh public key per signing event), Rekor (append-only Merkle tree of every signing event with inclusion proofs verifiable against published signed checkpoints). Typical sign-and-verify flow: Cosign obtains OIDC token → Fulcio issues certificate → fresh keypair → signs artifact → throws private key away → Rekor logs → returns inclusion proof; verification checks certificate chain → OIDC identity → Rekor inclusion → artifact signature. Sigstore vs GPG seven-dimension comparison: mechanism (long-term key vs short-lived OIDC-bound cert), credential burden (long-term secret vs none), trust root (HTTPS-delivered public key vs OIDC IdP + Rekor), audit (weak vs strong), ecosystem fit (OS package managers vs cloud-native), revocation (manual vs implicit), trust substrate (peer-to-peer / pre-institutional vs institutional / OIDC-anchored). Pre-institutional trust: GPG works for anonymous OSS contributors and pre-incorporation projects because it bottoms out in the consumer's judgment (subjective trust); Sigstore bottoms out in an OIDC IdP. Double-signing worth it when the artifact is consumed by both ecosystems; overkill for single-ecosystem audiences. Hybrid pipeline: GPG first, then Sigstore on the post-GPG artifact; Rekor covers final state.