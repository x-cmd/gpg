---
name: 4-annual-key-strategy-explained
description: Long-form rationale for x-cmd's supply-chain keying strategy. Two-key model (community master + annual isolation), no cryptographic expiry but annual operational isolation, repackage/resign lifecycle for paid LTS customers, why 398-day TTL numbers are wrong here, why never auto-remove old keys.
type: strategy
---

# Core Content

core_features:

- Two published keys per team: `official` (community master, signs `x-cmd.rpm`/`x-cmd.deb`) and `key-<year>` (annual isolation, signs `x-cmd-annual-<year>.rpm`)
- Annual key has cryptographic "no expiry" but operational "annual" rotation — physically sealed at year boundary, never used to sign new artifacts
- Repackage/resign lifecycle: `rpmsign --addsign` against existing release asset to stamp new signature header without recompiling — paid LTS feature
- `keyring/archive/` keeps historical annual keys forever, signed by the team's primary key for chain-of-custody verification
- Team takes no position on whether operator should prune historical keys from their own hosts — that's their call, their outage
- Two distinct audiences drive the split — community optimizes for friction-free upgrades; enterprise optimizes for strict year-on-year auditability

## Key Information

highlights:

- The "no expiry cryptographically, annual rotation operationally" choice resolves two competing forces: cryptographic hygiene wants hard expiry, but operations hates hard expiry in production (creates hard SLA boundary, paging incidents, service-credit disputes)
- 398/397/380-day TTLs are Web-PKI numbers (CA/B Forum shortened public-TLS to <200 days through 2026) — reusing them in code-signing reads as "team doesn't know which standard applies where"
- Per-calendar-year boundary (365 days business, no expiry crypto) aligns with finance/government audit cycles — explainable to non-technical stakeholders
- Repackage/resign lifecycle is a paid LTS feature; free users verify historical artifacts as long as they hold the corresponding historical key (permanently available in `keyring/archive/`)
- Both keys live in this repo's `index.tsv`; `key-<year>` rows get `package-signing-yearly` purpose; archived rows move to `keyring/archive/<handle>.<created-date>.asc`

## Use Cases

use_cases:

- Walking a finance/government procurement team through the keying strategy for an enterprise rollout
- Explaining to a security reviewer why "no expiry" + annual isolation is more responsible than 384-day expiry
- Understanding which packages are signed by which key (`official` vs `key-<year>`), and which to import
- Diagnosing why a 2027 artifact re-sign request goes to LTS support, not the open-source channel

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  faq_in_readme: <https://github.com/x-cmd/gpg/blob/main/README.md#faq--supply-chain--commercial-deployment>
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)

## Summary

The team's supply-chain keying strategy looks unusual: two published keys instead of one, one rotated every calendar year but cryptographically set to never expire. Two-key model resolves the tension between community-edition friction-free upgrades (one import, every future upgrade verifies silently) and enterprise-edition strict year-on-year auditability (finance/government needs to isolate each year's signatures). The annual key has no cryptographic expiry (so cross-boundary verifications never hit "key expired" errors that page on-call at 2 AM) but is operationally sealed at year boundary (so no artifact is signed by both `key-2026` and `key-2027`). The "no expiry / annual" choice resolves two competing forces: cryptographic hygiene wants hard expiry; operations hates hard expiry in production because it creates a paging-incident-at-midnight SLA boundary. Reject 398/397/380-day TTLs as wrong context — those are Web-PKI numbers (CA/B Forum shortened public-TLS to <200 days through 2026); reusing them in code-signing reads as copy-paste from the wrong standard. Repackage/resign lifecycle (`rpmsign --addsign` without recompiling) lets paid LTS customers get older artifacts re-signed with the current annual key; the re-signed packages are publicly hosted (transparency is part of the value) but *which* historic versions to re-sign and how often is a service-tier decision. Free users verify historical artifacts as long as they hold the corresponding historical key (permanently in `keyring/archive/`); paid LTS users pay for "every artifact carries the signature from the key I'm currently using" without managing a per-year key inventory themselves.