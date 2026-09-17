---
name: 8-trademark-in-gpg-key-uids
description: Four-dimension analysis (cryptographic safety / legal defense / terminal encoding / open-source culture) of whether to include ™/® in GPG key User IDs. Comparison table of four UID conventions. Recommended default: plain text, no symbol — matches Red Hat, SUSE, Canonical, Debian practice.
type: design-decision
---

# Core Content

core_features:

- Decision: include or omit trademark symbols (™ / ®) from GPG key User IDs
- Four dimensions analyzed:
  1. Cryptographic safety — symbol is metadata, no impact on signature validity, fingerprint, or trust establishment
  2. Legal / anti-phishing — marginal help in adversarial lawsuits; trademark protection rests on registration + official domain, not UID text
  3. Terminal encoding — ® risky on legacy / minimal terminals, ™ usually fine on Unicode locales, `(TM)` plain ASCII universally fine
  4. Open-source culture — Red Hat / SUSE / Canonical / Debian all use clean descriptive UIDs without ™/®; rely on TRADEMARK.md and domain for brand defense
- Four UID conventions compared:
  1. `X-CMD® (Official Package Signing Key) <…>` — encoding risk outweighs legal gain, avoid
  2. `X-CMD™ (Official Package Signing Key) <…>` — acceptable but unnecessary
  3. `X-CMD (TM) (Official Package Signing Key) <…>` — zero encoding risk, small legal clarity; pick if anti-phishing is hard requirement
  4. `X-CMD (Official Package Signing Key) <packages@x-cmd.com>` — recommended default; perfect compat, marginal legal (relies on registration + domain), excellent community reception
- Brand-defense alternatives: TRADEMARK.md in repo + team site footer (per industry standard) + fingerprint pinning training for consumers

## Key Information

highlights:

- UID text is not part of cryptographic material — fingerprint stays the same regardless
- Real anti-phishing defense is fingerprint pinning + HTTPS delivery from a team-controlled domain + private key kept off public infrastructure
- ® lives in Latin-1 supplement block (U+00A0–0x00FF), renders as `?` / `\xAE` on legacy code pages
- ™ lives in Unicode-only block (U+2122), renders fine on Unicode locales
- `(TM)` (7 ASCII chars) renders identically everywhere
- Red Hat, SUSE, Canonical, Debian — all clean descriptive UIDs, no ™/®
- Trademark protection typically rests on the registration certificate, not on whether the mark is in GPG metadata

## Use Cases

use_cases:

- Picking a UID convention for a new package-signing key
- Deciding whether to include trademark symbols in any open-source identity artifact
- Justifying the "clean UID" choice to a brand-defense / legal team
- Documenting the team's UID convention for future maintainers

## Related Resources

official:
  website: <https://x-cmd.com/mod/gpg>
  repo: <https://github.com/x-cmd/gpg>
related:
  rfc4880: <https://datatracker.ietf.org/doc/html/rfc4880>
  rpm_sign: <https://rpm.org/user_doc/building_packages.html>

## Summary

A design-decision article on whether to include trademark symbols (™ or ®) in GPG key User IDs (UIDs). The UID is the human-readable label that appears when consumers run `gpg --list-keys` — it is *not* part of the cryptographic material, so the symbol has no effect on signature validity, fingerprint, or trust establishment. Four dimensions analyzed: (1) cryptographic safety — symbol is metadata, no impact; (2) legal / anti-phishing — the symbol helps *marginally* in adversarial lawsuits by making the brand-claim obvious, but trademark protection rests on the registration certificate and official delivery channel, not UID text; (3) terminal encoding — `®` (U+00AE, in Latin-1 supplement) has real risk of rendering as replacement glyphs on legacy / minimal terminals, `™` (U+2122, Unicode-only) usually fine on Unicode locales, plain ASCII `(TM)` is universally reliable; (4) open-source community culture — Red Hat, SUSE, Canonical, Debian all use clean descriptive UIDs without ™/®, doing brand-defense work in TRADEMARK.md and the official domain rather than in GPG metadata. Four UID conventions compared: symbolic `®` (avoid — encoding risk outweighs legal gain), symbolic `™` (acceptable but unnecessary), plain-text `(TM)` (acceptable if legal gain is hard requirement), plain text no symbol (recommended default — perfect compat, marginal legal, excellent community reception). Recommended UID: `X-CMD (Official Package Signing Key) <packages@x-cmd.com>`. Brand-defense work moves to TRADEMARK.md in repo + team site footer + fingerprint-pinning training for consumers (the actual anti-phishing defense). When to deviate: teams with specific adversarial trademark problems (named competitor publishing look-alike keys, active phishing) may pick convention 3. Conventions 1 and 2 are hard to recommend — encoding risk is real and legal gain doesn't offset it.