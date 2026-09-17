---
x-title: Annual key strategy — design exploration
x-desc: A design the x-cmd team is considering: the two-key split (community + annual), why "no expiry" cryptographically but "annual rotation" operationally, and the repackage / resign lifecycle for users who need an older artifact re-signed with the current year's key. **Exploratory only — not yet implemented.**
x-sidebar: Annual key strategy — design exploration
x-keywords: annual key, enterprise, supply chain, no-expiry, rotation, repackage, resign, lts, annual isolation, key-2026, design exploration
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Annual key strategy — design exploration'
      inLanguage: 'en'
      about: 'x-cmd supply-chain keying strategy under exploration'
---

# Annual key strategy — design exploration

> **Status: exploratory.** As of this writing the x-cmd team
> has not yet adopted a multi-key release pipeline. This
> article walks through a *candidate* design — what the
> two-key split would look like, why "no expiry"
> cryptographically but "annual rotation" operationally is a
> defensible shape, and how the repackage / resign lifecycle
> would work — and is offered as analysis, not as a description
> of what is in production today. Treat the rest of the
> article as "if we adopted this design, here's how it would
> behave and what trade-offs it would create."

The candidate design has two parts that look unusual at
first glance: two published keys instead of one, with one
of them rotated every calendar year but cryptographically
set to never expire. This article is the long-form
analysis. It walks through why the design has the shape it
would, what each design decision defends against, and where
the intentional gaps would be.

## The two-key model (candidate)

Under this design, every release would ship two signed
artifacts:

| Artifact                          | Signed by         | Purpose                                                              |
| ---                               | ---               | ---                                                                  |
| `x-cmd.rpm` / `x-cmd.deb`         | `official` key    | Community edition. One import, every future upgrade verifies silently. |
| `x-cmd-annual-<year>.rpm` / `.deb`| `key-<year>` key  | Compliance edition. Annual isolation for finance / government audits.   |

Both keys would live in this repo under `keyring/`, both
documented in `index.tsv`, both aggregated into
`keyring/keyring.asc`. The design calls for publishing both
because the two audiences have different threat models and
operational constraints — bundling them into a single key
would force one audience to accept the other's trade-offs.

The community edition would optimize for **friction-free
upgrades**. A user installs the package once, the master key
gets imported, and every subsequent package — including
security patches, point releases, and version bumps —
verifies silently without any user intervention. There would
be no "renew your key every year" chore, and no
calendar-bound moment where an unattended host suddenly
starts failing signature checks.

The annual edition would optimize for **strict year-on-year
auditability**. A finance or government team needs to be
able to say "everything we deployed in 2026 was signed by
the 2026 isolation key, and that key never signed anything
from 2025 or 2027". The cost is a once-a-year key rotation;
the benefit is a clean per-year audit trail.

## "No expiry" cryptographically, "annual" operationally

The most counterintuitive piece of the design is the annual
key's expiry: it would have **none**. Cryptographically, the
private key would be set to never expire. Operationally, it
would be treated as if it had — the `key-2026` private key
only signs 2026 artifacts, and at the year boundary the
private half is "sealed" (rendered unusable for new
signatures) and `key-2027` takes over.

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

### The compromise under consideration

The design picks the best of both: **no cryptographic expiry,
strict operational isolation**. The `key-2026` private key
would never expire on the wire; clients with installed
artifacts spanning the 2026 → 2027 boundary would keep
verifying both. But the private half is physically sealed at
year-end so it can't sign anything new, and a fresh
`key-2027` takes over for 2027's artifacts. Every year-bound
audit trail stays clean because no artifact is signed by both
keys.

Consumers would verify artifacts spanning the rotation
without ever hitting a "key expired" error, while every
year-bound audit log is unambiguous about which key signed
what.

## Why not 398 / 397 / 380 days

These numbers — particularly 398 — are tied to the Web PKI
ecosystem (SSL/TLS certificates). The CA/Browser Forum
shortened public-TLS lifetimes through 2026 to under 200
days. Code-signing is a separate ecosystem and is not bound
by that timeline, but reusing the old 398-day number in a
code-signing context reads as a copy-paste from the wrong
standard. A sharp-eyed security reviewer would treat the
overall posture review as "the team doesn't understand
which standards apply to which".

Clean per-calendar-year boundary (`365 days` in business
terms, "no expiry" cryptographically) reads as deliberate
and intentional. It also lines up with the way
finance / government procurement teams already structure
their annual audit cycles — the technical rotation
coincides with the business year, which makes the rotation
explainable to non-technical stakeholders.

## Old keys stay available after rotation

If a consumer's automation removes the old annual key at
the year boundary, every host with an older artifact
installed will start failing its own daily signature scans
("signed-by-key unknown"), and a routine security scan
becomes a production outage.

The design's neutral stance: the repo publishes both the
current key (in `keyring/`) and every historical annual key
(in `keyring/archive/`). Consumers can verify artifacts
spanning the rotation without ever hitting a "key missing"
error. This is the autonomy we leave to the user — the
team's role ends at "make the bytes available"; the operator
decides the local policy.

If a particular compliance regime requires eventual pruning
of historical keys, that's the operator's call to make on
their own hosts — the team takes no position on it. The
"incompatibility-with-your-compliance" pot belongs to the
operator who made the choice.

## Repackage / resign lifecycle (under consideration)

An enterprise customer in 2027 wants to install a 2025-era
artifact. The artifact's original signature was made by
`key-2025`, which they may or may not have imported. Even
if they have it, they may want the artifact re-signed under
`key-2027` so their 2027 audit log is clean.

The design supports this through **repackage / resign
lifecycle**: an older artifact can be re-signed with the
current year's key without recompiling or modifying the
upstream binary. The CI workflow would run
`rpmsign --addsign` against the existing release asset to
stamp a new signature header; the bytes inside the package
are unchanged. The re-signed package would be published as
a new artifact, and both versions (original-signed-by-key-
2025, re-signed-by-key-2027) remain in the release history.

### Re-signing handled on demand

Re-signed older builds would be publicly hosted alongside
the original-signed versions, so consumers can pick whichever
matches the keys they've imported.

The team wouldn't proactively re-sign every historical
artifact for every release cycle. Anyone can verify any
historical artifact using the corresponding historical key
from `keyring/archive/` — that's the GPG default and doesn't
require any action from the team. Customers who want the
team to take on the re-signing labor can request specific
artifacts to be re-signed through the team's support
channels; whether and how fast a particular re-sign happens
depends on the maintenance relationship with that customer.

## How this *would* map to `index.tsv` (illustrative)

The two-key design would live entirely in the repo's
manifest. **The following table is illustrative only — no
such rows exist in `index.tsv` today.** It's here to show
what the schema would look like if the team adopted the
design:

```tsv
handle          uid              fingerprint    created     purpose
official        <uid>            <fingerprint>  <date>      release-signing,package-signing
key-2026        <uid>            <fingerprint>  <date>      package-signing-yearly
key-2027        <uid>            <fingerprint>  <date>      package-signing-yearly
```

Under this design, `key-2025` and earlier years would
move to `keyring/archive/` once rotated.

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
- [7. Signing an RPM with GPG](./7-signing-an-rpm-with-gpg.md) —
  the practical `rpmsign` tutorial that the repackage /
  resign lifecycle above would invoke at release time.