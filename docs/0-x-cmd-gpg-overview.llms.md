---
name: 0-x-cmd-gpg-overview
description: One-page summary of x-cmd/gpg — the team keyring repo. What it is, the two-key strategy (community + annual), three consumption paths, and links to the deep-dive articles.
type: summary
---

# Core Content

core_features:

- Canonical single-source-of-truth for x-cmd team's GPG public keys
- Two published keys per team: community master (signs x-cmd.rpm/.deb) + annual isolation key (signs x-cmd-annual-<year>.rpm for finance/government audits)
- 5-column `index.tsv` manifest: handle, uid, fingerprint, created, purpose
- ASCII-armored keys under `keyring/<handle>.asc`, aggregated into `keyring/keyring.asc`
- Three consumption paths: raw curl from GitHub, `x gpg` shell module, GitHub-Pages-via-x-cmd.com
- Pull from GitHub directly — proxy redistribution (CDN, reverse proxy, caching proxy) is explicitly forbidden by LICENSE

## Key Information

highlights:

- This repo is the producer; `x gpg` (in `x-cmd/x-cmd`) is the consumer
- The team publishes both a community key (long-lived, signs standard packages) and an annual `key-<year>` key (signs that year's enterprise package, sealed at year boundary)
- Cryptographic expiry is "no expiry"; operational rotation is "annual" — see LICENSE + FAQ Q5 rationale
- External PRs touching `keyring/` or `index.tsv` are closed without merge (fingerprint swap = supply-chain attack)
- Suggestions for the article (README prose, docs, typos) go to issues, not PRs
- LICENSE: Copyright 2026 x-cmd — All Rights Reserved; published for the limited purpose of fetching/using keys for verification

## Use Cases

use_cases:

- Looking up the x-cmd team's GPG public key before verifying a release tarball or RPM
- Onboarding an enterprise / government procurement team that needs to pin to a specific year's signing key
- Auditing the team's supply-chain keying strategy for compliance review
- Building CI that fetches the keyring directly from GitHub on every run

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)
  gpg_docs: <https://gnupg.org/documentation/>

## Summary

x-cmd/gpg is the canonical keyring repo for the x-cmd core team. It publishes the team's GPG public keys under `keyring/<handle>.asc` (per-key files) plus `keyring/keyring.asc` (the aggregated keyring, regenerated on every release commit), with a 5-column `index.tsv` manifest that documents each key's handle, UID, fingerprint, creation date, and signing purpose. The team's supply-chain strategy publishes two distinct keys — a community master key for the standard package, and an annual `key-<year>` isolation key for the per-year enterprise/compliance package — so finance and government procurement teams can audit each year independently. Pull from GitHub directly every time; the LICENSE explicitly forbids proxy redistribution (CDN, reverse proxy, caching proxy) because any intermediate between the consumer and GitHub can return stale bytes during a rotation or substitute a look-alike fingerprint. Deep dives on the rationale (why a separate repo, how the pipeline works, index.tsv schema, annual key strategy, verification recipe, three consumption paths) follow on pages 1–6.