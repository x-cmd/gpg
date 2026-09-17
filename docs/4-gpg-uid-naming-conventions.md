---
x-title: GPG UID naming conventions
x-desc: A design-decision analysis on whether to include trademark symbols (™/®) in GPG key User IDs — four dimensions (cryptographic safety, legal defense, terminal encoding, open-source community culture), comparison table of four conventions, and a recommended default.
x-sidebar: GPG UID naming conventions
x-keywords: gpg, uid, user id, trademark, tm, ®, package signing, open-source culture, terminal encoding, anti-phishing, brand defense, naming
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GPG UID naming conventions'
      inLanguage: 'en'
      about: 'Whether to include trademark symbols in GPG key User IDs'
---

# GPG UID naming conventions

A design decision teams eventually face when publishing a
package-signing GPG key: whether to include a trademark
symbol (`™` or `®`) in the key's User ID (UID). The UID
is the human-readable label that appears when consumers run
`gpg --list-keys` or any UI that surfaces key metadata. It
is *not* part of the cryptographic material — fingerprint
and key bits stay unchanged regardless — but it does
affect brand recognition, terminal rendering, and (in
adversarial situations) legal clarity around impersonation.

This walks through four dimensions teams typically weigh
when picking a UID convention: cryptographic safety, legal
defense, terminal encoding compatibility, and open-source
community culture reception. Closes with a comparison
table and a recommended default.

## What's a UID, exactly?

A GPG key's UID is the `Name (comment) <email>` label
stored in the key's user-id packet. It's editable by the
key holder and doesn't participate in any cryptographic
operation — verification uses the fingerprint, not the UID
text. UIDs are display metadata.

Typical UID conventions look like:

```
Acme Corporation (Official Package Signing Key) <packages@acme.example>
```

The first half (`Acme Corporation`) is the brand mark.
The parenthetical (`Official Package Signing Key`) is the
purpose label. The angle-bracket part (`<packages@acme.example>`)
is the contact email — though for package-signing keys it's
often a role account, not a personal one.

## Dimension 1 — cryptographic safety

**Adding the symbol changes nothing for security.**

`dnf` / `rpm` / `gpg --verify` validate signatures against
the mathematical relationship between the public and
private key, not against the UID string. Whether the UID
reads `Acme`, `Acme®`, `Acme™`, or `Acme (TM)`, the bytes
that get signed and verified are unchanged. The symbol is
metadata; it has no bearing on signature validity, key
fingerprint, or trust establishment.

The actual defense against malicious substitution is the
combination of:

- private key kept off publicly-reachable infrastructure
- public key delivered over HTTPS from a domain the team
  controls
- fingerprint pinned by consumers against an independent
  reference

UID text is irrelevant to any of that.

## Dimension 2 — legal defense / anti-phishing

The legal argument for adding `™`/`®` is roughly: if
they sue an impostor for trademark infringement, the
presence of the mark on the team's *own* keys makes it
harder for the impostor to claim "we didn't know it was a
brand" or "we just used the plain word". The argument has
force in some jurisdictions and is weak in others.

**Pro (with the symbol):**

- An impostor who copies the *plain* `Acme` can plausibly
  claim "this is a generic technical term, not a brand".
  Adding `™` or `®` makes the brand-claim obvious and
  reduces the impostor's room to argue.
- If the impostor copies the *symbolic* form too
  (`Acme™`), that's clear evidence of intentional
  impersonation — and the explicit mark may help courts
  find willful infringement, which can increase damages.

**Con (without the symbol):**

- Trademark protection typically rests on the registration
  certificate, not on whether the mark appears in GPG
  metadata. Most jurisdictions don't require the `™`/`®`
  character to be present in every public artifact to
  enforce the mark.
- The legal argument assumes an active lawsuit; for the
  99% of cases where no lawsuit is filed, the symbol
  provides zero practical protection.

**Verdict:** the symbol helps *marginally* in adversarial
legal scenarios; the protection rests on the trademark
registration and the official delivery channel regardless.
Don't rely on UID text as your primary anti-phishing
defense.

## Dimension 3 — terminal encoding / display

**`®` is risky; `™` is usually fine; plain text is
universal.**

`®` (U+00AE) and `™` (U+2122) are valid Unicode, but their
behavior on minimal / non-UTF-8 terminals varies:

- On a modern UTF-8 locale (`LANG=en_US.UTF-8` or
  similar), both render correctly in any decent terminal.
- On minimal / legacy setups (older PuTTY / SecureCRT
  configurations, certain embedded distros, alpine base
  images, CI logs that strip UTF-8), `®` has a much
  higher chance of rendering as a replacement glyph
  (`Acme?`, `Acme\xAE`, etc.) than `™`. This is because
  `®` lives in the Latin-1 supplement block (0x00A0–0x00FF)
  and some legacy code pages treat bytes 0xA0–0xFF as
  control characters; `™` lives in a Unicode-only block
  and tends to render consistently as long as the locale
  is Unicode.
- `(TM)` (the literal ASCII seven characters) renders
  identically everywhere — it's plain text.

Display corruption doesn't break verification
(`gpg --verify` works on bytes, not glyphs), but it
materially hurts the *professional appearance* of the UID
when a user lists the key on a misconfigured terminal —
the very moment the brand impression is being formed.

**Verdict:** if any symbol is used, `(TM)` (plain ASCII)
is the safest. `®` and even `™` are technically fine on
modern systems but carry a real risk of looking broken in
the long tail of legacy / minimal environments.

## Dimension 4 — open-source community culture

This dimension is real, even if it's softer than the
others.

GPG key UIDs in open-source infrastructure are typically
read as *identity labels*, not *marketing surfaces*. The
historical pattern, observable across Red Hat, SUSE,
Canonical, Debian, and similar: clean, descriptive UIDs
without `™`/`®` decoration. For example:

- Red Hat: `Red Hat, Inc. (release key 2) <security@redhat.com>`
- SUSE: `SUSE Linux Enterprise Server 12 ...`
- Canonical (Ubuntu): `Ubuntu Archive Automatic Signing Key <ftpmaster@ubuntu.com>`

These organizations hold valid trademark registrations
but keep their GPG UIDs purely descriptive. They do the
brand-defense work in the trademark notice at the bottom
of the web page and in the official domain — not in the
GPG metadata.

The reason this matters in practice: when a user runs
`sudo rpm --import` and the terminal pops up a key with a
glaring `®` or `™`, the impression *is* different from a
clean label. Some users — especially the long-tail of
operators who work in low-trust environments and have been
conditioned to be suspicious of anything that looks like
aggressive self-branding — find the symbol-laden form
off-putting. It's not a major effect, but it's a real one.

## Comparison of the four conventions

| Convention                              | UID example                                                | Compat | Legal | Community | Verdict |
| ---                                     | ---                                                        | ---    | ---   | ---       | ---   |
| **1. Symbolic (`®`)**                   | `Acme® (Official Package Signing Key) <…>`                 | ⚠️    | ✓✓    | –         | Avoid — encoding risk outweighs legal gain |
| **2. Symbolic (`™`)**                   | `Acme™ (Official Package Signing Key) <…>`                 | ⚠️    | ✓✓    | –         | Acceptable but unnecessary |
| **3. Plain-text `(TM)`**                | `Acme (TM) (Official Package Signing Key) <…>`             | ✓✓    | ✓     | ≈         | Acceptable if legal gain is hard requirement |
| **4. Plain text (recommended default)**  | `Acme (Official Package Signing Key) <packages@acme.example>` | ✓✓    | ≈     | ✓✓        | Recommended |

Notes on the columns:

- **Compat** = rendering reliability across minimal /
  non-UTF-8 terminals. `✓✓` = universally fine; `⚠️` = real
  risk on the legacy / minimal long tail.
- **Legal** = marginal value as anti-phishing evidence in a
  potential lawsuit. `≈` doesn't mean zero (trademark
  protection rests on the registration and the official
  channel); it means the UID text isn't doing meaningful
  legal work on its own.
- **Community** = subjective reception by the open-source
  community on first impression.

## Recommended default

Adopt convention 4 — clean, descriptive UID, no symbol:

```
Acme (Official Package Signing Key) <packages@acme.example>
```

Move brand-defense to the right places:

- **`TRADEMARK.md`** in the project repository (and on
  the team site) — explicit "Acme is a trademark of …"
  notice with reference to the registration certificate.
  This is the standard location the open-source community
  expects trademark notices.
- **Team site footer** — same notice, visible on every page
  where the key is referenced.
- **Fingerprint pinning** — train consumers to verify the
  fingerprint against an independent reference, not to
  recognize the UID. The UID is a label; the fingerprint
  is the trust anchor.

This combination matches what Red Hat, SUSE, Canonical,
and Debian do, and it puts the brand-defense work where it
actually has legal and operational effect.

## When to deviate

Teams with a *specific* adversarial trademark problem — a
named competitor publishing look-alike keys in the wild,
active phishing campaigns against the team's brand — may
reasonably pick convention 3 (plain-text `(TM)`). The
encoding risk is zero and the marginal legal clarity is
real, even if small. Conventions 1 and 2 are hard to
recommend — the encoding risk is real and the legal gain
isn't large enough to offset it.

## Where to read next

- [1. What is GPG and how do I use it?](./1-what-is-gpg-and-how-do-i-use-it.md) —
  end-user perspective; fingerprint as the trust anchor.
- [3. Annual key strategy](./3-annual-key-strategy-explained.md) —
  long-term key rotation trade-offs.
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — how a key gets
  published once you've chosen a UID convention.