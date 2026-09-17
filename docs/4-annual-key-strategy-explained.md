---
x-title: Annual key strategy explained
x-desc: The long-form version of FAQ Q4–Q8 — the two published keys (community + annual enterprise), why "no expiry" cryptographically but "annual rotation" operationally, the repackage / resign lifecycle, and the LTS economics.
x-sidebar: Annual key strategy explained
x-keywords: annual key, enterprise, supply chain, no-expiry, rotation, repackage, resign, lts, annual isolation, key-2026
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Annual key strategy explained'
      inLanguage: 'en'
      about: 'x-cmd supply-chain keying strategy'
---

# Annual key strategy explained

The team's supply-chain keying strategy looks unusual from the
outside — two published keys instead of one, with one of them
rotated every calendar year but cryptographically set to
never expire. This article is the long-form rationale. It
walks through why the strategy has the shape it does, what
each design decision defends against, and where the
intentional gaps are.

## The two-key model

Every release ships two signed artifacts:

| Artifact                          | Signed by         | Purpose                                                              |
| ---                               | ---               | ---                                                                  |
| `x-cmd.rpm` / `x-cmd.deb`         | `official` key    | Community edition. One import, every future upgrade verifies silently. |
| `x-cmd-annual-<year>.rpm` / `.deb`| `key-<year>` key  | Compliance edition. Annual isolation for finance / government audits.   |

Both keys live in this repo under `keyring/`, both are
documented in `index.tsv`, both are aggregated into
`keyring/keyring.asc`. The team publishes both because the
two audiences have different threat models and operational
constraints — bundling them into a single key would force
one audience to accept the other's trade-offs.

The community edition optimizes for **friction-free
upgrades**. A user installs the package once, the master key
gets imported, and every subsequent package — including
security patches, point releases, and version bumps —
verifies silently without any user intervention. There is
no "renew your key every year" chore, and there is no
calendar-bound moment where an unattended host suddenly
starts failing signature checks.

The enterprise / compliance edition optimizes for **strict
year-on-year auditability**. A finance or government team
needs to be able to say "everything we deployed in 2026 was
signed by the 2026 isolation key, and that key never signed
anything from 2025 or 2027". The cost is a once-a-year key
rotation; the benefit is a clean per-year audit trail.

## "No expiry" cryptographically, "annual" operationally

The most counterintuitive piece of the strategy is the
annual key's expiry: it has **none**. Cryptographically, the
private key is set to never expire. Operationally, the team
treats it as if it had — the `key-2026` private key only
signs 2026 artifacts, and at the year boundary the team
"seals" it (renders it unusable for new signatures) and
moves to `key-2027`.

Two competing forces drive this:

### Cryptographic hygiene wants a hard expiry

Standard GPG practice is to set an expiry date on the key
so that, in the worst case of a key compromise, the damage
is bounded in time. After the expiry, even an attacker in
possession of the private key can't forge signatures that
clients will accept (assuming the client checks expiry).

### Operations hates a hard expiry in production

In an enterprise environment, a hard expiry creates a
hard SLA boundary. The moment the key expires, every host
with an artifact signed by that key starts failing
`gpg --verify` on routine security scans — even hosts that
are mid-upgrade, mid-rollback, or simply unattended. A
"key expired" alert at 2 AM on a Sunday in an unattended
production environment is a paging incident waiting to
happen, and a service-credit dispute waiting to be filed.

### The compromise

The team picks the best of both: **no cryptographic expiry,
strict operational isolation**. The `key-2026` private key
never expires on the wire; clients with installed artifacts
spanning the 2026 → 2027 boundary keep verifying both. But
the team physically seals the key at year-end so it can't
sign anything new, and a fresh `key-2027` takes over for
2027's artifacts. Every year-bound audit trail stays clean
because no artifact is signed by both keys.

Consumers verify artifacts spanning the rotation without ever
hitting a "key expired" error, while every year-bound audit
log is unambiguous about which key signed what.

## Why not 398 / 397 / 380 days

These numbers — particularly 398 — are tied to the Web PKI
ecosystem (SSL/TLS certificates). The CA/Browser Forum
shortened public-TLS lifetimes through 2026 to under 200
days. Code-signing is a separate ecosystem and is not bound
by that timeline, but reusing the old 398-day number in a
code-signing context reads as a copy-paste from the wrong
standard. A sharp-eyed security reviewer will treat the
overall posture review as "the team doesn't understand
which standards apply to which".

Clean per-calendar-year boundary (`365 days` in business
terms, "no expiry" cryptographically) reads as deliberate
and intentional. It also lines up with the way
finance / government procurement teams already structure
their annual audit cycles — the technical rotation
coincides with the business year, which makes the rotation
explainable to non-technical stakeholders.

## Never auto-remove old keys on rotation

If a consumer's automation removes the old annual key at
the year boundary, every host with an older artifact
installed will start failing its own daily signature scans
("signed-by-key unknown"), and a routine security scan
becomes a production outage.

The team's position is: keep both keys installed. The repo
publishes both the current key (in `keyring/`) and every
historical annual key (in `keyring/archive/`). Consumers
can verify artifacts spanning the rotation without ever
hitting a "key missing" error.

If a particular compliance regime requires eventual pruning
of historical keys, that's the operator's call to make on
their own hosts — the team takes no position on it. The
"incompatibility-with-your-compliance" pot belongs to the
operator who made the choice.

## Repackage / resign lifecycle (LTS customers)

A paying enterprise customer in 2027 wants to install a
2025-era artifact. The artifact's original signature was
made by `key-2025`, which they may or may not have
imported. Even if they have it, they may want the artifact
re-signed under `key-2027` so their 2027 audit log is clean.

The team supports this through **repackage / resign
lifecycle**: an older artifact can be re-signed with the
current year's key without recompiling or modifying the
upstream binary. The CI workflow runs `rpmsign --addsign`
against the existing release asset to stamp a new
signature header; the bytes inside the package are
unchanged. The re-signed package is published as a new
artifact, and both versions (original-signed-by-key-2025,
re-signed-by-key-2027) remain in the release history.

### What's free vs what's paid

Re-signed older builds are publicly hosted (transparency
is part of the value proposition). However, **which historic
versions re-sign and how often** is a service-tier decision.
The team does not maintain re-signed older builds for free
users. Long-term-support re-signing is a paid LTS
subscription feature — finance / government customers
contractually paying for the right to ask "please re-sign
this artifact from 2024 with this year's key".

The pricing line is intentional: free users can verify any
historical artifact as long as they have the corresponding
historical key (which we keep in `keyring/archive/`
permanently), and paid LTS users get the convenience of
"every artifact carries a signature from the key I'm
currently using" without managing a per-year key inventory
themselves.

## How this maps to `index.tsv`

The two-key strategy lives entirely in this repo's
manifest:

```tsv
handle          uid              fingerprint    created     purpose
official        Li Junhao …      AAAA…          2022-03-15  release-signing,package-signing
key-2026        Li Junhao …      BBBB…          2026-01-04  package-signing-yearly
key-2027        Li Junhao …      CCCC…          2027-01-04  package-signing-yearly
```

(The above will populate as the team publishes each key.
`key-2025` and earlier years would move to
`keyring/archive/` once rotated.)

A consumer wanting to verify `x-cmd-annual-2026.rpm`:

```sh
# Pull the 2026 annual key directly
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/key-2026.asc \
  | gpg --import

# Confirm the fingerprint matches index.tsv
awk -F'\t' '$1=="key-2026" { print $3 }' index.tsv

# Verify the package
rpm -K x-cmd-annual-2026.rpm
```

A consumer wanting to verify both `x-cmd.rpm` (community)
and `x-cmd-annual-2026.rpm` (compliance):

```sh
# Pull both keys via the aggregate
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# Verify both packages
rpm -K x-cmd.rpm
rpm -K x-cmd-annual-2026.rpm
```

## What to read next

- [5. Verifying a key](./5-verifying-a-key.md) — the three-step
  fetch → import → compare recipe, with the look-alike and
  CDN-cache pitfalls called out.
- [6. Three ways to consume the keyring](./6-three-ways-to-consume.md) —
  raw curl, `x gpg`, and the GitHub-Pages-via-x-cmd.com
  redirect.