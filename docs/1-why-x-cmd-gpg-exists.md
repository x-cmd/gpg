---
x-title: Why x-cmd/gpg exists
x-desc: A primer for newcomers — what a GPG trust anchor is, the supply-chain problem this repo solves, and why x-cmd maintains its own keyring instead of using keys.openpgp.org or similar.
x-sidebar: Why x-cmd/gpg exists
x-keywords: gpg primer, trust anchor, supply chain, signature verification, why separate repo, x-cmd
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Why x-cmd/gpg exists'
      inLanguage: 'en'
      about: 'x-cmd/gpg purpose and rationale'
---

# Why x-cmd/gpg exists

A primer for readers who are new to GPG, new to supply-chain
keying, or new to x-cmd. By the end of this article you should
be able to explain to a colleague (or a procurement officer)
what a GPG trust anchor is, why x-cmd needs its own keyring
repo, and what would go wrong if you used a generic public
keyserver instead.

## What "trust anchor" means in practice

A GPG public key is a string of bytes that, when you import
it into GnuPG, makes your local installation willing to
**verify** signatures produced by the matching private key.
Once imported and marked trusted, every byte you download
that carries a valid signature from that private key passes
your local `gpg --verify` check.

That's the whole game. There is no certificate authority, no
revocation server, no central registry of "what's a valid
x-cmd key" — your decision to import a key, and your belief
that the bytes you imported are the bytes the team
published, is the entire trust model.

So when we say "trust anchor" we mean: the specific bytes of
a public key, exactly as the team published them, fetched from
a channel you have independently verified to be the team's
own. Every other consideration — fingerprint format, key
expiry, web-of-trust signatures — is downstream of "do I
have the right bytes".

## The supply-chain problem this repo solves

x-cmd ships signed artifacts: RPMs, DEBs, container images,
release tarballs. Every signed artifact contains a signature
header that says, in effect, "the holder of *this* private
key signed this artifact". The consumer's job is to verify
that signature against a public key they trust.

For that to work, two things have to be true:

1. The signature on the artifact was produced by *x-cmd's*
   private key (not a look-alike impostor's).
2. The public key the consumer is verifying against *is*
   x-cmd's public key (not a substituted impostor copy).

Condition 1 is what GPG's mathematics gives you — a signature
binds cryptographically to the specific private key. Condition
2 is what *this repo* gives you — a single, canonical,
team-controlled location where the public key bytes are
published, with the team's own commit history and signed
releases as the audit trail.

If you can't satisfy condition 2, the whole chain collapses —
you've reduced signature verification to "trust that whoever
sent you the key isn't lying about who they are". That's
not a security property; that's a hope.

## Why a separate repo

The x-cmd codebase lives in `x-cmd/x-cmd`, a large monorepo
that gets cloned by every consumer of `x gpg`, every package
build, and every release pipeline. Putting the team's GPG
keys in `x-cmd/x-cmd` would create three problems:

1. **Circular signing.** `x-cmd/x-cmd`'s release workflow
   fetches signing keys to sign release artifacts. If those
   keys live in the repo that the keys themselves sign, you've
   reduced your key distribution to "trust the unsigned
   bootstrap chain" — a real category of supply-chain
   weakness.
2. **Repo size and churn.** A monorepo changes thousands of
   times per week. Pulling the keys from it forces every
   consumer to also pull — and validate against — that churn,
   which means each consumer's verification depends on
   trusting every commit in the monorepo's history. The
   trust surface shrinks.

   The single-purpose `x-cmd/gpg` repo changes only when a
   key is added or rotated — a handful of commits per year.
3. **Independent review surface.** Rotations are infrequent
   but security-critical. A small, focused repo with its own
   diff history makes it trivial for a second team member to
   review "is this really the new key?" without scanning
   thousands of unrelated changes.

So the decision is: keep the trust anchor in its own repo,
where the bytes are stable, the review surface is tiny, and
the only path to a fake key is "convince the team to commit
it" — which the team won't do without an internal review.

## Why not keys.openpgp.org / keyserver.ubuntu.com

Public keyservers are a great general-purpose tool: if you
don't know where someone's key is, search the keyservers. For
a *team* distributing its own keys to its own consumers,
they're the wrong tool, for four reasons:

1. **Anyone can upload.** A keys.openpgp.org upload is
   self-attested — any user with the bytes can put them on
   the server. The server doesn't validate that the upload
   came from the team.
2. **Substitution attacks.** Once uploaded, an attacker can
   upload a *look-alike* key with a near-identical UID but
   different fingerprint. Consumers who `gpg --search-keys`
   by name or email may pick the wrong one.
3. **No first-party audit trail.** The keyservers don't
   preserve *who* uploaded *when* in a way that maps back to
   the team's commit history. This repo's `git log` does.
4. **LICENSE mismatch.** Public keyservers are general-purpose
   infrastructure; they re-serve bytes from any uploader. The
   LICENSE in this repo explicitly forbids third-party
   redistribution. Using a keyserver as a distribution channel
   violates the LICENSE.

For the 99% of users who already know they want the
x-cmd team key, the answer is: fetch it from
https://github.com/x-cmd/gpg (or via the GitHub-Pages-via-
x-cmd.com redirect). Don't search keyservers.

## What this repo is NOT

- **Not** a general-purpose keyserver. We don't accept
  uploads; we don't mirror keys.openpgp.org; we don't fetch
  keys from keyserver.ubuntu.com.
- **Not** a revocation authority. Compromised-key reports go to
  the team's private channels, not this repo.
- **Not** a signature archive. Signed commits and signed
  release artifacts live in the upstream repos that produced
  them. This repo only publishes the *public half* of the
  keypair.

## What to read next

- [2. How the keyring is published](./2-how-the-keyring-is-published.md) — the
  team-internal pipeline that gets a key from a fresh
  `gpg --export` to a row in `index.tsv`.
- [3. Reading the key catalog](./3-reading-the-key-catalog.md) —
  every column of `index.tsv` and the fingerprint math
  underneath.
- [4. Annual key strategy explained](./4-annual-key-strategy-explained.md) —
  the long-form version of FAQ Q4–Q8, including the
  "no expiry / annual rotation" decision.