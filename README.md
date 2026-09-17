# x-cmd/gpg — x-cmd team's public keys

The canonical, signed-by-the-team-elsewhere, X-only list of
GPG public keys for the x-cmd core team. Every key listed
here is published under `keyring/` in this repo as an
ASCII-armored file (`keyring/<handle>.asc`) and aggregated into
the concatenated `keyring/keyring.asc` keyring.

> 🌐 **中文版：[README.cn.md](./README.cn.md)** — same catalog,
> Chinese front matter.
>
> - **[Current keys](#current-keys)** — who, what fingerprint,
>   what the key signs.
> - **[How to verify](#how-to-verify-a-key)** — fetch the
>   keyring over HTTPS, import into GnuPG, compare the
>   fingerprint against what you saw on the website.
> - **[Key rotation](#key-rotation)** — what happens when a key
>   expires or a team member rotates.
> - **[Security policy](#security-policy)** — short version of
>   [`CONTRIBUTING.md`](./CONTRIBUTING.md). TL;DR: this repo is
>   maintained by the x-cmd team only; PRs touching `keyring/` are
>   closed without merge.
> - **[FAQ](#faq)** — the questions people actually ask.
>
> For end users: [`SKILL.md`](./SKILL.md) — how to consume this
> repo with `x gpg` and with raw `gpg`. For maintainers:
> [`CONTRIBUTING.md`](./CONTRIBUTING.md) — internal pipeline.

## Current keys

The table below is regenerated from [`index.tsv`](./index.tsv)
on every release commit by the team. **Pin to the
fingerprint, not the handle** — handles can be re-used across
rotations; fingerprints cannot.

<!-- BEGIN keys.md -->

| Handle | UID | Fingerprint | Created | Purpose |
| --- | --- | --- | --- | --- |
| _(no keys published yet — see_ [`CONTRIBUTING.md`](./CONTRIBUTING.md)_)_ | | | | |

<!-- END keys.md -->

To bulk-import every key in this repo:

```sh
# Download the concatenated keyring (~3 KB per key)
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

Or one key at a time:

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import
```

## How to verify a key

A bare fingerprint in a README is **not** a proof — anyone can
type 40 hex characters. Verification is a three-step process:

1. **Fetch** the key over a transport you already trust.
   `https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc`
   is fine if you trust GitHub; for higher assurance, fetch the
   same file from a second source (e.g. the team's site, a
   signed release tarball) and compare the resulting fingerprints.
2. **Import** into GnuPG: `gpg --import keyring/keyring.asc`.
3. **Compare** the resulting fingerprint against the value in
   `index.tsv` *and* against the value on the team's website.
   All three sources must agree on the fingerprint — that
   three-way match is your proof.

The [`x gpg`](https://x-cmd.com/mod/gpg) shell module automates
this — `x gpg import <handle>` fetches, imports, and prints the
fingerprint, then prompts you to confirm it matches the value
in `index.tsv` before writing the trust database.

### Fingerprint math

A GPG v4 fingerprint is the 40-hex-char SHA-1 of the key's
public-key packet. Two keys with the same fingerprint are, by
definition, the same key — there is no second source of identity
beyond the bits themselves. That is why pinning to a fingerprint
is pinning to a concrete, byte-level commitment, not a name.

## Key rotation

When a key expires or is rotated, the old `keyring/<handle>.asc`
moves to `keyring/archive/<handle>.<created-date>.asc`, the new key
takes the `keyring/<handle>.asc` slot, and `index.tsv` is updated
in the same commit. The README's "Retired keys" section is
generated from `keyring/archive/` on every release — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md) for the procedure and
the cryptographic-transition-statement requirement.

A retired key is kept in the archive forever, **signed by the
team's primary key**, so consumers with the team's key already
imported can verify the chain of custody. The retired key's
fingerprint never reappears under a new handle.

## Security policy

> **Pull from x-cmd's official channels only.** This GitHub
> repo and the team site are the only authorized sources for
> the keys under `keyring/`. **Proxy redistribution** — third-party
> mirrors, CDN / reverse-proxy / caching-proxy re-serving
> (jsdelivr, gcore, statically, or any service that
> automatically proxies raw.githubusercontent.com), public-
> keyserver uploads, bundling into other packages — is
> **not** authorized by the LICENSE. See [`LICENSE`](./LICENSE).
> If you see these keys served from any other domain, treat
> them as untrusted.

> **This repository is maintained exclusively by the x-cmd core
> team.** External PRs that touch `keyring/`, `index.tsv`, or any
> other trust-bearing file will be closed without merge.

The reason: every consumer of these keys — `x gpg`, package
mirrors, release tarballs — uses the fingerprint as a trust
anchor. A malicious PR swapping a real fingerprint for a
look-alike one is a supply-chain attack, not a contribution.

> **Have a suggestion for an article (a key description, a
> line of docs, a typo)?** Open an
> [issue](https://github.com/x-cmd/gpg/issues). Issue threads
> are public, reviewable, and feed into the team's
> documentation backlog.

Full version, including the three concrete consequences for
non-team contributors, in [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## License

**Copyright 2026 x-cmd — All Rights Reserved.** See
[`LICENSE`](./LICENSE) for the full text. The repository is
publicly available for the limited purpose of fetching and using
the GPG public keys under `keyring/` for signature verification;
modification, redistribution of modified versions, and
commercial use require prior written permission from x-cmd.

## FAQ

**Why a separate repo for keys, instead of `x-cmd/x-cmd`?**
Three reasons. (1) Releases are signed by these keys; the
`x-cmd/x-cmd` repo's release workflow needs to fetch the
signing keys from a location that's stable, public, and
*not* signed by itself — circular. (2) Rotations are
infrequent but security-critical; they want their own
small, focused repo with its own review surface. (3) `x gpg`
needs a small offline-friendly source for the keys, one that
doesn't depend on cloning the full `x-cmd/x-cmd` monorepo.

**Why ASCII-armored `.asc` files instead of binary `.gpg`?**
Armored files survive copy-paste into emails, chat, and the
GitHub web UI without corruption. They diff cleanly. The
`gpg --import` parser accepts both formats identically.

**How do I report a compromised key?**
Not through this repo. Contact the team directly through the
channels listed on [x-cmd.com](https://x-cmd.com). The team
will publish a cryptographic transition statement signed by
the *old* key, rotate to the new key, and add a row to
[`index.tsv`](./index.tsv).

**Does this repo publish the team's signing policy / CPS?**
No. The team follows the standard GPG best-practice of
publishing the public half only, with the `purpose` column
in `index.tsv` documenting *what* each key signs. For the
broader trust-policy document, see the team's site.

**Can I host a mirror, or operate a CDN / proxy that re-serves
this repo?**
**No, not without prior written permission.** This repo and
the team site are the only authorized sources. Third-party
mirrors, CDN / reverse-proxy / caching-proxy re-serving
(jsdelivr, gcore, statically, or any service that
automatically proxies raw.githubusercontent.com), public
keyserver uploads (keys.openpgp.org, keyserver.ubuntu.com,
etc.), and bundling into other packages are explicitly
forbidden by [`LICENSE`](./LICENSE). The reason is the
trust-anchor problem: a mirror or proxy that serves a
substituted fingerprint silently breaks every consumer that
trusts it; even an honest CDN cache returns stale bytes
during a key rotation. Pin your tooling to GitHub and fetch
directly every time.

## FAQ — supply chain & commercial deployment

The questions below cover x-cmd's supply-chain keying strategy
and the rationale behind the `x-cmd/gpg` repo's role in it. They
are written for both end users and for the procurement / security
review teams that vet x-cmd before enterprise rollout.

### Q1: GPG vs. GPG Key — what's the difference?

GPG (the software) is the smart lock: it executes the encryption,
decryption, and signature-verification actions. A GPG Key (the
data) is the actual public/private keypair — a digital credential
that's safe to publish. In the x-cmd ecosystem, the engine code
lives in the [`x gpg`](https://x-cmd.com/mod/gpg) shell module
inside `x-cmd/x-cmd`, while the keys and the signing history
are independently hosted in this repo (`x-cmd/gpg`).

### Q2: With Sigstore around, why does x-cmd still sign RPM/DEB with GPG?

Because the native Linux package managers trust GPG. `apt`,
`dnf`, and `yum` don't read Sigstore attestations — they read
the `gpg` signature embedded in the RPM/DB header. Sigstore is
becoming the standard for cloud-native supply-chain artifacts,
but the OS-level package channel is still GPG-only. To ship
through official OS channels with a single, vendor-blessed
install command, GPG has to be in the loop.

### Q3: Why drop the unsigned package variant entirely?

Keeping an unsigned `.rpm` alongside the signed one is a supply-
chain weakness: every scanner that catches the unsigned build will
flag it, every procurement checklist that asks "is this signed?"
will flag it, and a determined attacker will reach for the unsigned
artifact as the easiest pivot point. The unsigned state lives only
in CI memory and at no point touches a release asset. The
published surface is signed end-to-end by a layered trust
architecture (a "restricted" master key plus a per-year isolation
key) so there is no unsigned mirror to attack.

### Q4: Which package variants does x-cmd publish?

Two signed artifacts per release, each with its own key:

1. **`x-cmd.rpm` / `x-cmd.deb`** — community edition. Signed
   with the team's unrestricted master key. One import, every
   future upgrade verifies silently with zero ongoing admin.
2. **`x-cmd-annual-<year>.rpm` / `x-cmd-annual-<year>.deb`** —
   enterprise / compliance edition. Signed with that year's
   isolation key (e.g. `key-2026`). Targeted at finance and
   government procurement teams that require strict year-on-year
   asset isolation in their audit trails.

### Q5: Why "no expiry" cryptographically but "annual rotation" operationally?

Two competing forces. Cryptographic expiry (a hard `expire`
date on the key) is great hygiene — it bounds the damage from
a compromised key. But in production it creates a hard SLA
boundary: the moment the key expires, every install of every
artifact signed by it starts failing `gpg --verify` on
security-scanned hosts, including ones that are mid-upgrade,
mid-rollback, or simply unattended. In an enterprise environment
that's a paging incident waiting to happen — and a service-credit
dispute waiting to be filed.

The team's answer: **no cryptographic expiry, strict operational
isolation.** The annual key only signs that calendar year's
artifacts; at the year boundary the private half is sealed
away and a new key takes over. Consumers can verify artifacts
spanning the rotation without ever hitting a "key expired"
error, while every year-bound audit log stays clean.

### Q6: Why not 398 / 397 / 380 days — the SSL/TLS CA-Browser Forum magic numbers?

Because the magic numbers are tied to public-TLS-cert lifetimes,
which the CA/B Forum tightened through 2026 down to under 200
days. Code-signing is a separate ecosystem and isn't bound by
that timeline. Reusing the old 398-day number in a code-signing
context dates you: a sharp-eyed security reviewer will read it
as a copy-paste from the wrong standard and downgrade the
overall posture review. A clean per-calendar-year boundary
(`365 days` in business terms, "no expiry" cryptographically)
reads as deliberate and intentional.

### Q7: When the annual key rotates, do consumers' systems need the old key removed?

**No — never auto-remove.** If the enterprise builds automate
key removal on a year boundary, every host with an older
artifact installed will start failing its own daily signature
scans ("signed-by-key unknown"), and a routine security scan
becomes a production outage. The old key has to stay installed
alongside the new one.

x-cmd's answer: the repo publishes both the current key and
every historical key in `keyring/archive/`. The team takes no
position on whether a particular compliance regime should
eventually prune the archive; that decision, and its operational
consequences, are left to each operator's security team.

### Q8: A paying enterprise in 2027 wants to install a 2025-era artifact. How is that signed?

Through **repackage / resign lifecycle**: an older artifact can
be re-signed with the current key without recompiling or
modifying the upstream binary. The CI workflow runs
`rpmsign --addsign` against the existing release asset to
stamp a new signature header; the bytes inside the package are
unchanged.

The re-signed packages are publicly hosted (transparency is
part of the value proposition), but **which historic versions**
re-sign and **how often** is a service-tier decision — the team
does not maintain re-signed older builds for free users. Long-
term-support re-signing is a paid LTS subscription feature.

### Q9: Why `x-cmd/gpg` and not `x-cmd/gpgkeyring`?

Brand consistency and zero cognitive load:

- Tool module: `x gpg` (shell command)
- Public-key repo: `x-cmd/gpg` (this GitHub repo)
- Official access path: `https://x-cmd.com/gpg/` (via GitHub
  Pages, see Q10)

Three different surfaces, one word. End-users and CI scripts
never have to remember which spelling goes where.

### Q10: What's the single shortest import command for x-cmd's trust anchor?

```sh
# RHEL / CentOS / Rocky / Fedora — one-shot root-of-trust import
sudo rpm --import https://x-cmd.com
```

That URL resolves through GitHub Pages to this repo's
`keyring/keyring.asc`. The same path is also reachable as
`https://github.com/x-cmd/gpg` for users who prefer to pin to
GitHub directly — both are first-party; both are sanctioned by
[`LICENSE`](./LICENSE). Third-party CDN / mirror / proxy paths
are explicitly not sanctioned (see the LICENSE footer).

## Related

- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — module source (`mod/gpg/`)
- [`x gpg` module docs](https://x-cmd.com/mod/gpg) — consumer (shell)
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — maintainer docs
- [`SKILL.md`](./SKILL.md) — usage recipes