---
x-title: Cosign and Sigstore, explained
x-desc: What Sigstore is (the project), what Cosign is (the CLI), how Fulcio (CA) and Rekor (transparency log) fit into the signing workflow, and what a typical sign-and-verify flow looks like. **Project-agnostic; exploratory.**
x-sidebar: Cosign and Sigstore, explained
x-keywords: sigstore, cosign, fulcio, rekor, transparency log, keyless signing, oci, container, slsa, supply chain, cncf
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cosign and Sigstore, explained'
      inLanguage: 'en'
      about: 'Sigstore ecosystem and Cosign CLI'
---

# Cosign and Sigstore, explained

Sigstore and Cosign are the most visible pieces of a 2020s
shift in supply-chain signing: away from "long-term private key
signing" toward "short-lived certificate + transparency log"
signing, anchored to existing institutional identities
(GitHub, Google, etc.). This article walks through the
ecosystem as a whole — what each piece is, what each piece
does, and how a typical sign-and-verify flow looks.

The article deliberately doesn't pick a winner between
Sigstore and GPG. See
[9. Sigstore vs GPG, and double-signing](./9-sigstore-vs-gpg-and-double-signing.md)
for that comparison.

> **Status: exploratory.** This is a project-agnostic
> explanation of Sigstore and Cosign as they exist in the
> ecosystem. The x-cmd team hasn't adopted Sigstore for any
> actual release signing as of this writing.

## One-paragraph summary

**Sigstore** is a project (originally from Google, now a
CNCF graduated project under the Linux Foundation) that
provides *keyless* software signing — short-lived certificates
bound to OIDC identities, recorded in a public append-only
log, with verification tools. **Cosign** is Sigstore's CLI
tool for signing and verifying container images, blobs,
attestations, and similar artifacts. **Fulcio** is the
certificate authority that issues the short-lived signing
certificates. **Rekor** is the transparency log that records
every signing event publicly. Together they let a team
publish signed artifacts *without managing any long-term
private key* — the trade-off is that signing identity is
anchored to an OIDC identity provider (typically GitHub or
Google), not to a key the team holds.

## The three pieces

The Sigstore ecosystem has three core pieces, plus a handful
of supporting tools.

### Cosign — the CLI

Cosign is the user-facing tool. It does two things:

1. **Sign** — produce a signature, a certificate, and an
   entry in the transparency log.
2. **Verify** — check that a signature was produced by a
   particular identity and is recorded in Rekor.

Typical commands:

```sh
# Sign a container image (the most common use case)
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
certificates. When you run `cosign sign --keyless`, here's
what happens:

1. Cosign asks your OIDC identity provider (typically
   GitHub Actions or `gh auth login`) for an OIDC identity
   token — a signed JWT asserting "I am user X in org Y".
2. Cosign sends that token to Fulcio.
3. Fulcio verifies the OIDC token (the IdP's signature
   chain checks out), then issues an X.509 signing
   certificate binding the OIDC identity (e.g.
   `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`)
   to a fresh public key generated for this signing event.
4. The certificate has a ~10 minute expiry (Fulcio is
   configured with short certificate lifetimes by
   default).
5. Cosign uses the matching private key to sign the
   artifact, then throws the private key away.

The signed artifact, the signature, and the certificate are
then sent to Rekor (next section) for transparency logging.

Fulcio is operated by the Sigstore project (the same
organization that runs Rekor). Fulcio's own trust comes
from its root certificate, which is published and audited
by the community.

### Rekor — the transparency log

Rekor is an append-only, hash-linked Merkle tree of every
signing event the Sigstore network has processed. When you
sign with Cosign, the signature, certificate, and artifact
hash are bundled and submitted as a Rekor entry; Rekor
returns an inclusion proof that ties the entry to the
Merkle root.

The Rekor tree is published periodically as a signed
checkpoint that anyone can verify against an earlier
checkpoint (and ultimately against a "trust root" published
at a known URL). A verifier reconstructs the same tree
locally and checks that the entry is included.

Practical consequences:

- Every signing event is **public** and **tamper-evident**.
  If Fulcio issued a certificate, it's in Rekor; if Rekor
  has it, the inclusion proof verifies.
- Rekor grows forever. Full nodes hold the entire tree.
  Operators of Rekor full nodes commit to long-term storage.
- Rekor is operated by the Sigstore project; clients talk
  to a public instance at `rekor.sigstore.dev` but can also
  run their own.

### Supporting tools

A handful of other tools round out the ecosystem:

- **gitsign** — signs Git commits using Sigstore (Cosign +
  Fulcio + Rekor) instead of GPG. A drop-in replacement for
  `git commit -S`.
- **policy-controller / kyverno integration** — Kubernetes
  admission webhooks that reject every Reimage/Cosign-signed
  image whose signature doesn't verify against the
  configured identity.
- **SLSA provenance** — the Sigstore tools are commonly used
  to sign and verify SLSA provenance attestations
  (in-toto), alongside the artifact signatures.
- **The Update Framework (TUF)** integration — TUF-signed
  metadata can be signed with Cosign.

## A typical sign-and-verify flow

Putting the three pieces together, a typical flow looks
like:

### Signing (the publisher side)

```
1. CI build runs `cosign sign-blob package.rpm`
2. Cosign obtains an OIDC token from the IdP (e.g.
   GitHub Actions exposes `$ACTIONS_ID_TOKEN_REQUEST_TOKEN`
   for this)
3. Cosign sends the token to Fulcio
4. Fulcio returns a short-lived X.509 certificate bound to
   the OIDC identity
5. Cosign generates a fresh keypair, signs the artifact
   with the private half, throws the private half away
6. Cosign submits {signature, certificate, artifact-hash} to
   Rekor
7. Rekor returns an inclusion proof
8. Cosign writes the signature, certificate, and Rekor
   bundle alongside the artifact (or attaches them in the
   container registry, in the case of OCI images)
```

### Verifying (the consumer side)

```
1. Consumer pulls the artifact + signature + certificate +
   bundle
2. Consumer runs `cosign verify-blob` (or `cosign verify` for
   OCI)
3. Cosign checks the certificate chain: signed by Fulcio,
   within the ~10 minute validity window
4. Cosign checks the OIDC identity in the certificate:
   matches the expected identity (e.g.
   `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`)
5. Cosign checks the Rekor inclusion proof: the signature
   was logged, and the bundle's Merkle root matches the
   signed checkpoint
6. Cosign verifies the artifact signature against the
   certificate's public key
```

Every step is independently checkable. A consumer doesn't
need to trust the Sigstore servers beyond trusting that
the published Merk tree is genuine (which is rooted in
public-key-signed checkpoints).

## The OIDC identity contract

The most consequential design choice in Sigstore is *which
identity the signing certificate binds to*. The OIDC token
includes a claim like:

- `email` — the user's email address (for human signers)
- `sub` — the OIDC subject identifier
- A URL-shaped "identity" claim that uniquely identifies the
  workflow, the repo, the ref, etc. — used for CI signers

A typical signing identity for a GitHub Actions workflow
looks like:

```
https://github.com/example/app/.github/workflows/release.yml@refs/tags/v1.2.3
```

Verifiers pin to that identity. The fingerprint of "who
signed this artifact" is therefore "the GitHub user X
running workflow Y on commit Z of repo R" — an *event* in
the institutional sense, not a long-term key.

The contract between publisher and verifier is:

- Publisher commits to *who* signs (the OIDC identity).
- Verifier pins to that identity.
- The math and the transparency log guarantee the signature
  was produced by that identity at the time recorded.

This is exactly the inverse of the GPG contract, where the
publisher commits to *which key* signs and the verifier pins
to the key bytes.

## Adoption (2026 state)

Sigstore graduated from the CNCF in 2023 and is now used in
production by:

- **Kubernetes** upstream — signed release artifacts and
  container images.
- **GitHub** — `gh` CLI can sign releases with Cosign.
- **npm** — sigstore-based attestations for npm packages.
- **Several Linux distributions** — alongside their GPG
  signing for OS-level packages.
- **The Sigstore project itself** — signs its own binaries
  with both GPG and Cosign.

Cosign's UI has stabilized over 2023–2026; the
transparency-log and Fulcio infrastructure are public and
auditable. Most tooling integration has settled on
`cosign sign` / `cosign verify` as the canonical commands.

## Limitations and gotchas

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
  expire in ~10 minutes. That means signatures are *self-
  attesting* at the moment of signing, but you can't go back
  and ask "is this signature still valid?" without checking
  the transparency log and the certificate's chain of trust.
- **No OS-level package manager support (yet).** As of 2026,
  `dnf`, `apt`, etc. don't natively verify Cosign signatures.
  For OS-level distribution, GPG is still the way.
- **No pre-institutional identity.** Sigstore needs an
  institutional anchor (typically a GitHub org). An anonymous
  OSS contributor who doesn't have one can't sign with
  Sigstore in the same way they'd sign with GPG.

## What to read next

- [9. Sigstore vs GPG, and the double-signing strategy](./9-sigstore-vs-gpg-and-double-signing.md) —
  the comparison with GPG and the hybrid strategy.
- [7. Signing an RPM with GPG](./7-signing-an-rpm-with-gpg.md) —
  the GPG-side counterpart of a typical double-signing
  pipeline.
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — once a signing
  strategy has been picked, how keys / identities get
  published.

## Sources

- [Sigstore project homepage](https://www.sigstore.dev/)
- [Cosign documentation](https://docs.sigstore.dev/policy-controller/overview)
- [Fulcio — Sigstore's CA](https://github.com/sigstore/Fulcio)
- [Rekor — Sigstore's transparency log](https://github.com/sigstore/Rekor)
- [CNCF announcement of Sigstore graduation](https://www.cncf.io/announcements/2023)