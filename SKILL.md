---
name: gpg
description: Authoritative list of x-cmd team's GPG public keys. Use when the user asks for "x-cmd public key", "x-cmd signing key", "x gpg", "verify x-cmd release", "import x-cmd keyring", or wants to look up a fingerprint / UID / purpose for any x-cmd maintainer.
metadata: type=keyring, source=team-curated, schema=tsv-5-col, refresh=manual, license=see-LICENSE, scope=gpg-public-keys
---

# x-cmd/gpg — using the team's public keys

Two consumption paths. Pick whichever fits the workflow.

## 1. Direct curl + gpg (no install)

The repo is plain text + ASCII-armored keyring, served over
HTTPS from GitHub's raw content CDN.

```sh
# Whole keyring in one shot (every key the team has published)
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/keys.asc \
  | gpg --import

# One key only — <handle> matches pub/<handle>.asc on disk
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/<handle>.asc \
  | gpg --import

# Just the manifest, no key bytes
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/index.tsv
```

After import, `gpg --list-keys --fingerprint <KEYID>` shows the
fingerprint and UID; cross-check both against
[`index.tsv`](./index.tsv) *and* against the value on
[x-cmd.com](https://x-cmd.com) before trusting.

## 2. Use `x gpg` shell module (auto-caching + fingerprint check)

```bash
x gpg                          # list keys already in your local keyring
x gpg import                   # fetch + import the full x-cmd keyring
x gpg import <handle>          # one key only
x gpg info <handle>            # show fingerprint + UID + purpose, cross-checked
x gpg verify <sig> <file>      # verify a detached signature against an x-cmd key
x gpg -h                       # full help
```

`x gpg import` always prints the resulting fingerprint and asks
you to confirm it matches `index.tsv` before writing to the
trust database. That's the same three-step check as §1, but
without the manual curl pipeline.

## `index.tsv` schema (5 columns)

| # | Col | Type | Example | Meaning |
|---|---|---|---|---|
| 1 | handle | str | `lijunhao` | x-cmd handle (matches `pub/<handle>.asc`) |
| 2 | uid | str | `Li Junhao (x-cmd) <l@x-cmd.com>` | Primary UID from the key |
| 3 | fingerprint | str | `4E1C 1B9E 5C5F 0A2D 7B3C …` (40 hex, no spaces) | The trust anchor |
| 4 | created | date | `2024-03-15` | Key creation date (`gpg --list-keys --with-colons`) |
| 5 | purpose | str | `release-signing` | Free-text: what this key signs |

Rows are sorted by `handle` (ASCII, lexicographic). Plain
`sort` produces a stable diff.

## Common shell queries

```sh
# What fingerprint is the team's release-signing key?
awk -F'\t' '$5 == "release-signing" { print $1, $3 }' index.tsv

# All keys created in 2025
awk -F'\t' '$4 ~ /^2025-/' index.tsv

# Does a given fingerprint appear in the team's index?
grep -F "<40-HEX-FINGERPRINT>" index.tsv

# Cross-check a fingerprint in three places (GitHub, team site, your local copy)
fpr=$(awk -F'\t' '$1=="lijunhao"{print $3}' index.tsv | tr -d ' ')
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/lijunhao.asc \
  | gpg --show-keys --with-colons \
  | awk -F: '/^fpr:/{print $10}'
```

The last snippet is the bare minimum automated check — it
fetches the same file GitHub serves, parses the fingerprint out
of the GnuPG output, and lets you `diff` it against the value
in `index.tsv`. Match → the bytes GitHub shows are the bytes
the team published.

## Pinning to a fingerprint

For release verification, **always pin to the fingerprint,
not the UID or handle**. UIDs can be edited; handles can be
re-used across rotations; fingerprints cannot. The standard
recipe:

```sh
EXPECTED="4E1C1B9E5C5F0A2D7B3C…"   # copy-pasted from index.tsv column 3
ACTUAL=$(gpg --show-keys --with-colons pub/keys.asc \
  | awk -F: '/^fpr:/{print $10; exit}')
[ "$EXPECTED" = "$ACTUAL" ] || { echo "FINGERPRINT MISMATCH"; exit 1; }
```

For more elaborate workflows (signed commits on the
`x-cmd/x-cmd` repo, signed release tarballs), see
[`x gpg verify`](https://x-cmd.com/mod/gpg) — it implements
the same pin in a one-liner.

## Key rotation

When a key is rotated, the old `pub/<handle>.asc` is moved to
`pub/archive/`. The README's "Retired keys" table is generated
from that directory on every release. **Do not** trust a
retired key for *new* signatures — only for verifying old
artifacts that pre-date the rotation.

## Trust policy

This repo only publishes the public half of each keypair.
Trust decisions (tofu / web-of-trust / assigned-owner) are
the consumer's responsibility. The team recommends:

- **First import:** `x gpg import` (or the curl pipeline in §1)
  + cross-check the fingerprint against `index.tsv` and the
  team site. That's your one-time onboarding.
- **Signing new artifacts:** pin to the *current* fingerprint
  in `index.tsv` (`created` column = most recent).
- **Signing old artifacts:** pin to the matching fingerprint in
  `pub/archive/` — the README's retired-table gives the
  per-artifact-date fingerprint.

## Sources

- <https://github.com/x-cmd/gpg> — this repo (raw key bytes + manifest)
- <https://x-cmd.com> — team site (cross-check fingerprints)
- <https://x-cmd.com/mod/gpg> — `x gpg` shell module
- <https://x-cmd.com/mod/x-cmd> — broader x-cmd module docs

## Reporting

- **Doc / README / typo** → [issue](https://github.com/x-cmd/gpg/issues).
  This repo does not accept external PRs touching `pub/` or
  `index.tsv`. Full policy in
  [`CONTRIBUTING.md`](./CONTRIBUTING.md).
- **Compromised key** → contact the team directly via the
  channels on [x-cmd.com](https://x-cmd.com). Do not file an
  issue.