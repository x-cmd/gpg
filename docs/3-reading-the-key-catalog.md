---
x-title: Reading the key catalog
x-desc: The `index.tsv` schema in detail — every column, the fingerprint math, handle naming conventions, and why "pin to fingerprint, not handle" is the only safe rule.
x-sidebar: Reading the key catalog
x-keywords: index.tsv, fingerprint, sha-1, openpgp packet, handle naming, pin to fingerprint
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Reading the key catalog'
      inLanguage: 'en'
      about: 'index.tsv schema and fingerprint math'
---

# Reading the key catalog

The `index.tsv` file at the repo root is the manifest every
other piece of documentation points at. This article walks
through every column, the cryptographic math behind the
fingerprint column, the conventions for the handle column,
and the rule that makes all of this safe: **pin to the
fingerprint, never to the handle**.

## Schema — 5 columns, tab-separated

```tsv
handle	uid	fingerprint	created	purpose
```

No header in the file (header is documentation only). Rows
are sorted by `handle` (lexicographic, ASCII), so a plain
`sort` produces a stable diff — useful when you're reviewing
a rotation PR.

## Column 1 — `handle`

The team's internal identifier for the key. Lowercase ASCII,
no spaces. Same identifier used in the team's other repos
(handles in the `x-cmd` monorepo, the team site, GitHub
commit attribution). Matches the file path
`keyring/<handle>.asc` on disk.

**Handles can be re-used across rotations.** If a maintainer
rotates their key, the *new* key may take the same handle —
only the fingerprint (column 3) tells you it's a different
key. This is why "pin to fingerprint" is the rule and
"pin to handle" is not.

Reserved-by-convention handles (when populated):

| Handle       | Purpose                                                          |
| ---          | ---                                                              |
| `official`   | Team's unrestricted master key. Signs the community package.    |
| `key-<year>` | Annual isolation key for that calendar year.                     |

These names reflect the team's supply-chain keying strategy
described in detail in
[4. Annual key strategy explained](./4-annual-key-strategy-explained.md).
This repo doesn't enforce the names — they're a team
convention, not a validation rule.

## Column 2 — `uid`

The key's primary UID as published in the OpenPGP packet —
typically `Name (comment) <email>`. This is *display text*;
it's editable by the key owner. A UID collision between two
keys (e.g. an impostor uploads a key claiming the same
email) is **not** a security property — only the fingerprint
is.

When the team updates a UID (e.g. an email change), the
`uid` column in `index.tsv` changes; the `fingerprint`
column does not. So `index.tsv` revision history can show
UID-only changes, fingerprint-only changes, or both.

## Column 3 — `fingerprint` — the trust anchor

40-character uppercase hexadecimal string, no spaces
(some tooling inserts spaces every 4 chars for readability —
`4E1C 1B9E 5C5F 0A2D 7B3C …` — but the canonical form in
`index.tsv` is unspaced). Computed as the SHA-1 of the
key's *public-key packet*, per RFC 4880 §12.2.

What this means concretely: a fingerprint is a 160-bit
cryptographic commitment to the exact bytes of the public
key. Two keys with the same fingerprint are, by
construction, the same key. There is no other source of
identity — no "key id", no name, no email — that pins to
the same level of cryptographic certainty.

### Why pin to fingerprint, not handle or UID

- **Handle** — chosen by the team, can be re-used across
  rotations. It's a label, not an identity.
- **UID** (`Name <email>`) — editable by the key owner. A
  look-alike attacker can construct a key with the same
  name and email but a different fingerprint, and the UID
  column won't help you tell them apart.
- **Key ID** (the short `0xDEADBEEF` form, often displayed
  by `gpg --list-keys`) — derived from the fingerprint, but
  truncated to 32 bits. Vulnerable to *key-ID collision
  attacks*: an attacker can construct a key whose truncated
  32-bit key ID matches a target. Not a cryptographic
  commitment.
- **Fingerprint** — full 160 bits. Collision resistance is
  bounded only by the SHA-1 preimage space (2^160). No
  practical collision attack exists.

So when a CI pipeline, an `apt`/`dnf` repo, or a `x gpg`
caller pins to "is the signing key for artifact X the one
in `index.tsv`?", the answer has to be: extract the
fingerprint from the artifact's signature, look it up in
`index.tsv`, and only trust the artifact if the fingerprint
matches. Anything weaker than that is vulnerable to
substitution.

## Column 4 — `created`

`YYYY-MM-DD` — the date the key was created, per
`gpg --list-keys --with-colons` (`pub` line `creation-date`
field, ISO 8601 format). This is purely informational —
useful for understanding the history of the keyring ("which
keys are 2 years old and might be nearing their self-imposed
rotation window?") but not a security property.

If a key was created, rotated, and re-created under the same
handle, two rows in `index.tsv` would exist for that handle
(at different points in time, with different fingerprints).
The `created` column lets you disambiguate them.

## Column 5 — `purpose`

Free-text, lowercased, hyphenated. The team uses values like
`release-signing`, `package-signing`, `mirror`, etc. There's
no formal enumeration — the column documents what each key
*is for*, so a consumer's tooling can answer "show me the
key for the 2026 enterprise package" with a single
`awk` filter.

Examples of purpose values the team plans to use:

| Purpose             | What this key signs                                          |
| ---                 | ---                                                          |
| `release-signing`   | Source tarball signatures published to GitHub Release.       |
| `package-signing`   | RPM / DEB / container image signatures for `x-cmd.<ext>`.    |
| `package-signing-yearly` | Annual enterprise package signatures (`x-cmd-annual-<year>`). |
| `mirror-signing`    | Signatures for the team's mirror buckets / CDN uploads.      |

Again, the values are conventions, not enforced. The CI
checks that `index.tsv` parses and that fingerprints are
unique — it doesn't validate purpose strings.

## Practical queries

```sh
# What fingerprint is the team's release-signing key?
awk -F'\t' '$5 == "release-signing" { print $1, $3 }' index.tsv

# All keys created in 2025
awk -F'\t' '$4 ~ /^2025-/' index.tsv

# Does a given fingerprint appear in the team's index?
grep -F "<40-HEX-FINGERPRINT>" index.tsv

# Cross-check a fingerprint in three places (GitHub, team site, your local copy)
fpr=$(awk -F'\t' '$1=="official"{print $3}' index.tsv | tr -d ' ')
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/official.asc \
  | gpg --show-keys --with-colons \
  | awk -F: '/^fpr:/{print $10}'
diff <(echo "$fpr") <(curl ... | awk ...)
```

The last snippet is the bare minimum automated check — it
fetches the same file GitHub serves, parses the fingerprint
out of the GnuPG output, and lets you `diff` it against the
value in `index.tsv`. Match → the bytes GitHub shows are the
bytes the team published. Mismatch → stop and investigate
before trusting the key.

## What to read next

- [4. Annual key strategy explained](./4-annual-key-strategy-explained.md) —
  the long-form version of FAQ Q4–Q8.
- [5. Verifying a key](./5-verifying-a-key.md) — the three-step
  fetch → import → compare recipe.
