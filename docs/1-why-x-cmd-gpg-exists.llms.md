---
name: 1-why-x-cmd-gpg-exists
description: Primer — what a GPG trust anchor is, the supply-chain problem this repo solves, why x-cmd keeps its own keyring instead of using keys.openpgp.org / keyserver.ubuntu.com, and what this repo is NOT.
type: primer
---

# Core Content

core_features:

- Defines "trust anchor" as: the exact bytes of a public key, fetched from a channel independently verified to be the team's own
- Explains the two-condition model for signature verification: signature was produced by the team's private key (math gives you this), and the public key being verified against is the team's public key (this repo gives you this)
- Three reasons for a separate repo: avoids circular signing with x-cmd/x-cmd, shrinks trust surface by isolating key churn from monorepo churn, gives rotations a tiny independent review surface
- Four reasons against public keyservers (keys.openpgp.org, keyserver.ubuntu.com): anyone can upload, substitution attacks via look-alike keys, no first-party audit trail, LICENSE mismatch (LICENSE explicitly forbids third-party redistribution)

## Key Information

highlights:

- This repo is the producer; `x gpg` (in x-cmd/x-cmd) is the consumer
- The team's signing strategy is two keys per team: community master + annual `key-<year>` isolation key
- Pull from GitHub directly every time — proxy redistribution (CDN, reverse proxy, caching proxy) is forbidden by LICENSE because intermediate caches can return stale bytes during a rotation or substitute look-alike fingerprints
- Cryptographic expiry is "no expiry"; operational rotation is "annual" — see FAQ Q5
- External PRs touching `keyring/` or `index.tsv` are closed without merge (fingerprint swap = supply-chain attack)
- Suggestions for the article (README prose, docs, typos) go to issues, not PRs

## Use Cases

use_cases:

- Onboarding a new team member or contractor who needs to understand the trust model
- Walking a procurement / compliance officer through why the team's keying strategy looks the way it does
- Justifying to a security reviewer why this repo is its own thing rather than living inside `x-cmd/x-cmd`
- Defending the decision not to push keys to keys.openpgp.org

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)
  gpg_docs: <https://gnupg.org/documentation/>
  public_ks: <https://keys.openpgp.org/>

## Summary

A primer for newcomers to GPG, supply-chain keying, or x-cmd. Defines "trust anchor" as the exact bytes of a public key fetched from a channel independently verified to be the team's own. Walks through the two-condition model for signature verification: (1) the signature was produced by the team's private key — what the math gives you — and (2) the public key being verified against is the team's public key — what this repo gives you. Justifies the separate-repo design with three reasons (no circular signing, smaller trust surface, independent review surface for rotations) and rejects public keyservers (keys.openpgp.org, keyserver.ubuntu.com) for four reasons (anyone can upload, substitution attacks, no first-party audit trail, LICENSE mismatch). Closes by enumerating what the repo is NOT — not a general-purpose keyserver, not a revocation authority, not a signature archive.