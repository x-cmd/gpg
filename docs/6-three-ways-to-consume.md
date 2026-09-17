---
x-title: Three ways to consume the keyring
x-desc: Three consumption paths for the keyring — direct curl from GitHub, the `x gpg` shell module, and the GitHub-Pages-via-x-cmd.com redirect. Trade-offs, common recipes, and offline / air-gapped usage notes.
x-sidebar: Three ways to consume the keyring
x-keywords: curl, x gpg, github pages, x-cmd.com, air-gap, offline, bulk import, three-way fetch
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Three ways to consume the keyring'
      inLanguage: 'en'
      about: 'Keyring consumption paths and trade-offs'
---

# Three ways to consume the keyring

There are three first-party paths for fetching the x-cmd team
keyring bytes, each optimized for a different consumer. This
article walks through all three with concrete recipes, calls
out the trade-offs, and closes with the offline / air-gapped
recipe that finance / government environments typically need.

## Path 1 — Direct curl from GitHub

The simplest path: `curl` against `raw.githubusercontent.com`.

```sh
# Aggregate — every key in one shot
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# Single key by handle
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import

# Just the manifest (no key bytes)
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/index.tsv
```

**When to use**: any consumer that has direct internet
access to GitHub. CI pipelines, dev workstations, container
builds, package-build steps.

**Pros**:
- Zero dependencies. `curl` and `gpg` are preinstalled on
  every Linux distribution.
- Pull happens at the moment of use — no stale cache, no
  shared state to maintain.
- Trivially scriptable. No special tooling required.

**Cons**:
- Requires direct internet access to `raw.githubusercontent.com`.
- Each consumer repeats the same boilerplate
  (parse output, fingerprint check, trust store update).
- No awareness of the team's supply-chain strategy
  (community vs annual key) — that's on the consumer to
  encode.

## Path 2 — `x gpg` shell module

The x-cmd team publishes a shell module that wraps the
direct-curl path with caching, fingerprint cross-checking,
and a small set of high-level commands.

```sh
# One-shot import of the full keyring (with caching)
x gpg import

# Single key, with the fingerprint shown before trust is written
x gpg import <handle>

# Look up a key in the team's index without importing
x gpg info <handle>

# Verify a detached signature against an x-cmd key
x gpg verify <sig> <file>

# Sign a file with a team key (requires team-side access)
x gpg sign <file>

# List keys already in your local keyring
x gpg ls
```

**When to use**: end users who have `x` installed and want
to skip the boilerplate. Also useful for one-off lookups
("what's the team's release-signing fingerprint?") without
mutating the local keyring.

**Pros**:
- Caching — once you've imported, you have the bytes
  locally. Re-running `x gpg import` is cheap.
- Cross-check built in — `x gpg info` shows the fingerprint
  and prompts you to confirm it matches `index.tsv` before
  writing to the trust database.
- Knows about the team's two-key strategy — `x gpg info
  official` vs `x gpg info key-2026` return different
  metadata (community vs annual).
- Source code is in `x-cmd/x-cmd` (`mod/gpg/`); audit it
  the same way you'd audit any other x-cmd module.

**Cons**:
- Requires `x` to be installed. Adds a dependency on the
  `x-cmd/x-cmd` monorepo.
- The caching layer means you might be verifying against
  slightly stale bytes — `x gpg update` re-fetches.

## Path 3 — GitHub-Pages-via-x-cmd.com redirect

The team site at `https://x-cmd.com` exposes a subdomain
path (`https://x-cmd.com/gpg/`) that resolves to this
repo's `keyring/keyring.asc`. The redirect is set up via
GitHub Pages plus the team's own edge configuration, so
`https://x-cmd.com/gpg/keyring.asc` and
`https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc`
serve the same bytes — verified at release time by the
team's CI.

```sh
# Via the team's own domain
curl -fsSL https://x-cmd.com/gpg/keyring.asc \
  | gpg --import

# Or — short URL for RHEL/CentOS package verification
sudo rpm --import https://x-cmd.com
```

The second form is the one-line trust-anchor import for RPM
package verification: `rpm --import <URL>` reads whatever
bytes the URL serves into the system keyring.

**When to use**: any consumer who prefers to pin to the
team's own domain instead of `raw.githubusercontent.com`.
Especially useful for RHEL/CentOS package-import commands
where a short URL is more legible.

**Pros**:
- One URL fits both use cases (read-only fetch and
  `rpm --import`).
- Easier to remember than `raw.githubusercontent.com/...`.
- The team can move the bytes to a different backing repo
  later (say, the team site becomes self-hosted) without
  breaking consumer scripts that pinned to
  `https://x-cmd.com`.

**Cons**:
- One extra hop (GitHub Pages redirect), which is a tiny
  availability risk.
- Harder for a consumer to independently audit — pinning
  to `raw.githubusercontent.com` lets you check the byte
  source against GitHub's HTTPS certificate chain directly;
  pinning to `x-cmd.com` requires you to trust the team's
  certificate setup.

## Pick one or combine

A common pattern in production:

1. **Bootstrap**: `x gpg import` once, in a controlled
   environment (your laptop, a CI runner you control).
2. **Production hosts**: use the `x-cmd.com` redirect for
   `rpm --import`-style package verification, because it's
   short and stable.
3. **CI verification step**: `curl | gpg --import` against
   `raw.githubusercontent.com` directly, every time, so the
   production CI is immune to `x gpg` cache staleness.

All three pull from the same byte source. The choice is
about ergonomics and trust-stacking, not about getting
"the right bytes".

## Offline / air-gapped usage

For environments without direct internet access (finance /
government on-prem networks, classified enclaves), the
three paths collapse into one:

```sh
# On an internet-connected machine:
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  > /media/usb/keyring.asc

# On the air-gapped machine:
gpg --import /media/usb/keyring.asc
```

The transport between the two machines is your choice —
sneakernet, classified-network transfer, whatever your
security regime allows. The keyring file is plain ASCII
text; it can be transferred over any channel that preserves
bytes.

Critical caveat: the air-gapped machine's `index.tsv` should
be transferred in the same bundle, so a consumer can
confirm fingerprint agreement without depending on the air-
gapped network having any external reachability.

## Cross-reference: how `x gpg` consumes this repo

For the implementation of `x gpg` itself — the shell module
that does paths 2 and 3 — see [`x-cmd/x-cmd`'s
`mod/gpg/`](https://github.com/x-cmd/x-cmd/tree/main/mod/gpg).
The module is small (a few hundred lines of POSIX shell) and
auditable end-to-end.

## What to read next

That's the end of the doc series. The remaining documents
in this repo are the technical reference:

- [`README.md`](../README.md) — front-of-page summary
- [`README.cn.md`](../README.cn.md) — Chinese front-of-page
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — maintainer
  pipeline and policy
- [`SKILL.md`](../SKILL.md) — AI-agent recipe
