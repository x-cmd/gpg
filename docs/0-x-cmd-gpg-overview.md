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
