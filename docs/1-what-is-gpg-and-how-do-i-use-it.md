---
x-title: What is GPG and how do I use it?
x-desc: A practical introduction to GPG for end users — what GPG is, what a GPG key is, the most common scenarios (installing a signed package, verifying a release, importing a public key), the three consumption paths, and pitfalls to avoid.
x-sidebar: What is GPG and how do I use it?
x-keywords: gpg, what is gpg, gpg tutorial, public key, private key, gpg key, gpg signature, gpg verify, gpg import, x gpg, end user
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'What is GPG and how do I use it?'
      inLanguage: 'en'
      about: 'GPG introduction for end users'
---

# What is GPG and how do I use it?

A practical introduction to GPG for end users — the
audience that *consumes* signed software rather than
publishing it. If you want to install the x-cmd RPM, verify
a release tarball, or just understand what those fingerprint
strings mean, this article is for you.

The article assumes Linux/macOS with `gpg` (or `gpg2`)
installed. Most distributions ship it by default; if yours
doesn't, `dnf install gnupg2` / `apt install gnupg2`.

## What is GPG?

**GPG** (GNU Privacy Guard) is an open-source encryption
program that implements the **OpenPGP** standard. It
performs the cryptographic actions: encrypt, decrypt,
generate signatures, verify signatures. It's the
program.

A **GPG key** is the data credential the program
operates on — a keypair comprising a **public key** (safe
to share) and a **private key** (must be kept secret). The
public key verifies signatures made by the matching
private key; the private key is what signs things.

The two are inseparable in practice: GPG without keys has
nothing to encrypt or sign with, and keys without GPG have
nothing that can perform the cryptographic operations.

If you remember one thing from this article, remember
this: **a fingerprint is the cryptographic identity of a
key**. It's the 40-character hex string (e.g.
`4E1C1B9E5C5F0A2D7B3C...`) you see when you run
`gpg --list-keys --fingerprint`. Two keys with the same
fingerprint are, by definition, the same key — there is no
other source of identity. If you pin to the right
fingerprint, you've pinned to the right key.

## Common end-user scenarios

### Scenario 1: install a signed package

Most Linux package managers (`dnf`, `apt`, `zypper`,
`pacman`) verify GPG signatures automatically. The typical
flow:

```sh
# RHEL / Fedora / Rocky: import the team's signing key, then install
sudo rpm --import https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc
sudo dnf install x-cmd

# Debian / Ubuntu: import the team's signing key, then install
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | sudo gpg --dearmor \
  | sudo tee /etc/apt/keyrings/x-cmd.gpg > /dev/null
sudo apt update && sudo apt install x-cmd
```

If you skip the import, the package manager refuses to
install or pops up a warning about an untrusted signature.
That's by design — unsigned packages are a supply-chain
risk.

### Scenario 2: verify a downloaded artifact manually

For tarballs, source releases, or any artifact that isn't
package-manager-managed, verify by hand:

```sh
# Pull the team's keyring and import it
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# Verify the signature
gpg --verify x-cmd-1.2.3.tar.gz.asc x-cmd-1.2.3.tar.gz

# Output should end with "Good signature from ..."
```

If the output says `BAD signature`, **stop and investigate
before trusting the file**. The bytes don't match what the
team signed.

### Scenario 3: confirm the fingerprint matches an independent reference

Importing a key is not enough — you also need to confirm
the fingerprint matches what an independent source says.
The x-cmd repo publishes the team's fingerprint in
[`index.tsv`](../index.tsv); cross-check after import:

```sh
# After `gpg --import`, list the fingerprint
gpg --list-keys --with-colons x-cmd \
  | awk -F: '/^fpr:/{print $10}'
# Compare to the fingerprint in index.tsv
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/index.tsv \
  | awk -F'\t' 'NR > 1 {print $3}'
```

If they match, the bytes you got are the bytes the team
published. If they don't match, **stop** — you may have
downloaded from a different source than you thought.

### Scenario 4: use the `x gpg` shell module

If you have `x` (the x-cmd shell framework) installed, the
`x gpg` module wraps the manual flow with caching and
fingerprint cross-checking:

```sh
# One-shot import of the full keyring (with caching)
x gpg import

# Look up a specific key without importing
x gpg info official

# Verify a signature
x gpg verify x-cmd-1.2.3.tar.gz.asc x-cmd-1.2.3.tar.gz

# List keys already in your local keyring
x gpg ls
```

`x gpg` is in `x-cmd/x-cmd`'s `mod/gpg/` — small enough to
audit end-to-end if you want to verify the tool itself.

## Three ways to fetch the keyring

There are three first-party paths to get the team's
keyring bytes. They all serve the same bytes.

**Path 1 — Direct curl from GitHub.** Fastest, no
dependencies, works everywhere.

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

**Path 2 — `x gpg` shell module.** Adds caching and
fingerprint cross-check.

```sh
x gpg import
```

**Path 3 — `https://x-cmd.com/gpg/`.** A short URL
presentation layer built at deploy time from this GitHub
repo. Convenient for short URLs like
`rpm --import https://x-cmd.com`.

```sh
sudo rpm --import https://x-cmd.com
```

**Do not** route through any third-party CDN, reverse
proxy, or caching service (jsdelivr, gcore, statically,
etc.). A proxy can return bytes from a few minutes ago —
including bytes that predate a key rotation — and your
fingerprint check will still pass against those stale
bytes. See the LICENSE footer for the trust-anchor
rationale.

## Common pitfalls

### "Public key not found" when installing

Your system rpm database doesn't have the team's public
key. Import it (Scenario 1 above).

### The CDN cache

The most common way to import a stale key without knowing it.
Pull from GitHub directly every time, with no intermediate.

### The look-alike key

An attacker constructs a key with a UID that matches the
team's (same name, same email) but a different fingerprint.
You `gpg --search-keys <email>` on a public keyserver,
see a UID that matches, and import — without checking the
fingerprint. **Never search keyservers for a known team's
key.** Fetch from this repo directly. Pin to fingerprint.

### The handle re-use

After a rotation, the new key takes the same handle. Your
tooling pins to "the official key" — but the byte string
changed. **Always pin to fingerprint, not handle.**

### The short key ID

Some recipes pin to the 32-bit short key ID
(`0xDEADBEEF`). Truncating 160 bits to 32 bits is reversible
in practice — an attacker can construct a key whose
truncated short ID matches a target. Don't use the short key
ID for pinning. **Always use the full 40-character
fingerprint.**

## Trusting on first use

If you can't reach an independent reference for source-of-
truth (e.g. you can't reach `x-cmd.com` or compare to
`index.tsv`), you're reduced to "trust the bytes GitHub
serves". For most consumer use cases that's fine — the
GitHub certificate pins `https://your.tld` to the GitHub
certificate authority, which is in your OS trust store.

If your threat model includes "GitHub serves the wrong
bytes", you need out-of-band verification — typically, a
signed message from the team's other channels confirming a new
fingerprint. The team publishes such messages on
transitions.

## Where to read next

- [3. Annual key strategy](./3-annual-key-strategy-explained.md) —
  the team's long-term GPG key rotation strategy
  (community master + annual isolation key).
- [4. GPG UID naming conventions](./4-gpg-uid-naming-conventions.md) —
  ™/® in UID, fingerprint-as-anchor.
- [5. Sigstore, Cosign, and double-signing](./5-sigstore-cosign-and-double-signing.md) —
  the modern alternative for cloud-native supply chains,
  and the case for double-signing both.