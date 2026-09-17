---
name: 7-signing-an-rpm-with-gpg
description: Practical tutorial for RPM package signing with GPG — importing the team's signing private key, configuring rpmsign via ~/.rpmmacros, single + batch signing, verifying, resigning existing RPMs for the LTS repackage workflow, gpg-agent passphrase handling in CI, common pitfalls.
type: how-to
---

# Core Content

core_features:

- Prerequisites: team's signing private key in local gpg keyring, `rpm-build` + `rpm-sign` + `gnupg2` packages installed, built `.rpm` to sign
- `gpg --import /secure/path/to/team-signing-key.private.asc` then `gpg --list-secret-keys --keyid-format long` to confirm
- `~/.rpmmacros` with `%_gpg_name <primary UID>` and `%_gpgbin /usr/bin/gpg2` to unambiguously select the key in CI
- Single-RPM signing: `rpmsign --addsign package.rpm` (appends; preserves existing signatures) vs `--resign` (replaces all)
- Batch signing: `for rpm in /build/RPMS/*/*.rpm; do rpmsign --addsign "$rpm"; done` — handles multi-arch in one loop
- Verification: `rpm -K package.rpm` (basic OK/NOK), `rpm -Kv package.rpm` (verbose, shows signing key fingerprint + UID) — cross-check fingerprint against `index.tsv` column 3
- Repackage/resign lifecycle for LTS: pull historical artifact unchanged, `rpmsign --addsign` to stamp current-year signature without recompiling, publish alongside original-signed version
- Passphrase in CI: either `gpg-agent --daemon --max-cache-ttl 3600` + `gpg-preset-passphrase`, or `rpmsign --passphrase-file <chmod-600-secret>`
- Five common pitfalls: "public key not found" (fix: `rpm --import`), wrong key selected (fix: explicit `%_gpg_name`), multiple signatures (intentional for LTS, use `--resign` to clean), passphrase prompt in CI (agent or `--passphrase-file`), "package is not signed" on resign (rebuild with `%_gpg_name` set during `rpmbuild`)

## Key Information

highlights:

- `--addsign` appends; `--resign` replaces — day-to-day default is `--addsign` (safer)
- LTS repackage workflow is the *intended* use of `--addsign`: original `key-2025` signature stays valid, current `key-2026` signature is appended, both verify on hosts that have either key
- `%_gpg_name` in `~/.rpmmacros` is the unambiguous way to select which key signs in a multi-key keyring
- `gpg-agent` must be started with `--max-cache-ttl` long enough to cover the batch — typical 1-hour CI job is fine with 3600s
- `rpmsign` does not read `~/.bashrc` or pass `GPG_TTY` to gpg-agent — CI must use `--passphrase-file` or pre-load agent via `gpg-preset-passphrase`
- Each release-CI step is independently auditable: which key signed (step 5 + cross-check vs `index.tsv`), what it signed (file list), when (release tag)
- The signing key's fingerprint must match `index.tsv` column 3 — three-way fingerprint check from article 5 applies

## Use Cases

use_cases:

- Day-to-day RPM release: import key, set `%_gpg_name`, batch-sign, verify, publish to yum/dnf repo
- LTS repackage: re-sign an old artifact with the current year's key without recompiling
- Adding the team's public key to a fresh rpm database: `rpm --import <handle>.asc`
- Auditing an existing signed RPM: `rpm -Kv` to confirm signing key + fingerprint

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  rpm_sign_docs: <https://rpm.org/user_doc/building_packages.html>
  rpmsign_man: `man rpmsign`
  rpm_gpg_macros: `man rpmkeys` / `man rpm-macros`

## Summary

A practical tutorial for signing RPM packages with GPG, from importing the team's signing private key through publishing to a yum/dnf repository. The prerequisites are the team's signing private key (held by whoever runs the signing — typically CI secret store or build engineer's offline laptop), `rpm-build` + `rpm-sign` + `gnupg2` packages, and a built `.rpm` to sign. The signing key is imported with `gpg --import /path/to/private.asc` and selected unambiguously in CI via `~/.rpmmacros` with `%_gpg_name <primary UID>`. `rpmsign --addsign package.rpm` appends a signature without removing existing ones (day-to-day default, safer) — `--resign` replaces all (use only when intentionally invalidating prior signatures). Batch signing is a one-line loop over `*.rpm`; multi-arch builds (x86_64 / aarch64 / noarch) all sign independently in the same loop. Verification: `rpm -K package.rpm` for basic OK/NOK, `rpm -Kv` for verbose output showing fingerprint + UID — cross-check the signing key's fingerprint against `index.tsv` column 3 (the three-way fingerprint check from article 5). The LTS repackage workflow: pull historical artifact unchanged from the release archive, `rpmsign --addsign` against current year's key (no recompiling, no inner-bytes changes), publish alongside the original-signed version. Passphrase handling in CI: either `gpg-agent --daemon --max-cache-ttl 3600` + `gpg-preset-passphrase`, or `rpmsign --passphrase-file <chmod-600-secret>` (rpmsign does not read `~/.bashrc` or pass `GPG_TTY` to gpg-agent). Five common pitfalls: "public key not found" (fix `rpm --import`), wrong key selected (fix explicit `%_gpg_name`), multiple signatures (intentional for LTS, use `--resign` to clean), passphrase prompt in CI (agent or `--passphrase-file`), "package is not signed" on resign (rebuild with `%_gpg_name` set during `rpmbuild`). A typical release job sequence: import key from CI secret → set rpmmacros → pre-load passphrase → batch-sign all RPMs → verify → publish to yum repo. Each step independently auditable.