---
x-title: Signing an RPM with GPG
x-desc: How to add a GPG signature to an RPM package — importing the signing private key, configuring rpmsign, signing one or many packages, verifying the result, and resigning existing RPMs (the repackage / resign lifecycle used for paid LTS customers).
x-sidebar: Signing an RPM with GPG
x-keywords: rpm, rpmsign, gpg, package signing, createrepo, repackage, resign, lts, batch signing, gpg-agent
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Signing an RPM with GPG'
      inLanguage: 'en'
      about: 'RPM package signing workflow'
---

# Signing an RPM with GPG

A practical tutorial for the team's release workflow: how to
take a built `.rpm` file, stamp it with the team's GPG
signature, and publish it to a yum/dnf repository. The same
recipe works for the repackage / resign lifecycle that paid
LTS customers rely on (re-signing an old artifact with the
current year's key — see
[4. Annual key strategy explained](./4-annual-key-strategy-explained.md)).

## Prerequisites

You need three things before you can sign an RPM:

1. **The team's signing private key** in your local GPG
   keyring. The matching public key is in
   [`keyring/<handle>.asc`](../keyring/) of this repo; the
   private half is held by whoever is doing the signing
   (typically the release CI's secure secret store, or a
   build engineer's offline laptop).
2. **`rpm-build` and `rpm-sign` packages** installed. On
   RHEL-family distros:

   ```sh
   dnf install rpm-build rpm-sign gnupg2
   ```
3. **A built `.rpm` file** to sign. This article assumes
   you've already produced the package and just need to
   stamp it.

## Importing the signing key

The private key comes from wherever the team stores secrets
— typically exported once and stored encrypted, or pulled
from a CI secret. The import is the same as for a public key
(`gpg --import`), but the file holds the private half:

```sh
# From an exported private-key bundle
gpg --import /secure/path/to/team-signing-key.private.asc

# Confirm the import
gpg --list-secret-keys --keyid-format long
```

The output should list the team's key with both `pub` and
`sec` rows. Note the long key ID (the 16-hex-char form after
the `rsa4096/` algorithm label) — you'll pass it to
`rpmsign`.

## Configuring rpmsign

`rpmsign` reads `~/.rpmmacros` (or `%_topdir/.rpmmacros`) for
default-key and signing settings. For CI / batch work, set
the key explicitly in the macro file so you don't rely on
`gpg-agent`'s "default key" guess:

```text
# ~/.rpmmacros
%_gpg_name  Li Junhao (x-cmd signing key) <l@x-cmd.com>
%_gpgbin    /usr/bin/gpg2
```

If you have multiple keys in your keyring, `%_gpg_name`
unambiguously identifies which one to use. Format is the
primary UID of the key.

For passphrase handling in CI, see
[Passphrase and gpg-agent](#passphrase-and-gpg-agent) below.

## Signing a single RPM

```sh
rpmsign --addsign /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

`--addsign` *adds* a signature without removing existing
ones — useful for the LTS repackage workflow where you want
the artifact to carry signatures from both this year's key
and last year's. To replace all existing signatures, use
`--resign` instead:

```sh
rpmsign --resign /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

For day-to-day releases `--addsign` is the safer default;
`--resign` is only appropriate when you intentionally want
to invalidate every prior signature on the file.

## Signing multiple RPMs (batch / CI)

For a directory of built packages:

```sh
# Loop over every .rpm in the build output dir
for rpm in /path/to/build/RPMS/*/*.rpm; do
  echo "signing $rpm"
  rpmsign --addsign "$rpm"
done
```

For multi-architecture builds (`.x86_64.rpm`, `.aarch64.rpm`,
`.noarch.rpm`), the same loop covers all of them — each RPM
is signed independently.

## Verifying the signature

```sh
rpm -K /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

Output:

```text
/path/to/x-cmd-1.2.3-1.x86_64.rpm: rsa4096 (sig1) OK (Full RSA)
```

`OK` means the signature verifies against a public key in
the system's rpm key database. For verbose output (showing
the signing key's fingerprint and UID):

```sh
rpm -Kv /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

Cross-check the signing key's fingerprint against
[`index.tsv`](../index.tsv) column 3 — that's the
"three-way fingerprint comparison" workflow from
[5. Verifying a key](./5-verifying-a-key.md).

## Repackage / resign lifecycle (LTS workflow)

When a paying LTS subscriber wants a 2025-era package re-signed
with `key-2026` (or whichever is current), the workflow is:

1. Pull the historical package from the team's release
   archive (the file is unchanged from when it was
   originally built and signed by `key-2025`).
2. `rpmsign --addsign` against the current year's key:

   ```sh
   rpmsign --addsign /path/to/x-cmd-1.2.0-1.x86_64.rpm
   ```

   The bytes *inside* the package are unchanged. Only the
   signature header is augmented.
3. Re-publish the artifact to the release archive alongside
   the original-signed version. Both versions remain
   available; consumers pick which one to import based on
   which keys they have.

This is exactly the repackage / resign lifecycle described
in
[4. Annual key strategy explained](./4-annual-key-strategy-explained.md#repackage--resign-lifecycle-lts-customers)
— `rpmsign --addsign` against an existing release asset,
without recompiling or modifying the upstream binary.

## Passphrase and gpg-agent

`rpmsign` invokes `gpg` under the hood, which will prompt
for the signing key's passphrase on every invocation. For
human-driven signing, that's fine — type the passphrase
once. For CI / batch, you need either:

- **`gpg-agent` with a pre-loaded cache.** Start `gpg-agent`
  in your CI job before invoking `rpmsign`; set
  `default-cache-ttl` long enough to cover the batch; preset
  the passphrase via `gpg-preset-passphrase`.
- **`--passphrase-file <path>`** passed to `rpmsign` (which
  forwards to `gpg`). Point at a file containing the
  passphrase. The file should be `chmod 600` and live in
  the CI's secret store.

For one-off signing on a developer laptop, the simplest
recipe is:

```sh
# Start gpg-agent, enter passphrase once, run rpmsign
gpg-agent --daemon --max-cache-ttl 3600
rpmsign --addsign package.rpm
gpgconf --kill gpg-agent
```

The agent caches the passphrase for an hour, signs all
RPMs in that window, then is killed. For multi-hour CI
builds, raise `--max-cache-ttl` accordingly.

## Common pitfalls

### "Public key not found" when running `rpm -K`

The system rpm database doesn't have the team's public key.
Import it:

```sh
# Pull from this repo
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | sudo rpm --import -
```

### Wrong key selected

`rpmsign` signs with whichever key matches `%_gpg_name` in
`~/.rpmmacros`. If your keyring has multiple keys and the
macro file is missing or wrong, you may sign with the wrong
one. Always set `%_gpg_name` explicitly in CI; the resulting
RPM's signature header records exactly which key signed it.

### Multiple signatures

`--addsign` adds a signature *alongside* existing ones.
After running it twice with different signing keys, the RPM
has two valid signatures — both verify, neither is invalid.
This is the desired behavior for the LTS repackage workflow
(both old and new keys can verify). For a clean release
where you want only the current key, use `--resign` instead.

### Passphrase prompt in CI

`rpmsign` doesn't read `~/.bashrc` or pass `GPG_TTY` to
`gpg-agent`. For headless CI, either use
`--passphrase-file` (with `chmod 600`), or pre-load
`gpg-agent` with `gpg-preset-passphrase` and a configured
`--max-cache-ttl`.

### Re-signing fails with "package is not signed"

`--addsign` requires the package to already have at least
one signature (the original from `rpmbuild`). If `rpmbuild`
was run without signing enabled, the package is unsigned;
`rpmsign --addsign` will refuse. Either:
- Rebuild with `%_gpg_name` set during `rpmbuild`, which
  embeds the signature as part of the original build (the
  cleaner path), or
- Use `rpmsign --addsign` on a package that was already
  signed at build time.

## Putting it together (release CI)

A typical release job:

```sh
# 1. Import the team's signing key from CI secret store
echo "$TEAM_SIGNING_KEY_PRIVATE" | gpg --import --batch

# 2. Set up rpmmacros
cat > ~/.rpmmacros <<EOF
%_gpg_name  Li Junhao (x-cmd signing key) <l@x-cmd.com>
EOF

# 3. Pre-load passphrase into gpg-agent
echo "$TEAM_GPG_PASSPHRASE" \
  | gpg-preset-passphrase --preset $(gpg --list-secret-keys --with-colons \
                                       | awk -F: '/^sec/{print $5}')

# 4. Sign every RPM in the build output
for rpm in /build/RPMS/*/*.rpm; do
  rpmsign --addsign "$rpm"
done

# 5. Verify
rpm -Kv /build/RPMS/*/*.rpm

# 6. Publish to yum repo
createrepo --update /var/www/repo/x-cmd/
```

Each step is independently auditable: which key signed (step
5 + cross-check vs `index.tsv`), what it signed (the file
list), when (the release tag).

## What to read next

- [5. Verifying a key](./5-verifying-a-key.md) — the
  three-step fetch → import → compare recipe; run it after
  signing to confirm the right key landed.
- [4. Annual key strategy explained](./4-annual-key-strategy-explained.md#repackage--resign-lifecycle-lts-customers) —
  how this signing workflow plugs into the annual rotation
  and repackage / resign lifecycle.
- [2. How the keyring is published](./2-how-the-keyring-is-published.md) —
  the team-internal pipeline this article is the
  signing-step for.

## FAQ

A subset of the central
[FAQ in article 0](./0-x-cmd-gpg-overview.md#faq--software-distribution--code-signing-cryptography)
most relevant to this article. The full 8-question set lives
in article 0; the answers are reproduced there in a
project-agnostic, industry-wide form.

### Q1: GPG software vs. GPG Key — what's the technical relationship?

This article exercises both: the **gpg** program (which
`rpmsign` shells out to for the cryptographic operations) and
the **keypair** (which holds the cryptographic material the
program operates on). Importing the team's signing private
key into your local GPG keyring is the data-side prerequisite
for `rpmsign` to find a key to sign with. See article 0 for
the full answer.

### Q3: Why don't publishers usually distribute unsigned raw packages?

Because `dnf` / `yum` / `rpm` will refuse to install
unsigned RPMs by default, and unsigned distribution has no
cryptographic tamper protection during transit (MITM and
package poisoning become trivial). The signing workflow in
this article is the standard mitigation: produce the
package, stamp it with `rpmsign --addsign` before
publishing, and the consumer's `rpm -K` will return `OK`.
The `--addsign` operation is what binds the cryptographic
identity (the team's keypair) to the bytes (the RPM file).
See article 0 for the full answer.