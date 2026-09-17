---
name: 2-how-the-keyring-is-published
description: Team-internal pipeline for adding a key — gpg --export, self-signed detached signature, index.tsv row, internal PR review, aggregate regeneration. CI checks: parse, fingerprint-vs-manifest, self-signature, uniqueness. Key rotation procedure with archive/ directory.
type: pipeline
---

# Core Content

core_features:

- Two-file layout: `keyring/<handle>.asc` (canonical per-key artifact) + `keyring/keyring.asc` (aggregate regenerated on every release)
- 5-column manifest: `index.tsv` with handle, uid, fingerprint, created, purpose
- New-key flow: `gpg --armor --export` → self-sign → append manifest row → internal PR (not open this repo) → second team-member review → release
- Aggregate regeneration: `cat keyring/<handle>.asc` in lex handle order, blank line between; CI fails if regenerated != committed
- CI gates: `gpg --list-packets` parse, fingerprint-vs-manifest drift check, self-signature validation, fingerprint uniqueness
- Rotation: cryptographic transition statement on team site + `keyring/archive/<handle>.<created-date>.asc` + `index.tsv` update in same commit
- External PRs touching `keyring/` or `index.tsv` are closed without merge (fingerprint swap = supply-chain attack)

## Key Information

highlights:

- The team is the only path to a commit on `keyring/` or `index.tsv` — every change needs an internal second-pair-of-eyes review
- `keyring/keyring.asc` is byte-stable across regenerations (lex handle order + blank line between keys)
- CI does NOT verify web-of-trust / trust signatures — that's the consumer's job via `x gpg` or `gpg --check-sigs`
- Archive keeps retired keys forever, signed by the team's primary key so consumers can verify chain of custody
- External verification options: `git log` on `<handle>.asc` (commit messages reference the maintainer and detached-signature filename), `.github/workflows/` files are themselves in git, GitHub-signed release tags verified with `git tag --verify`

## Use Cases

use_cases:

- Adding a new maintainer's key to the keyring
- Rotating a key at expiry or after a compromise
- Diagnosing a CI failure on a `keyring/` PR
- Forensically confirming a release's key bytes match the team's intent (cross-check `git log` + detached-signature + CI workflow file)

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  contributing: <https://github.com/x-cmd/gpg/blob/main/CONTRIBUTING.md>
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/gpg/`)

## Summary

The pipeline from a fresh `gpg --export` to a row in `index.tsv` plus bytes in `keyring.asc` is small by design — six gpg commands, a one-line manifest update, and a CI pass. The two-file layout (`keyring/<handle>.asc` for single-key consumers, `keyring/keyring.asc` for the aggregate, regenerated on every release in lex handle order) keeps per-key fetches cheap and the aggregate byte-stable. The team-internal flow for adding a key: export → self-sign → append manifest row → internal PR (not open this repo) → second team-member review (parse, fingerprint match, detached-signature verifies, valid self-signature) → release. CI gates every push with four checks: `gpg --list-packets` parse, fingerprint-vs-manifest drift detection, self-signature validation, fingerprint uniqueness. CI does NOT verify web-of-trust or trust signatures — that's the consumer's job. Rotation moves the old file to `keyring/archive/<handle>.<created-date>.asc`, paired with a cryptographic transition statement on the team site, and updates `index.tsv` in the same commit. External readers can confirm a release's integrity via `git log` on `<handle>.asc` (commit messages name the maintainer and the detached-signature file), reading the `.github/workflows/` files directly, and verifying GitHub-signed release tags with `git tag --verify`.