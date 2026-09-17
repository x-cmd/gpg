---
name: 5-verifying-a-key
description: Three-step verification recipe (fetch, import, compare) with look-alike attack, CDN-cache pitfall, self-signature-vs-team-signature, trust-on-first-use guidance.
type: how-to
---

# Core Content

core_features:

- Three steps: fetch (pull bytes from a trusted channel), import (pipe to gpg --import, validates self-signatures), compare (fingerprint must match index.tsv AND team site — three-way agreement)
- Self-signature ≠ team endorsement: gpg --import validates "key is well-formed", not "key is the team's"
- Team endorsement requires: bytes match the team's commit history, fingerprint matches index.tsv + team site, optionally cross-signature from another team member's key
- Four common pitfalls and their defenses: CDN cache (stale bytes during rotation), look-alike key (UID match with different fingerprint), handle re-use (rotation takes same handle), short key ID (32-bit truncation is reversible)
- TOFU (trust-on-first-use): if no independent channel available, GitHub's HTTPS endpoint is the platform-level trust; out-of-band signed message from team needed for "GitHub serves wrong bytes" threat model

## Key Information

highlights:

- The "Compare" step is load-bearing — without it, importing a key proves only that you got a syntactically valid OpenPGP blob (an impostor trivially satisfies this)
- Three-way Compare: local keyring fingerprint == index.tsv fingerprint == team-site fingerprint
- Never search public keyservers for a known team's key — always fetch from this repo directly
- Always pin to the full 40-char fingerprint, never to the 32-bit short key ID (`0xDEADBEEF`) — short ID is reversible in practice
- Pull from GitHub directly every time, no CDN / reverse proxy / caching intermediate — see LICENSE for proxy-redistribution clause
- The team site publishes signed rotation messages so consumers who don't trust GitHub can verify the new fingerprint out-of-band

## Use Cases

use_cases:

- First-time onboarding for a single maintainer's key
- Verifying a fresh import after a rotation
- Diagnosing "my gpg --verify fails after a rotation" — usually a stale-bytes-via-CDN issue
- Building a CI/CD pipeline that imports the keyring as a one-time bootstrap step

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)
  gpg_verify: <https://gnupg.org/documentation/manuals/gnupg/Operational-GPG-Commands.html>

## Summary

A fingerprint in a README is not proof — anyone can type 40 hex characters. Verification is a three-step process that ties the bytes you imported to the team's intent using channels that don't all rely on the same source. Step 1 (fetch): pull `keyring/keyring.asc` over HTTPS from raw.githubusercontent.com, ideally cross-checked against the team site. Step 2 (import): pipe to `gpg --import` which parses OpenPGP packets and validates each key's self-signature. Step 3 (compare): the fingerprint GnuPG prints must match `index.tsv` AND the value on the team site — three-way agreement. The Compare step is load-bearing; without it, importing a key proves only that you got a syntactically valid OpenPGP blob (an impostor trivially satisfies this). Self-signature ≠ team endorsement — `gpg --verify` validates "key is well-formed", not "key is the team's"; team endorsement requires the bytes matching commit history, the fingerprint matching index.tsv + team site, and optionally cross-signature from another team member. Four pitfalls and defenses: CDN cache (stale bytes during rotation — pull directly, no intermediate), look-alike key (UID match with different fingerprint — never search keyservers for a known team), handle re-use (rotation takes same handle — pin to fingerprint), short key ID (32-bit truncation reversible — use full 40-char fingerprint). Trust-on-first-use: if no independent channel is reachable, GitHub's HTTPS endpoint is the platform-level trust; for the "GitHub serves wrong bytes" threat model, require an out-of-band signed message from the team (published on the team site at every rotation).