---
name: 1-what-is-gpg-and-how-do-i-use-it
description: End-user introduction to GPG — what GPG is (program) and what a GPG key is (keypair), the four common end-user scenarios (install a signed package, verify a downloaded artifact manually, cross-check fingerprint against an independent reference, use x gpg shell module), three consumption paths (curl from GitHub, x gpg, x-cmd.com), and pitfalls (CDN cache, look-alike key, handle re-use, short key ID).
type: tutorial
---

# Core Content

core_features:

- GPG = program (GNU Privacy Guard, OpenPGP standard); GPG key = data credential (keypair with public + private halves)
- Fingerprint = 40-char hex = the cryptographic identity of a key (two keys with same fingerprint are, by definition, the same key)
- Four end-user scenarios:
  1. Install a signed package (rpm --import + dnf/apt install)
  2. Verify a downloaded artifact manually (gpg --import + gpg --verify; expect "Good signature")
  3. Cross-check fingerprint against an independent reference (gpg --list-keys --with-colons vs index.tsv fingerprint column)
  4. Use x gpg shell module (one-shot import with caching, fingerprint cross-check)
- Three consumption paths (all serve same bytes):
  1. Direct curl from GitHub (fastest, no deps)
  2. x gpg shell module (caching + cross-check)
  3. https://x-cmd.com (short URL presentation layer built from GitHub at deploy time)
- Five common pitfalls:
  1. CDN cache — proxy returns stale bytes during rotation; pull from GitHub directly
  2. Look-alike key — same UID, different fingerprint; never search keyservers for a known team's key
  3. Handle re-use — rotation takes same handle, fingerprint changes; pin to fingerprint, not handle
  4. Short key ID — 32-bit truncation is reversible; use full 40-char fingerprint
  5. "Public key not found" — import the team's key first

## Key Information

highlights:

- Fingerprint = the cryptographic identity; pin to fingerprint, never to handle, UID, or short key ID
- All three consumption paths serve identical bytes; choice is about ergonomics and trust-stacking
- GPG works for any signer including anonymous OSS contributors; Sigstore requires institutional anchor (GitHub org)
- "Good signature from ..." is the expected output; "BAD signature" means stop and investigate
- Pull from GitHub directly every time, no CDN / reverse proxy / caching service — see LICENSE for trust-anchor rationale
- Cross-checking fingerprint against an independent reference (index.tsv + team site) is the load-bearing step
- For end-user threat models, GitHub's HTTPS endpoint + OS trust store is usually sufficient; for "GitHub serves wrong bytes" threat models, require out-of-band signed messages from the team

## Use Cases

use_cases:

- Installing a signed package from the team's repository
- Verifying a manually-downloaded tarball or release artifact
- Setting up a fresh system to trust the team's keyring
- Auditing what's already in your local keyring
- Understanding what GPG signatures mean when an installer / vendor provides one

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  gpg_docs: <https://gnupg.org/documentation/>
  rpm_verify: <https://rpm.org/user_doc/commands.html>
  x_cmd: <https://x-cmd.com/>

## Summary

Practical GPG tutorial covering what GPG is (program) and what a GPG key is (keypair), the four most common end-user scenarios, three consumption paths, and pitfalls. GPG (GNU Privacy Guard) is the open-source program implementing OpenPGP; a GPG key is the data credential (keypair) with a public half (safe to share) and a private half (must be kept secret). The fingerprint (40-char hex) is the cryptographic identity of a key — two keys with the same fingerprint are by definition the same key. Four scenarios: (1) install a signed package via `rpm --import` + `dnf`/`apt` install — package managers verify GPG signatures automatically and refuse unsigned packages by design; (2) verify a downloaded artifact manually with `gpg --verify` — expect "Good signature from ..." output, stop on "BAD signature"; (3) cross-check fingerprint against an independent reference (index.tsx) after import — load-bearing step; (4) use the `x gpg` shell module which wraps the manual flow with caching and fingerprint cross-check. Three consumption paths serve identical bytes: direct curl from GitHub (fastest, no deps), `x gpg` shell module (caching + cross-check), `https://x-cmd.com` (short URL presentation layer built at deploy time from this GitHub repo). Five common pitfalls: CDN cache (proxy returns stale bytes during rotation; pull directly, no intermediate), look-alike key (same UID, different fingerprint; never search keyservers for a known team's key), handle re-use (rotation takes same handle, fingerprint changes; pin to fingerprint not handle), short key ID (32-bit truncation is reversible; use full 40-char fingerprint), "Public key not found" on install (import the team's key first). For end-user threat models, GitHub's HTTPS endpoint + OS trust store is usually sufficient; for "GitHub serves wrong bytes" threat models, require out-of-band signed messages from the team on transitions.