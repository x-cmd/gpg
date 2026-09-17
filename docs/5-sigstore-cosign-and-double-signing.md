---
x-title: Sigstore, Cosign, and double-signing
x-desc: What Sigstore is (the project), what Cosign is (the CLI), how Fulcio (CA) and Rekor (transparency log) fit together — and the comparison with GPG, the pre-institutional trust distinction, the double-signing strategy, and when double-signing is worth it. **Project-agnostic; exploratory.**
x-sidebar: Sigstore, Cosign, and double-signing
x-keywords: sigstore, cosign, fulcio, rekor, transparency log, keyless signing, oci, container, slsa, supply chain, gpg vs sigstore, double-signing, dual signing, cncf
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Sigstore, Cosign, and double-signing'
      inLanguage: 'en'
      about: 'Sigstore / Cosign ecosystem and GPG comparison'
---

# Sigstore, Cosign, and double-signing

Two distinct signing paradigms dominate software supply-chain
security in 2026. **GPG** has been the de-facto standard for
signing software since the 1990s; **Sigstore** is a more
recent (2021+) ecosystem centered on *keyless* signing
backed by an OIDC identity provider and a public
transparency log. This article covers Sigstore and Cosign
in depth, then compares the two paradigms and analyzes
when *double-signing* both is a reasonable hybrid strategy.

> **Status: exploratory.** As of this writing the x-cmd team
> has not adopted either approach for actual release signing.

## One-paragraph summary

**Sigstore** is a project (originally from Google, now a
CNCF graduated project under the Linux Foundation) that
provides *keyless* software signing — short-lived
certificates bound to OIDC identities, recorded in a public
append-only log, with verification tools. **Cosign** is
Sigstore's CLI tool for signing and verifying container
images, blobs, attestations, and similar artifacts.
**Fulcio** is the certificate authority that issues the
short-lived signing certificates. **Rekor** is the
transparency log that records every signing event
publicly. Together they let a team publish signed artifacts
*without managing any long-term private key* — the
trade-off is that signing identity is anchored to an OIDC
identity provider (typically GitHub or Google), not to a
key the team holds.

## The three pieces

### Cosign — the CLI

Cosign is the user-facing tool. It does two things: **sign**
(produce a signature, a certificate, and an entry in the
transparency log) and **verify** (check that a signature was
produced by a particular identity and is recorded in
Rekor).

Typical commands:

```sh
# Sign a container image (most common use case)
cosign sign --keyless ghcr.io/example/app:v1.2.3

# Sign an arbitrary blob (e.g. an RPM)
cosign sign-blob --output-signature sig \
  --output-certificate cert \
  package.rpm

# Verify a container image
cosign verify --keyless ghcr.io/example/app:v1.2.3

# Verify a blob
cosign verify-blob --signature sig \
  --certificate cert \
  --certificate-identity github \
  --certificate-oidc-issuer https://github.com/login/oauth \
  package.rpm
```

`--keyless` is the default in 2026 — the whole point of
Sigstore is that you don't need a long-term private key.

### Fulcio — the certificate authority

Fulcio is the CA that issues the short-lived signing
certificates. When you run `cosign sign --keyless`:

1. Cosign asks your OIDC identity provider (typically
   GitHub Actions or `gh auth login`) for an OIDC identity
   token — a signed JWT asserting "I am user X in org Y".
2. Cosign sends that token to Fulcio.
3. Fulcio verifies the OIDC token, then issues an X.509
   signing certificate binding the OIDC identity
   (e.g. `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`)
   to a fresh public key generated for this signing event.
4. The certificate has a ~10 minute expiry (Fulcio is
   configured with short certificate lifetimes by default).
5. Cosign uses the matching private key to sign the
   artifact, then throws the private key away.

The signed artifact, the signature, and the certificate
are then sent to Rekor for transparency logging.

### Rekor — the transparency log

Rekor is an append-only, hash-linked Merkle tree of every
signing event the Sigstore network has processed. Every
Cosign signing event bundles the signature, certificate,
and artifact hash and submits them as a Rekor entry;
Rekor returns an inclusion proof tying the entry to the
Merkle root.

Practical consequences:

- Every signing event is **public** and **tamper-evident**.
- Rekor grows forever; full nodes hold the entire tree.
- Rekor is operated by the Sigstore project; clients talk
  to a public instance at `rekor.sigstore.dev` but can
  also run their own.

### Supporting tools

A handful of other tools round out the ecosystem:

- **gitsign** — signs Git commits using Sigstore (Cosign +
  Fulcio + Rekor) instead of GPG. A drop-in replacement
  for `git commit -S`.
- **policy-controller / kyverno integration** — Kubernetes
  admission webhooks that reject every Cosign-signed image
  whose signature doesn't verify against the configured
  identity.
- **SLSA provenance** — Sigstore tools commonly sign and
  verify SLSA provenance attestations (in-toto) alongside
  artifact signatures.
- **The Update Framework (TUF) integration** — TUF-signed
  metadata can be signed with Cosign.

## Typical sign-and-verify flow

### Signing (publisher side)

```
1. CI build runs `cosign sign-blob package.rpm`
2. Cosign obtains an OIDC token from the IdP (e.g. GitHub
   Actions exposes `$ACTIONS_ID_TOKEN_REQUEST_TOKEN`)
3. Cosign sends the token to Fulcio
4. Fulcio returns a short-lived X.509 certificate bound to
   the OIDC identity
5. Cosign generates a fresh keypair, signs the artifact
   with the private half, throws the private half away
6. Cosign submits {signature, certificate, artifact-hash}
   to Rekor
7. Rekor returns an inclusion proof
8. Cosign writes the signature, certificate, and Rekor
   bundle alongside the artifact (or attaches them in the
   container registry, in the case of OCI images)
```

### Verifying (consumer side)

```
1. Consumer pulls the artifact + signature + certificate +
   bundle
2. Consumer runs `cosign verify-blob` (or `cosign verify`
   for OCI)
3. Cosign checks the certificate chain: signed by Fulcio,
   within the ~10 minute validity window
4. Cosign checks the OIDC identity in the certificate:
   matches the expected identity
5. Cosign checks the Rekor inclusion proof: the signature
   was logged, and the bundle's Merkle root matches the
   signed checkpoint
6. Cosign verifies the artifact signature against the
   certificate's public key
```

Every step is independently checkable.

## Sigstore vs GPG

| Dimension                 | GPG (traditional)                                              | Sigstore (keyless)                                                  |
| ---                       | ---                                                            | ---                                                                  |
| **Core mechanism**        | Sign with a long-term private key; verify with the matching public key. | Sign with a short-lived certificate (~10 min) bound to an OIDC identity; verify via the certificate chain and the transparency log entry. |
| **Credential burden**     | A long-term private key that must be generated, distributed, rotated, and protected. Leak = historical forgery risk for the entire lifetime of the key. | No long-term key. A fresh keypair is generated per signing event inside the signing process; the private half is destroyed immediately after. |
| **Trust root**            | The public key, delivered over a trusted channel (HTTPS from a team-controlled domain), plus whatever the consumer imports into their keyring. | The OIDC identity provider (GitHub, Google, etc.) that vouches for the signing identity, plus the Sigstore transparency log (Rekor) that records every signing event publicly. |
| **Audit / non-repudiation** | Weak: a signature says "this key signed this artifact", but doesn't tell you where or when in any public, verifiable way. | Strong: every signature is mirrored to Rekor with the OIDC identity, the artifact hash, and a timestamp. Anyone can audit who signed what, when, from which identity. |
| **Ecosystem fit**         | Native to RPM / DEB / Apt / Pacman. Verification is built into OS-level package managers. | Native to OCI container images, Kubernetes admission controllers, npm packages, GitHub Actions attestations. Limited OS-level package manager support as of 2026. |
| **Revocation**            | Possible via the OpenPGP revocation certificate, but distribution of the revocation itself is a manual problem. | Implicit: certificates are short-lived by design; revocation is automatic (no new cert = no new signature). |
| **Trust substrate**      | Peer-to-peer — anyone can sign, anyone can pin; works for individuals and pre-institutional projects. | Institutional — anchored to an OIDC identity provider (GitHub, Google, …); requires being a known entity. |

### Trust root analysis

**GPG's trust chain** runs through three layers:

1. The math: signature validity (always true if the key pair
   hasn't been tampered with).
2. The key: the *specific bytes* of the public key the
   consumer imports. The fingerprint is a 160-bit
   commitment; if you have the right bytes, you're
   cryptographically locked to whoever produced them.
3. The channel: how the consumer got those bytes. The actual
   trust sits — `https://x-cmd.com/gpg/`, signed release
   tarballs, a verified-USB curl. Defense: trust the channel
+ fingerprint pinning.

**Sigstore's trust chain** runs through:

1. The math: signature validity against the short-lived
   certificate.
2. The certificate: a Fulcio-issued x.509 cert binding an
   OIDC identity to a public key generated for this one
   signing event.
3. The log entry: a Rekor record committing the
   certificate, the artifact hash, and the OIDC identity
   to a Merkle tree whose root is published periodically.

### Subjective trust: GPG as a pre-institutional mechanism

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
but it has a prerequisite: the IdP must be willing to
issue you an identity token, which usually requires you
to have an account, a payment method, an organization,
or some other institutional anchor.

Concretely:

- **Anonymous OSS contributors** can sign their code with
  GPG, and downstream consumers can pin their fingerprint,
  without anyone having to be a person or a company.
- The same contributor probably can't use Sigstore the same
  way, because they don't have a GitHub org or equivalent
  institutional anchor to bind their signing identity to.

This is a meaningful *legal* property too. The fingerprint
is, in many jurisdictions, *admissible evidence* that a
particular document or artifact was signed by the holder
of the matching private key. The trust chain is direct
rather than mediated, which some legal frameworks treat
as stronger evidence of authorship than an institutional
attestation that depends on a third party remaining
trustworthy and accountable. (Jurisdiction-dependent; this
is not legal advice.)

This isn't "GPG > Sigstore" generally — Sigstore's
institutional anchoring is a feature in commercial
contexts. It's a reason to think about *what kind of
trust you actually need* before picking one, and to
recognize that the two paradigms optimize for different
scenarios.

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
- The signer doesn't have (or doesn't want) a corporate /
  org identity.

**Sigstore is the right choice when:**

- The artifact is a container image, a Kubernetes resource,
  or a software supply-chain attestation (SLSA / in-toto).
- The consumer has Kubernetes admission controllers or
  CI/CD systems that already integrate Sigstore
  verification (`cosign verify`, `kyverno`, policy
  controllers, etc.).
- The team wants verifiable public audit trails of who
  signed what and when — without committing to long-term
  key management discipline.
- The team has an institutional identity (a GitHub org or
  similar) to anchor the signing identity to.

**Neither alone is the right choice when:** the artifact
needs to be consumable across both old (OS-level) and new
(cloud-native) ecosystems. See "double-signing" below.

## Double-signing strategy

**Double-signing** (sometimes called dual-signing) means
producing two independent signatures on the same artifact:
one in each paradigm. A consumer verifies the signature
matching their toolchain; consumers in the other ecosystem
verify theirs; no consumer has to change their tooling to
accept the artifact.

For an RPM, a typical pipeline:

1. Build the `.rpm` (unsigned).
2. **GPG layer.** `rpmsign --addsign package.rpm` — embeds
   an OpenPGP signature header into the RPM metadata.
3. **Sigstore layer.** `cosign sign-blob package.rpm` — produces
   a detached signature (`sig`) and certificate (`cert`)
   referencing a Rekor log entry.
5. Publish the artifact, the cosign `sig` + `cert` + bundle,
   and the public GPG key.

### When double-signing is worth it

Double-signing is worth the extra CI / storage cost when:

- The artifact is consumed by **both** ecosystems (OS-level
  package managers *and* cloud-native supply chain tooling).
- The team can absorb the operational overhead: signing
  twice in CI, publishing two sets of artifacts, documenting
  two verification paths.
- The team's threat model has both old and new attack
  surfaces.

Concrete teams/projects where this has worked: Kubernetes
upstream (signs container images with cosign, plus source
tarballs with GPG); several Linux distributions shipping
cloud-native artifacts alongside their OS packages; the
Sigstore project itself.

### When double-signing is overkill

Double-signing is **not** worth the overhead when:

- The audience is exclusively one ecosystem.
- The artifact isn't reused across contexts.
- The team can't afford the doubled CI complexity.

### Hybrid pipeline notes

If double-signing is adopted, a few practical considerations:

- **Order matters.** Sign with GPG first (modifies the RPM
  header); then sign the resulting RPM with cosign. Reversing
  the order means cosign would be signing the pre-GPG state
  and the GPG signature wouldn't be covered by Rekor.
- **Transparency log coverage.** Rekor records the artifact
  hash as of the moment of signing. The cosign signature
  covers the post-GPG RPM.
- **Key rotation cost.** The GPG layer still needs a key
  rotation story; the Sigstore layer rotates by itself
  (certificates are short-lived). The two layers have
  independent operational cadences.
- **Documentation.** The team has to maintain documentation
  for two verification paths. README / docs need both
  `rpm -K` and `cosign verify-blob` recipes.

## Sigstore adoption (2026)

Sigstore graduated from the CNCF in 2023 and is now used
in production by:

- **Kubernetes** upstream — signed release artifacts and
  container images.
- **GitHub** — `gh` CLI can sign releases with Cosign.
- **npm** — sigstore-based attestations for npm packages.
- **Several Linux distributions** — alongside their GPG
  signing for OS-level packages.
- **The Sigstore project itself** — signs its own binaries
  with both GPG and Cosign.

## Limitations of Sigstore

A few things to keep in mind:

- **OIDC IdP reliability.** Sigstore's trust chain bottoms
  out in the OIDC identity provider. If your GitHub org is
  compromised, an attacker can mint signing certificates as
  you. Defense is hardening the OIDC identity (org security,
  branch protection, mandatory 2FA, etc.).
- **Rekor's growth.** Rekor's Merkle tree is append-only and
  grows forever. Full Rekor nodes hold the entire tree;
  lightweight clients can verify inclusion proofs against a
  published checkpoint.
- **Time-bound verification windows.** Fulcio's certificates
  expire in ~10 minutes. Signatures are self-attesting at
  the moment of signing, but ongoing validity requires
  checking the transparency log.
- **No OS-level package manager support (yet).** As of 2026,
  `dnf`, `apt`, etc. don't natively verify Cosign signatures.
  For OS-level distribution, GPG is still the way.
- **No pre-institutional identity.** Sigstore needs an
  institutional anchor. An anonymous OSS contributor who
  doesn't have one can't sign with Sigstore in the same way
  they'd sign with GPG.

## Where to read next

- [0. x-cmd/gpg overview](./0-x-cmd-gpg-overview.md) — what
  this repo is and the maintenance policy.
- [1. What is GPG and how do I use it?](./1-what-is-gpg-and-how-do-i-use-it.md) —
  end-user introduction; fingerprint as the trust anchor.
- [3. Annual key strategy](./3-annual-key-strategy-explained.md) —
  long-term GPG key rotation trade-offs; pairs with the
  Sigstore layer if you adopt double-signing.
- [4. GPG UID naming conventions](./4-gpg-uid-naming-conventions.md) —
  ™/® in UIDs; brand-defense work goes elsewhere.
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — release-side
  pipeline conventions once a signing strategy has been
  picked.

## Sources

- [Sigstore project homepage](https://www.sigstore.dev/)
- [Cosign documentation](https://docs.sigstore.dev/policy-controller/overview)
- [Fulcio — Sigstore's CA](https://github.com/sigstore/Fulcio)
- [Rekor — Sigstore's transparency log](https://github.com/sigstore/Rekor)
- [RFC 4880 — OpenPGP Message Format](https://datatracker.ietf.org/doc/html/rfc4880)