---
x-title: Sigstore vs GPG, and the double-signing strategy
x-desc: A side-by-side comparison of Sigstore (keyless signing with transparency log) and traditional GPG (long-term key signing with trusted-channel delivery) — covering trust roots, audit properties, ecosystem fit, and when double-signing makes sense as a hybrid strategy. **Project-agnostic; exploratory.**
x-sidebar: Sigstore vs GPG, and double-signing
x-keywords: sigstore, gpg, keyless signing, transparency log, rekor, fulcio, double-signing, dual signing, supply chain, slsa, cosign, rpm
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Sigstore vs GPG, and the double-signing strategy'
      inLanguage: 'en'
      about: 'Sigstore vs GPG comparison and double-signing strategy'
---

# Sigstore vs GPG, and the double-signing strategy

Two distinct signing paradigms dominate software supply-chain
security in 2026. **GPG** (and its predecessor PGP) has
been the de-facto standard for signing software since the
1990s; **Sigstore** is a more recent (2021+) ecosystem
centered on keyless signing backed by an OIDC identity
provider and a public transparency log. They solve
different problems, and a team shipping packages that need to
be consumable across both old and new ecosystems often ends
up considering both.

This article compares the two across mechanism, trust root,
audit properties, and ecosystem fit. It closes with an
analysis of when *double-signing* (signing the same artifact
with both systems) is a reasonable strategy — and when it's
overkill.

> **Status: exploratory.** As of this writing the x-cmd team
> has not adopted either approach for actual release signing.
> The rest of this article is analysis, not a description of
> what is in production today.

## Two paradigms, two threat models

| Dimension                 | GPG (traditional)                                              | Sigstore (keyless)                                                  |
| ---                       | ---                                                            | ---                                                                  |
| **Core mechanism**        | Sign with a long-term private key; verify with the matching public key. | Sign with a short-lived certificate (≈10 min) bound to an OIDC identity; verify via the certificate chain and the transparency log entry. |
| **Credential burden**     | A long-term private key that must be generated, distributed, rotated, and protected (offline laptop, HSM, CI secret, etc.). Leak = historical forgery risk for the entire lifetime of the key. | No long-term key. A fresh keypair is generated per signing event inside a signing process; the private half is destroyed immediately after. No long-term secret to protect. |
| **Trust root**            | The public key, delivered over a trusted channel (HTTPS from a domain the team controls), plus whatever the consumer imports into their keyring. | The OIDC identity provider (GitHub, Google, etc.) that vouches for the signing identity, plus the Sigstore transparency log (Rekor) that records every signing event publicly. |
| **Audit / non-repudiation** | Weak: a signature says "this key signed this artifact", but doesn't tell you where or when in any public, verifiable way. | Strong: every signature is mirrored to Rekor with the OIDC identity, the artifact hash, and a timestamp. Anyone can audit who signed what, when, from which identity. |
| **Ecosystem fit**         | Native to RPM / DEB / Apt / Pacman. Verification is built into OS-level package managers. | Native to OCI container images, Kubernetes admission controllers, npm packages, GitHub Actions attestations. Limited OS-level package manager support as of 2026. |
| **Revocation**            | Possible via the OpenPGP revocation certificate, but distribution of the revocation itself is a manual problem. | Implicit: certificates are short-lived by design; revocation is automatic (no new cert = no new signature). |
| **Trust substrate**      | Peer-to-peer — anyone can sign, anyone can pin; works for individuals and pre-institutional projects. | Institutional — anchored to an OIDC identity provider (GitHub, Google, …); requires being a known entity. |

The two paradigms protect against different things. GPG's
security claim is essentially "the math holds and the
private key stayed private" — a long-term *capability*
guarantee, contingent on operational discipline. Sigstore's
security claim is "this signature was produced by this OIDC
identity, and the entire event is in a public log" — a
short-term *event* guarantee, contingent on the OIDC IdP and
the log remaining trustworthy.

## Trust root analysis

These two paradigms build trust in fundamentally different
ways.

**GPG's trust chain** runs through three layers:

1. The math: signature validity (always true if the key pair
   hasn't been tampered with).
2. The key: the *specific bytes* of the public key the
   consumer imports into their keyring. The fingerprint is
   a 160-bit commitment to those bytes; if you have the
   right bytes, you're cryptographically locked to whoever
   produced them.
3. The channel: how the consumer got those bytes. This is
   where the actual trust sits — `https://x-cmd.com/gpg/`,
   signed release tarballs, a verified-USB curl, etc. As long
   as the channel isn't compromised, the chain holds.

The weakest link is layer 3: an attacker who controls the
delivery channel can substitute a different public key, and
the consumer's `gpg --verify` will succeed against the
attacker's key. Defense is *trust the channel* + *fingerprint
pinning* (cross-check the imported fingerprint against an
independent reference).

**Sigstore's trust chain** runs through a different three
layers:

1. The math: signature validity against the short-lived
   certificate.
2. The certificate: a Fulcio-issued x.509 cert binding an
   OIDC identity (e.g. `x-cmd/x-cmd` repo on GitHub) to a
   public key generated for this one signing event.
3. The log entry: a Rekor record committing the certificate,
   the artifact hash, and the OIDC identity to a Merkle tree
   whose root is published periodically.

The weakest link is layer 2: Sigstore's trust depends on the
OIDC identity provider (GitHub, Google, etc.) being honest
about who they're issuing identity tokens to. If your
GitHub org is compromised, an attacker can mint signing
certificates as you. Defense is *hardening the OIDC identity*
(GitHub org security, branch protection, 2FA on maintainers,
etc.) — which is the same operational discipline GPG
requires, just on a different surface.

## Subjective trust: GPG as a pre-institutional mechanism

GPG's design has a property that's easy to overlook in
technical comparisons: it's a *pre-institutional* trust
mechanism. Two parties can establish cryptographic,
court-admissible proof of identity and agreement before
either of them is a person, a legal entity, or an
incorporated organization.

This works because GPG's trust chain is *peer-to-peer* —
you choose to trust a specific key because you decided to,
not because an institution vouched for it. The chain
bottoms out in your own judgment (subjective trust), and
once that judgment is recorded (e.g. you imported the key
into your keyring), the mathematics takes over.

Sigstore's design, by contrast, bottoms out in an *OIDC
IdP* — typically GitHub, Google, or Microsoft. These are
institutional actors with legal personhood. A signing
identity in Sigstore is *someone the IdP has agreed to
vouch for*. That's useful in most commercial contexts,
but it has a prerequisite: the IdP must be willing to issue
you an identity token, which usually requires you to have
an account, a payment method, an organization, or some
other institutional anchor.

Concretely:

- **Anonymous OSS contributors** can sign their code with
  GPG, and downstream consumers can pin their fingerprint,
  without anyone having to be a person or a company. The
  trust is pure cryptographic — no institution mediates.
- The same contributor probably can't use Sigstore the same
  way, because they don't have a GitHub org or equivalent
  institutional anchor to bind their signing identity to.

This is a meaningful *legal* property, not just a technical
one. The fingerprint is, in many jurisdictions, *admissible
evidence* that a particular document or artifact was signed
by the holder of the matching private key. The trust chain
is direct rather than mediated, which some legal frameworks
treat as stronger evidence of authorship than an
institutional attestation that depends on a third party
remaining trustworthy and accountable. (Jurisdiction-
dependent; this is not legal advice, but it's a real
consideration for some compliance regimes.)

Practical consequences:

- **Pre-incorporation projects.** A project that hasn't
  incorporated yet can ship signed artifacts with GPG. The
  same project with Sigstore typically needs an org account
  on GitHub or another IdP, which usually requires being a
  legal person or having one as a sponsor.
- **Cross-jurisdictional scenarios.** GPG keys travel
  across borders without registry liability. The legal
  weight of a signature depends on which jurisdiction's law
  applies, which can be negotiated contractually without
  anyone being the institutional mediator.
- **Court-admissible proof.** A properly-pinned GPG
  signature is, in some jurisdictions, stronger evidence of
  authorship than an institutional attestation — because
  the cryptographic link is direct rather than mediated.
- **Sigstore's institutional prerequisite.** Sigstore is
  excellent for software produced *by* a company or org
  with an institutional identity, and less suitable for
  the long tail of OSS produced by individuals who don't
  have or want one.

This isn't a reason to prefer GPG over Sigstore in
general — Sigstore's institutional anchoring is a feature
in most commercial contexts. It's a reason to think about
*what kind of trust you actually need* before picking one,
and to recognize that the two paradigms optimize for
different scenarios.

## Use-case boundaries

**GPG is the right choice when:**

- The artifact is consumed by an OS-level package manager
  (`dnf`, `yum`, `apt`, `zypper`, `pacman`). These have
  native OpenPGP verification built in and will refuse to
  install unsigned RPMs / DEBs by default.
- The ecosystem is conservative — RHEL, CentOS, Rocky,
  Debian, Ubuntu LTS, SUSE, Alpine, embedded Linux distros.
- Consumers are sysadmins running legacy infrastructure
  where Sigstore verification tooling isn't available.

**Sigstore is the right choice when:**

- The artifact is a container image, a Kubernetes resource,
  or a software supply-chain attestation (SLSA / in-toto).
- The consumer has Kubernetes admission controllers or
  CI/CD systems that already integrate Sigstore
  verification (`cosign verify`, `kyverno`, `policy
  controllers`, etc.).
- The team wants verifiable public audit trails of who
  signed what and when — without committing to long-term
  key management discipline.

**Neither alone is the right choice when:**

- The artifact needs to be consumable across both old
  (OS-level) and new (cloud-native) ecosystems. See "double-
  signing" below.

## What is double-signing?

**Double-signing** (sometimes called dual-signing) means
producing two independent signatures on the same artifact:
one in each paradigm. A consumer verifies the signature
matching their toolchain; consumers in the other ecosystem verify
theirs; no consumer has to change their tooling to accept
the artifact.

For an RPM, a double-signing pipeline typically looks like:

1. Build the `.rpm` (unsigned).
2. **GPG layer.** `rpmsign --addsign package.rpm` — embeds an
   OpenPGP signature header into the RPM metadata. Now
   `rpm -K` succeeds and `dnf install` accepts it.
3. **Sigstore layer.** `cosign sign-blob package.rpm` (or
   `cosign sign` for an OCI image) — produces a detached
   signature (`sig`) and certificate (`cert`) referencing a
   Rekor log entry. Now `cosign verify-blob` succeeds.
4. Publish the artifact, the cosign `sig` + `cert` + bundle,
   and the public GPG key (in the team's
   `keyring/keyring.asc` for OS-level consumers).

Both signatures are independently verifiable; neither
needs the other to work; a consumer in either ecosystem
gets the same artifact with the same trust.

## When double-signing is worth it

Double-signing is worth the extra CI / storage cost when:

- The artifact is consumed by **both** ecosystems (OS-level
  package managers *and* cloud-native supply chain tooling).
- The team can absorb the operational overhead: signing
  twice in CI, publishing two sets of artifacts (RPM
  signature header embedded in the package + cosign `sig` /
  `cert` / bundle alongside), documenting two verification
  paths.
- The team's threat model has both old and new attack
  surfaces — i.e. it cares about both legacy sysadmin
  pipelines (where GPG is required) and modern
  cloud-attestation pipelines (where Sigstore is expected).

Concrete teams/projects where this has worked: Kubernetes
upstream (signs container images with both Docker Content
Trust and cosign, plus the source tarballs with GPG);
several Linux distributions shipping cloud-native artifacts
alongside their OS packages; the Sigstore project itself
(which signs its own binaries with both cosign and GPG).

## When double-signing is overkill

Double-signing is **not** worth the overhead when:

- The audience is exclusively one ecosystem. If your
  consumers are all `dnf`-based, adding cosign signatures
  is cost without benefit. If they're all in a Sigstore-
  verifying Kubernetes cluster, the GPG layer is dead
  weight.
- The artifact isn't reused across contexts. A script that
  only runs in CI doesn't need both signature types.
- The team can't afford the doubled CI complexity. Two
  signatures means two key-management stories, two failure
  modes, two docs. If operational simplicity matters more
  than ecosystem reach, pick one and ship.

## Hybrid pipelines — practical notes

If double-signing is adopted, a few practical considerations:

- **Order of operations matters.** Sign with GPG first
  (modifies the RPM header); then sign the *resulting* RPM
  with cosign. Reversing the order means cosign would be
  signing the pre-GPG state and the GPG signature wouldn't
  be covered by the transparency log.
- **Transparency log coverage.** Rekor records the artifact
  hash as of the moment of signing. The cosign signature
  covers the post-GPG RPM. Consumers using only cosign
  verification are pinning the post-GPG state; that's the
  intended design.
- **Key rotation cost.** The GPG layer still needs a key
  rotation story (see
  [4. Annual key strategy — design exploration](./4-annual-key-strategy-explained.md)).
  The Sigstore layer rotates by itself (certificates are
  short-lived). The two layers have independent operational
  cadences.
- **Documentation.** The team has to maintain documentation
  for two verification paths. The README / docs need both
  `rpm -K` and `cosign verify-blob` recipes. Adding a
  third-party consumer (e.g. `slsa-verifier`) sometimes
  pulls in yet another verification path; this can compound.

## What to read next

- [4. Annual key strategy — design exploration](./4-annual-key-strategy-explained.md) —
  long-term GPG key rotation trade-offs; pairs with the
  Sigstore layer if you adopt double-signing.
- [7. Signing an RPM with GPG](./7-signing-an-rpm-with-gpg.md) —
  the `rpmsign --addsign` step that the GPG layer of a
  double-signing pipeline would invoke.
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — release-side
  pipeline conventions once a signing strategy has been
  picked.

## Sources

- [Sigstore project documentation](https://docs.sigstore.dev/) —
  Fulcio (CA), Rekor (transparency log), cosign (CLI).
- [RFC 4880 — OpenPGP Message Format](https://datatracker.ietf.org/doc/html/rfc4880)
- [Sigstore: TBD (2023+) — software-signing landscape](https://blog.sigstore.dev/)