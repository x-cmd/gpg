---
x-title: x-cmd/gpg — overview
x-desc: The x-cmd team's GPG public keyring — one-page summary linking to the deep-dive articles below.
x-sidebar: x-cmd/gpg overview
x-keywords: x-cmd, gpg, public key, trust anchor, supply chain, signature verification, keyring
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/gpg — the team keyring'
      inLanguage: 'en'
      about: 'x-cmd team GPG keyring'
---

# x-cmd/gpg — the team's keyring

> The canonical, single-source-of-truth list of GPG public keys
> for the x-cmd core team. **Pull from this repo directly** —
> every byte served from any other domain is untrusted. See
> [`LICENSE`](../LICENSE) for the proxy-redistribution clause
> and the trust-anchor rationale.

This page is the one-page summary. The articles below go deep
on the *why* and *how* — what a GPG trust anchor is, how the
keyring gets published, how to read `index.tsv`, why the team
publishes both a community key and an annual enterprise key,
how to verify a fingerprint, and the three ways to consume
the bytes.

## The current keys

| Handle | UID | Fingerprint | Created | Purpose |
| --- | --- | --- | --- | --- |
| _(no keys published yet — see_ [`CONTRIBUTING.md`](../CONTRIBUTING.md)_)_ | | | | |

The live table is regenerated from
[`index.tsv`](../index.tsv) on every release commit by the
team. Pin to the **fingerprint**, not the handle — handles can
be re-used across rotations; fingerprints cannot.

## Two published keys per team (FAQ Q4)

The team's supply-chain keying strategy publishes two signing
keys, each with a different operational role:

- **Community key** — signs the standard community package
  (`x-cmd.rpm` / `x-cmd.deb`). One import, every future
  upgrade verifies silently.
- **Annual key (`key-<year>`)** — signs the per-year enterprise
  compliance package (`x-cmd-annual-<year>.rpm`). Strict year-
  on-year isolation for finance / government procurement
  audits.

Both keys live in this repo under [`keyring/`](../keyring/),
aggregated into [`keyring/keyring.asc`](../keyring/keyring.asc).
The articles below cover why this split, why "no expiry"
cryptographically but "annual rotation" operationally, and the
repackage / resign lifecycle that paid LTS customers rely on.

## How to consume (three paths)

```sh
# 1. Direct fetch — pull keyring.asc straight from GitHub
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# 2. One key at a time
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import

# 3. x gpg shell module (auto-caching + fingerprint cross-check)
x gpg import
x gpg info <handle>
x gpg verify <sig> <file>
```

**Pull from GitHub directly. Do not** route through any CDN,
reverse proxy, or caching service — see
[`LICENSE`](../LICENSE) for the proxy-redistribution clause.

## Read next

- [1. Why x-cmd/gpg exists](./1-why-x-cmd-gpg-exists.md) — the
  supply-chain problem this repo solves
- [2. How the keyring is published](./2-how-the-keyring-is-published.md) —
  the team-internal pipeline from a fresh `gpg --export` to a
  row in `index.tsv`
- [3. Reading the key catalog](./3-reading-the-key-catalog.md) —
  every column of `index.tsv`, fingerprint math, why pin to
  fingerprint
- [4. Annual key strategy explained](./4-annual-key-strategy-explained.md) —
  the long-form version of FAQ Q4–Q8
- [5. Verifying a key](./5-verifying-a-key.md) — the three-step
  fetch → import → compare recipe
- [6. Three ways to consume the keyring](./6-three-ways-to-consume.md) —
  raw curl, `x gpg`, and the GitHub-Pages-via-x-cmd.com redirect

For the technical reference (file layout, schema, CI), see
[`CONTRIBUTING.md`](../CONTRIBUTING.md). For AI-agent recipes
and quick command line, see [`SKILL.md`](../SKILL.md).

## FAQ — software distribution & code-signing cryptography

The questions below are educational and project-agnostic.
They describe the industry-wide status quo for GPG key
management, lifetime design, and supply-chain security in
software distribution — pros, cons, and trade-offs of the
major approaches, without recommending any one of them.
Articles 1–6 in this series dig into individual topics in
more depth.

### Q1: GPG software vs. GPG Key — what's the technical relationship?

It's the relationship between a software program and a data
credential.

- **GPG (GNU Privacy Guard)** is an open-source encryption
  program implementing the OpenPGP international standard.
  It performs the concrete *computational actions* —
  encryption, decryption, signature generation, signature
  verification.
- **GPG Key (keypair)** is the data credential the software
  consumes. It comprises a publicly-shareable *public key*
  (others use it to encrypt to you or to verify your
  signature) and a strictly confidential *private key*
  (you use it to decrypt or to create signatures).

The two are inseparable in practice — GPG without keys has
nothing to encrypt with, and keys without GPG have nothing
that can perform the cryptographic operations.

### Q2: Sigstore (keyless signing) exists now — why do RPM / DEB still rely on GPG?

The answer is historical compatibility with the operating
system's native toolchain.

- **Sigstore** is widely adopted in modern cloud-native
  environments — Docker images, Kubernetes components,
  npm / PyPI packages. Its core idea is short-lived
  ephemeral certificates plus a transparency log (Rekor) to
  eliminate long-term private-key management.
- **RPM (dnf / yum)** and **DEB (apt)** are the native
  base package managers of mainstream Linux distributions.
  Their underlying verification engines were designed with
  deep integration to OpenPGP (GPG) from day one. To keep
  the system-level security defense from being bypassed,
  they remain 100% dependent on GPG keys for digitally
  signing packages or source-index files.

The two ecosystems co-exist; for OS-level packages the
de-facto channel is still GPG.

### Q3: Why don't publishers usually distribute unsigned raw packages?

**Pro (advantages)**

- Developer has zero key-management overhead; the release
  process is extremely simple.
- Users or enterprises can completely re-sign packages
  offline under their own internal-network security policy.

**Con (disadvantages)**

- Network transmission and CDN nodes lack cryptographic
  tamper protection — MITM and package poisoning become
  trivial.
- Most modern Linux distribution package managers will
  pop up an error and refuse to install unsigned packages
  by default, increasing user-side operational friction.

### Q4: What are the pros and cons of hardcoding a GPG key to "never expire"?

**Pro (advantages)**

- **Extreme business continuity.** Servers deployed years
  ago can re-run checks or environment restores at any
  future point without automation scripts crashing due to
  "publisher key expired".
- **Very low maintenance cost.** Publishers don't need to
  rotate CI/CD keys at a specific date each year, nor
  publish announcements reminding global users to refresh
  their public keys.

**Con (disadvantages)**

- **Unlimited blast radius.** Once a private key leaks from
  a compromised build server or dev machine, attackers can
  forge any future new version indefinitely. Recovery
  requires the very complex "revocation certificate"
  distribution mechanism.

### Q5: Why have many historical certificates and keys had lifetimes of "398 days" or "397 days"?

The history traces to mandatory lifetime limits imposed by
the CA/Browser Forum, Apple, and Google on publicly-trusted
Web certificates (SSL/TLS).

- **398 days (cryptographic design).** Since 2020,
  international standards mandate that one-year Web
  certificates cannot exceed a 398-day maximum lifecycle.
  This is 365 days (1 year) baseline plus 33 days of buffer
  for cross-year holidays and multi-timezone transitions.
- **397 days (engineering practice).** Because global
  servers have timezone-conversion drift, some automated
  compliance scanners report "certificate expired" false
  positives at the 398-day boundary due to a few hours of
  timezone skew. Prudent engineers therefore hardcode 397
  days in practice, conceding 1 day defensively in exchange
  for 100% green-light pass rate from global scanners.

### Q6: What changed for SSL/TLS certificates in 2026, and does code signing get affected?

- **Web certificates have dropped sharply.** Per the latest
  international resolution, since March 2026 the maximum
  validity for publicly-trusted Web certificates has been
  compressed to under 200 days, with plans to shorten
  further to ~100 days in 2027 — automation aims to
  eliminate long-term keys entirely.
- **Code signing is compliance-exempt.** International
  root-certificate programs and OS-level security-audit
  specifications explicitly classify package signing and
  code signing as infrastructure anchors, *not* part of
  the Web-certificate lifetime-reduction program. In the
  Linux-package-distribution and enterprise-compliance
  field, 1- to 2-year long-term key rotation remains the
  industry-mainstream practice.

### Q7: For commercial software adopting "1-year rotation" key isolation, what are the pros and cons?

**Pro (advantages)**

- **High security and compliance.** Aligns with the
  "Annual Security Audit" metric required by most
  financial and government-enterprise procurement teams.
  Even if a year's private key leaks, the risk is fully
  contained to that single year of versions.
- **Commercial stickiness.** Mandatory annual trust-source
  updates serve as a natural technical anchor for
  enterprise customers to renew their "Technical Support
  & Security Service Contract".

**Con (disadvantages)**

- **Old-system compatibility friction.** If old systems
  delete the old key at year boundary, historical-version
  software previously running on them starts reporting
  errors during routine dependency scans due to missing
  signature source.
- **Dual-signing dilemma.** Trying to embed two keys (old
  + new) into the same RPM produces inconsistent behavior
  across Linux distribution verification engines (old
  CentOS vs new Rocky Linux), easily triggering unknown
  failures in production.

### Q8: If "1-year rotation" is adopted, how does industry solve cross-year transition and historical rollback?

The pattern is **Trust Anchor Registry**:

- **Permanent public-key repository.** A dedicated
  credential path on the official site (a public data repo
  or dedicated CDN path) combines all historical annual
  public keys (`key-2025.gpg`, `key-2026.gpg`, …) into a
  single keyring.
- **Control returned to users.** Enterprise systems
  import both this year's and next year's public keys.
  The system then has both historical and future keys;
  regardless of whether an old system is moving to a new
  version, or a clean system is installing a historical
  package, the package manager can unlock locally. The
  ultimate audit decision of "should we forcibly invalidate
  the old key?" is left to the enterprise's own operations
  policy.