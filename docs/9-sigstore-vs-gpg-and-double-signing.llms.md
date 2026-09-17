---
name: 9-sigstore-vs-gpg-and-double-signing
description: Sigstore vs GPG comparison — mechanism, credential burden, trust root, audit properties, ecosystem fit, trust substrate. When double-signing (GPG + Sigstore on the same artifact) makes sense as a hybrid strategy, and when it's overkill.
type: comparison
---

# Core Content

core_features:

- Two paradigms: GPG (long-term key signing, trusted-channel delivery) vs Sigstore (keyless signing with short-lived certificates, OIDC identity, transparency log)
- Six dimensions compared in a table:
  - Mechanism: long-term private key signing vs short-lived certificate signing
  - Credential burden: protect long-term secret vs no long-term secret
  - Trust root: HTTPS-delivered public key vs OIDC IdP + Rekor transparency log
  - Audit / non-repudiation: weak (no public audit trail) vs strong (Rekor records every signing event)
  - Ecosystem fit: OS package managers (RPM / DEB / apt) vs cloud-native (OCI / Kubernetes / GitHub Actions)
  - Revocation: manual distribution of revocation cert vs implicit (no new cert = no new signature)
  - Trust substrate: peer-to-peer / pre-institutional vs institutional (OIDC-anchored)
- Pre-institutional trust: GPG works for anonymous OSS contributors and pre-incorporation projects; Sigstore requires an institutional anchor (GPR org, etc.)
- Double-signing strategy: when worth it (artifact consumed by both ecosystems, team can absorb doubled CI cost) vs when overkill (single ecosystem audience, single context)
- Hybrid pipeline notes: order matters (GPG first, then Sigstore on post-GPG RPM), Rekor covers post-GPG state, independent operational cadences for key rotation

## Key Information

highlights:

- GPG trust chain bottoms out in the consumer's own judgment (subjective trust); Sigstore trust chain bottoms out in the OIDC IdP
- GPG fingerprint is, in many jurisdictions, admissible evidence of authorship — direct cryptographic link, not mediated by a third party
- Sigstore's institutional prerequisite makes it less suitable for individuals producing OSS without a corporate / org identity
- This is not "GPG > Sigstore" generally — Sigstore's institutional anchoring is a feature in commercial contexts
- Both signing systems require operational discipline, just on different surfaces (private-key hygiene vs OIDC identity hardening)
- Double-signing's order matters: GPG first (modifies RPM header), then cosign on the post-GPG artifact (so Rekor covers the final state)
- Real-world teams using double-signing: Kubernetes upstream, several Linux distros shipping cloud-native artifacts alongside OS packages, the Sigstore project itself

## Use Cases

use_cases:

- Choosing between GPG, Sigstore, or double-signing for a new release pipeline
- Justifying the choice to a security team or procurement review
- Deciding whether double-signing overhead is worth it for a specific artifact and audience
- Onboarding a team member to the trade-offs between the two paradigms

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  sigstore: <https://docs.sigstore.dev/>
  fulcio: <https://github.com/sigstore/Fulcio>
  rekor: <https://github.com/sigstore/Rekor>
  cosign: <https://github.com/sigstore/cosign>
  rfc4880: <https://datatracker.ietf.org/doc/html/rfc4880>

## Summary

Side-by-side comparison of Sigstore (keyless signing with transparency log) and traditional GPG (long-term key signing with trusted-channel delivery), plus analysis of when double-signing both is a reasonable hybrid. Six dimensions compared: mechanism (long-term private key vs short-lived OIDC-bound certificate), credential burden (protect long-term secret vs no long-term secret), trust root (HTTPS-delivered public key vs OIDC IdP + Rekor transparency log), audit / non-repudiation (weak / no public trail vs strong / every signing event publicly recorded), ecosystem fit (OS package managers vs cloud-native supply chain), revocation (manual vs implicit), trust substrate (peer-to-peer / pre-institutional vs institutional / OIDC-anchored). Pre-institutional trust analysis: GPG works for anonymous OSS contributors and pre-incorporation projects because it bottoms out in the consumer's own judgment (subjective trust); Sigstore bottoms out in an OIDC IdP and requires an institutional anchor (GitHub org, etc.) — GPG fingerprint is, in many jurisdictions, admissible evidence of authorship with a direct (not mediated) cryptographic link; Sigstore's institutional prerequisite makes it less suitable for individuals producing OSS without a corporate / org identity. This isn't "GPG > Sigstore" generally — Sigstore's anchoring is a feature in commercial contexts, just a different optimization. Double-signing worth it when the artifact is consumed by both ecosystems and the team can absorb doubled CI cost; overkill when audience is single-ecosystem or artifact doesn't cross contexts. Hybrid pipeline notes: order matters (GPG first, then Sigstore on post-GPG RPM so Rekor covers final state), independent operational cadences for key rotation (GPG layer still needs rotation story; Sigstore layer rotates by itself).