---
name: 10-cosign-and-sigstore-explained
description: Sigstore ecosystem walkthrough — Cosign CLI, Fulcio (CA issuing short-lived OIDC-bound certificates), Rekor (transparency log), typical sign-and-verify flow, OIDC identity contract, adoption, limitations.
type: reference
---

# Core Content

core_features:

- Sigstore: CNCF-graduated project (originally Google, 2021+) providing keyless software signing — short-lived OIDC-bound certificates, public append-only log
- Cosign: the user-facing CLI; signs/verifies container images, blobs, attestations; `--keyless` is the 2026 default
- Fulcio: the certificate authority; issues ~10-min X.509 certificates binding OIDC identity (GitHub Actions, etc.) to a fresh public key per signing event
- Rekor: append-only Merkle tree of every signing event; inclusion proofs verifiable against published checkpoints
- Three-component flow: signing (Cosign obtains token → certificate issues → cosign signs with fresh key → Rekor logs) and verification (Cosign checks certificate chain → OIDC identity → Rekor inclusion → artifact signature)
- OIDC identity contract: publisher commits to *who* signs (the OIDC identity like https://github.com/org/repo/.github/workflows/file@refs/tags/v1.2.3); verifier pins to that identity; inverse of GPG's "which key signs" contract
- Supporting tools: gitsign (Git commit signing), policy-controller / kyverno (Kubernetes admission), SLSA provenance integration, TUF integration
- Adoption: Kubernetes upstream, GitHub CLI, npm, several Linux distros, Sigstore project itself
- Limitations: OIDC IdP reliability is the trust anchor (GitHub org security matters), Rekor grows forever, ~10-min verification windows, no OS-package-manager support yet, requires institutional anchor (no anonymous signers)

## Key Information

highlights:

- Sigstore's contract is "who signs", not "which key signs" — GPG is the inverse
- The signing private key is destroyed after each Cosign sign event — no long-term secret to protect
- Rekor inclusion proofs are publicly auditable against signed checkpoints rooted at a published URL
- OIDC token includes URL-shaped "identity" claims for CI signers (e.g. GitHub Actions workflow + repo + ref)
- Fulcio certificates expire in ~10 minutes; signatures are self-attesting at moment of signing, but ongoing validity requires checking the transparency log
- Trade-off: Sigstore removes long-term key management, but adds dependency on OIDC IdP (GitHub / Google / etc.) as the institutional anchor
- Pre-institutional limitation: anonymous OSS contributors who don't have a GitHub org or equivalent can't use Sigstore the way they'd use GPG

## Use Cases

use_cases:

- Signing container images (OCI) and verifying via Kubernetes admission controllers
- Signing and verifying SLSA provenance attestations
- Signing Git commits (gitsign) without managing long-term GPG keys
- Adopting keyless signing in CI/CD pipelines (no secret management for signing keys)
- Auditing who signed what and when, via Rekor inclusion proofs

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

## Summary

Sigstore and Cosign are the most visible pieces of the 2020s shift from "long-term private key signing" to "short-lived certificate + transparency log" signing, anchored to existing institutional identities (GitHub, Google). Three core components: Cosign (the user-facing CLI; signs/verifies container images, blobs, attestations; `--keyless` is the 2026 default), Fulcio (the certificate authority issuing ~10-min X.509 certificates binding OIDC identity to a fresh public key per signing event), and Rekor (the append-only Merkle tree of every signing event with inclusion proofs verifiable against published signed checkpoints). Signing flow: Cosign obtains OIDC token from IdP → Fulcio issues certificate → Cosign generates fresh keypair → signs artifact → throws private key away → submits {signature, certificate, artifact-hash} to Rekor → Rekor returns inclusion proof. Verification flow: check certificate chain → check OIDC identity matches expected → check Rekor inclusion proof → verify artifact signature against certificate's public key. The OIDC identity contract: publisher commits to *who* signs (an OIDC identity like `https://github.com/org/repo/.github/workflows/file@refs/tags/v1.2.3`), verifier pins to that identity — inverse of GPG's "which key signs" contract. Adoption (2026): Kubernetes upstream, GitHub CLI, npm, several Linux distros, Sigstore project itself. Limitations: OIDC IdP reliability is the trust anchor (GitHub org security matters); Rekor grows forever; ~10-min certificate verification windows; no OS-package-manager support yet; requires institutional anchor (no anonymous signers).