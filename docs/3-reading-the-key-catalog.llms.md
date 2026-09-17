---
name: 3-reading-the-key-catalog
description: index.tsv schema, fingerprint as cryptographic commitment, handle vs UID vs key-id vs fingerprint for pinning, practical awk queries.
type: data-presentation
---

# Core Content

core_features:

- 5-column TSV manifest: handle, uid, fingerprint, created, purpose — no header in the file, sorted by handle (lex/ASCII) for stable diffs
- `handle`: team-chosen lowercase ASCII identifier, can be re-used across rotations; reserved handles include `official` (community master) and `key-<year>` (annual isolation)
- `uid`: primary UID from OpenPGP packet — display text, editable, not a security property
- `fingerprint`: 40-char uppercase hex, SHA-1 of the public-key packet (RFC 4880 §12.2), 160-bit cryptographic commitment — THE trust anchor
- `created`: YYYY-MM-DD from `gpg --list-keys --with-colons`; informational only
- `purpose`: free-text lowercase-hyphenated (release-signing, package-signing, package-signing-yearly, mirror-signing); convention not enforced

## Key Information

highlights:

- Pin to fingerprint, never to handle, UID, or key-id
- Handle is a label, can be re-used across rotations
- UID is editable; look-alike attacker can construct same-name same-email key with different fingerprint
- Short key-id (the `0xDEADBEEF` form) is truncated 32 bits from the fingerprint — vulnerable to key-id collision attacks
- Full 160-bit fingerprint has no practical collision attack
- For "is the signing key for artifact X the one in index.tsv?": extract fingerprint from artifact's signature, look up in index.tsv, only trust if matches — anything weaker is vulnerable to substitution
- Three-way fingerprint cross-check: index.tsv vs GitHub raw vs team site — all three must agree

## Use Cases

use_cases:

- Looking up the fingerprint for a specific purpose (release-signing, package-signing-yearly, etc.)
- Auditing which keys are aging and might be near their rotation window
- Cross-checking a fingerprint you've seen in a vendor advisory against the team's index
- Forensically proving the bytes you fetched from GitHub match the bytes the team published

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)
  rfc4880: <https://datatracker.ietf.org/doc/html/rfc4880#section-12.2>

## Summary

The `index.tsv` manifest at the repo root is the source of truth for every other piece of documentation. Five tab-separated columns: `handle` (team-chosen lowercase ASCII identifier, can be re-used across rotations, reserved names include `official` for the community master and `key-<year>` for the annual isolation), `uid` (primary UID from the OpenPGP packet — display text, editable, NOT a security property), `fingerprint` (40-char uppercase hex, SHA-1 of the public-key packet per RFC 4880 §12.2, 160-bit cryptographic commitment — THE trust anchor), `created` (YYYY-MM-DD, informational only), and `purpose` (free-text, convention not enforced). Pin to fingerprint, never to handle, UID, or short key-id: handle is a label, UID is editable, short key-id is truncated 32 bits and vulnerable to collision attacks. For "is the signing key for artifact X the one in index.tsv?": extract the fingerprint from the artifact's signature, look it up in index.tsv, and only trust if it matches. Three-way fingerprint cross-check across index.tsv, GitHub raw, and the team site gives the strongest assurance.