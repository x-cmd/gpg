---
x-title: Verifying a key
x-desc: The three-step fetch → import → compare recipe, with the look-alike attack, CDN-cache pitfall, and self-signature-vs-team-signature distinctions called out.
x-sidebar: Verifying a key
x-keywords: verify, fingerprint, three-way check, look-alike attack, cdn cache, self-signature, trust on first use
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Verifying a key'
      inLanguage: 'en'
      about: 'Practical key verification recipe'
---

# Verifying a key

A fingerprint in a README is **not** a proof — anyone can
type 40 hex characters. Verification is a three-step process
that ties the bytes you imported to the team's intent, using
channels that don't all rely on the same source. This
article walks through the recipe, the pitfalls, and what
each step actually proves (and what it doesn't).

## The three steps

### 1. Fetch

Pull the key over a transport you already trust.

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc
```

`https://raw.githubusercontent.com` is fine if you trust
GitHub. For higher assurance, fetch the same file from a
second source — the team site, a signed release tarball,
a colleague's verified clone — and verify the bytes are
identical.

**Do not** route through any CDN, reverse proxy, or caching
service. See [`LICENSE`](../LICENSE) and the FAQ for the
trust-anchor rationale.

### 2. Import

Pipe the fetched bytes into GnuPG:

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

GnuPG parses the OpenPGP packet stream, validates each key's
self-signature (more on this below), and adds the keys to
your local keyring.

### 3. Compare

The fingerprint GnuPG shows you has to match the value in
`index.tsv` *and* the value on the team's site. All three
sources must agree.

```sh
# Fingerprint from your local keyring (after import)
gpg --list-keys --with-colons keyring/keyring.asc \
  | awk -F: '/^fpr:/{print $10}'

# Fingerprint from index.tsv (in this repo)
awk -F'\t' 'NR > 0 { print $3 }' index.tsv

# Fingerprint from the team site
curl -fsSL https://x-cmd.com/gpg/ | grep -oE '[0-9A-F]{40}'
```

The three fingerprints must match exactly. If any one of them
differs, **stop** and investigate before trusting the key.

## What each step proves

| Step | Proves                                                   | Doesn't prove                                       |
| ---  | ---                                                      | ---                                                 |
| Fetch | The bytes GitHub serves are the bytes you got          | That those bytes are what the team published        |
| Import | The bytes are a valid OpenPGP key with valid self-signatures | That the key is *the team's* key, not an impostor's |
| Compare (3-way) | The bytes you have were trusted; they match an independent source the team publishes | That the *team* is who they claim to be (out of scope) |

The "Compare" step is the load-bearing one. Without it,
importing a key proves only that you got a syntactically
valid OpenPGP blob — an impostor can satisfy that bar
trivially. With it, you have cryptographic agreement
across three sources.

## Self-signature vs. team-signature

GnuPG's `gpg --verify <key>.asc` (or just `gpg --import`
on a fresh keyring) validates the key's *self-signature* —
a signature that the key's own private half placed on its
own public-key packet. Every OpenPGP key has one — that's
how GnuPG knows the key isn't structurally malformed.

This is **not** the same as the team endorsing the key.
A self-signature says "this key is well-formed", not "this
key belongs to x-cmd". For team endorsement, you need:

- The bytes in this repo's `keyring/<handle>.asc` matching
  the team's commit history.
- The fingerprint matching `index.tsv` and the team site.
- (For deeper assurance) a cross-signature from another
  team member's key — which the team publishes as part of
  the rotation procedure described in
  [2. How the keyring is published](./2-how-the-keyring-is-published.md#key-rotation).

If you're verifying a single key for the first time, all
three of the above are typically overkill — the three-way
Compare is sufficient. If you're onboarding a critical
production environment or auditing for compliance, layer in
the cross-signature check too.

## Common pitfalls

### The CDN cache

The single most common way to import a *stale* key without
realizing it. A CDN or reverse proxy between you and
GitHub can return bytes from a few seconds ago, a few
minutes ago, or a few hours ago — including bytes that
predate the current rotation. The fingerprint will still
look valid (the bytes are still a real OpenPGP key), but it
won't match the team's current `index.tsv`.

**Defense**: pull from GitHub directly every time, with no
intermediate. Compare against an independent source (the
team site). See [`LICENSE`](../LICENSE) for the explicit
proxy-redistribution clause.

### The look-alike key

An attacker constructs a key with a UID that matches the
team's (same name, same email) but a different fingerprint.
You `gpg --search-keys <email>` on a public keyserver, see
a UID that matches, and import — without checking the
fingerprint.

**Defense**: never search keyservers for a known team's
key. Fetch from this repo directly. Pin to fingerprint.

### The handle re-use

A team rotates a key; the new key takes the same handle.
Your tooling pins to "the official key", which matches the
new fingerprint — but your audit log now references the
old fingerprint and the new fingerprint interchangeably.

**Defense**: pin to fingerprint, not handle. When reviewing
your own audit logs, always use the fingerprint as the
identifier.

### The short key ID

Some old recipes pin to the 32-bit short key ID
(`0xDEADBEEF`). Truncating 160 bits to 32 bits is reversible
in practice — an attacker can construct a key whose
truncated short ID matches a target. Don't use the short
key ID for pinning.

**Defense**: always use the full 40-character fingerprint.

## Trust on first use (TOFU)

If you can't establish a three-way Compare (e.g. you have
no way to reach the team site), you're reduced to
"trust the bytes GitHub serves". For most consumer use
cases that's fine — `https://raw.githubusercontent.com`
is GitHub's HTTPS endpoint, pinned by GitHub's certificate,
verified by your OS trust store. The remaining attack
surface is "GitHub serves the wrong bytes", which GitHub
treats as a security incident at the platform level.

If your threat model includes "GitHub serves the wrong
bytes", you need out-of-band verification — typically, a
signed message from the team's other channels confirming
the new fingerprint. The team's site publishes such
messages when a key rotates.

## What to read next

- [6. Three ways to consume the keyring](./6-three-ways-to-consume.md) —
  raw curl, `x gpg`, and the GitHub-Pages-via-x-cmd.com
  redirect.
- [3. Reading the key catalog](./3-reading-the-key-catalog.md) —
  fingerprint as a cryptographic commitment, in detail.