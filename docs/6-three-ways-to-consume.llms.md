---
name: 6-three-ways-to-consume
description: Three consumption paths for the keyring — direct curl from GitHub, the x gpg shell module, the GitHub-Pages-via-x-cmd.com redirect — with offline / air-gapped recipe for finance/government environments.
type: how-to
---

# Core Content

core_features:

- Path 1: `curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc | gpg --import` — zero-dependency, pull-at-use, trivially scriptable; best for CI / dev / package builds
- Path 2: `x gpg` shell module — adds caching, fingerprint cross-check, knows about two-key strategy (community vs annual); best for end users with `x` installed
- Path 3: GitHub Pages + `x-cmd.com` subdomain redirect (`https://x-cmd.com/gpg/keyring.asc`) — same bytes, team-domain URL, ergonomic for `rpm --import https://x-cmd.com`
- All three pull from the same byte source (verified by team CI at release time)
- Offline / air-gapped recipe: `curl | gpg --import` on connected machine, transfer via sneakernet, `gpg --import /media/usb/keyring.asc` on isolated machine; bundle `index.tsv` with the keyring for offline fingerprint cross-check
- Common combination — bootstrap with `x gpg import`, production hosts use `x-cmd.com` redirect for `rpm --import`, CI verification steps use `curl` against `raw.githubusercontent.com` directly every time

## Key Information

highlights:

- Choice between paths is about ergonomics and trust-stacking, not "right bytes vs wrong bytes"
- The `x-cmd.com` redirect lets the team move the backing repo later without breaking consumer scripts pinned to the team domain
- Pinning to `x-cmd.com` requires trusting the team's certificate setup; pinning to `raw.githubusercontent.com` lets you check the byte source against GitHub's HTTPS certificate chain directly
- `x gpg` source code lives in `x-cmd/x-cmd` (`mod/gpg/`) — auditable like any other x-cmd module
- For classified enclaves / on-prem finance: bundle `keyring.asc` + `index.tsv` in a single transfer; consumer can fingerprint cross-check without any external network reachability

## Use Cases

use_cases:

- Bootstrapping a CI runner's GPG trust store on first run
- Importing the keyring into a CI container image
- Wiring RPM package verification (`rpm --import`) to a stable, short team URL
- Onboarding a finance / government air-gapped environment (sneakernet the bytes, bundle the manifest)

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  consumer_module: <https://github.com/x-cmd/x-cmd/tree/main/mod/gpg>

## Summary

Three first-party paths to the x-cmd team keyring bytes, each optimized for a different consumer. Path 1 (`curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc | gpg --import`) is zero-dependency, pulls at the moment of use, trivially scriptable — best for CI pipelines, dev workstations, container builds. Path 2 (`x gpg` shell module) adds caching, fingerprint cross-check, and awareness of the two-key strategy — best for end users with `x` installed. Path 3 (`https://x-cmd.com/gpg/keyring.asc` via GitHub Pages + team edge) gives a short, stable team-domain URL — best for `rpm --import https://x-cmd.com` and other places where a memorable URL matters more than direct byte-source auditability. All three serve identical bytes (verified by team CI at release time); the choice is about ergonomics and trust-stacking, not "right bytes vs wrong bytes". Common production combination: bootstrap with `x gpg import` in a controlled environment, production hosts use the `x-cmd.com` redirect for `rpm --import`-style package verification, CI verification steps `curl` against `raw.githubusercontent.com` directly every time so production CI is immune to `x gpg` cache staleness. For offline / air-gapped environments (finance, government, classified enclaves): `curl | gpg --import` on a connected machine, transfer via sneakernet (keyring is plain ASCII text, survives any byte-preserving channel), `gpg --import /media/usb/keyring.asc` on the isolated machine — bundle `index.tsv` in the same transfer so the consumer can fingerprint cross-check without any external network reachability.