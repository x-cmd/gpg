# x-cmd/gpg — x-cmd team's public keys

The canonical, signed-by-the-team-elsewhere, X-only list of
GPG public keys for the x-cmd core team. Every key listed
here is published under `pub/` in this repo as an
ASCII-armored file (`pub/<handle>.asc`) and aggregated into
the concatenated `pub/keys.asc` keyring.

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
>   maintained by the x-cmd team only; PRs touching `pub/` are
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
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/keys.asc \
  | gpg --import
```

Or one key at a time:

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/<handle>.asc \
  | gpg --import
```

## How to verify a key

A bare fingerprint in a README is **not** a proof — anyone can
type 40 hex characters. Verification is a three-step process:

1. **Fetch** the key over a transport you already trust.
   `https://raw.githubusercontent.com/x-cmd/gpg/main/pub/keys.asc`
   is fine if you trust GitHub; for higher assurance, fetch the
   same file from a second source (e.g. the team's site, a
   signed release tarball) and compare the resulting fingerprints.
2. **Import** into GnuPG: `gpg --import pub/keys.asc`.
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

When a key expires or is rotated, the old `pub/<handle>.asc`
moves to `pub/archive/<handle>.<created-date>.asc`, the new key
takes the `pub/<handle>.asc` slot, and `index.tsv` is updated
in the same commit. The README's "Retired keys" section is
generated from `pub/archive/` on every release — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md) for the procedure and
the cryptographic-transition-statement requirement.

A retired key is kept in the archive forever, **signed by the
team's primary key**, so consumers with the team's key already
imported can verify the chain of custody. The retired key's
fingerprint never reappears under a new handle.

## Security policy

> **Pull from x-cmd's official channels only.** This GitHub
> repo and the team site are the only authorized sources for
> the keys under `pub/`. **Proxy redistribution** — third-party
> mirrors, CDN / reverse-proxy / caching-proxy re-serving
> (jsdelivr, gcore, statically, or any service that
> automatically proxies raw.githubusercontent.com), public-
> keyserver uploads, bundling into other packages — is
> **not** authorized by the LICENSE. See [`LICENSE`](./LICENSE).
> If you see these keys served from any other domain, treat
> them as untrusted.

> **This repository is maintained exclusively by the x-cmd core
> team.** External PRs that touch `pub/`, `index.tsv`, or any
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
the GPG public keys under `pub/` for signature verification;
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

## Related

- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — module source (`mod/gpg/`)
- [`x gpg` module docs](https://x-cmd.com/mod/gpg) — consumer (shell)
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — maintainer docs
- [`SKILL.md`](./SKILL.md) — usage recipes