# ADR-0012: Plausible-deniability backup mode

Date: 2026-04-15
Status: Accepted

## Context

BlueWallet pioneered a "plausible-deniability" feature for bitcoin wallets: a user can set a primary password and a separate decoy password. The primary password decrypts the real wallet. The decoy decrypts a pre-seeded dummy wallet with negligible sats. Under physical coercion (wrench attack, border-crossing inspection, jurisdictional pressure), the user can surrender the decoy password and plausibly claim "this is my wallet."

It's not theoretical — bitcoin users traveling through certain jurisdictions, users in authoritarian regions, and victims of targeted crime have all reported pressure scenarios. The feature costs very little to implement and has meaningfully saved people's funds.

Our age-based backup format (ADR-0004) supports multi-recipient encryption natively. Implementing plausible deniability is a small, clean extension — not a cryptographic novelty, just a UX affordance on top of what we're already building.

## Decision

**Ship plausible-deniability mode as a per-user opt-in setting.** Off by default (most users don't need it and it adds onboarding complexity); single-toggle enable in advanced settings.

### How it works

When the user enables plausible-deniability mode, the app maintains **two wallets under one app install**:

- **Real wallet** — the user's actual keys, funds, contacts, chores
- **Decoy wallet** — pre-seeded with a small amount of sats, dummy contacts, dummy transaction history, generated on-device

The user sets two passphrases:
- **Real passphrase** — unlocks the real wallet on app start
- **Decoy passphrase** — unlocks the decoy wallet

The portable `.age` backup file contains both wallets encrypted with their respective passphrase-derived Argon2id recipients, plus optional Nostr key / family / paper-share recipients for the real wallet only:

```
age encrypted file
├── recipient: real passphrase (Argon2id) → unlocks real wallet
├── recipient: decoy passphrase (Argon2id) → unlocks decoy wallet
├── recipient: user's Nostr key → unlocks real wallet
├── recipient: family helper → unlocks real wallet (optional)
└── recipient: paper Shamir shares → unlocks real wallet (optional)
```

age's multi-recipient format handles this natively. Decoy and real wallet ciphertexts are separate but packaged in the same file. An adversary inspecting the file sees opaque stanzas and has no way to tell which recipient decrypts to which wallet.

### Indistinguishability property

The critical property: given the app, one passphrase, and zero other information, an adversary cannot determine:
- Whether a second wallet exists
- Whether the decrypted wallet is the "real" one
- How many total wallets are packaged in the backup file

Implementation requirements to preserve this:

1. **Decoy must look plausible** — real-seeming transaction history, non-trivial (but small) balance, believable contacts. The app generates this automatically from templates the user can customize.
2. **Decoy must transact** — optionally, the decoy wallet can auto-receive small test transactions on a schedule so its history isn't obviously synthetic.
3. **No server-side trace** — the server cannot know whether a user has plausible deniability enabled. Our backup endpoint stores opaque `.age` blobs; we can't tell a one-wallet blob from a two-wallet blob. Event history for real vs decoy wallets goes through the same endpoints with the same signed-event envelopes — the server sees indistinguishable traffic.
4. **No local UI trace when decoy is active** — when the decoy passphrase is entered, the app shows only the decoy wallet. No "switch to real wallet" button, no hint in the settings, no notifications from the real wallet. As far as the app appears to know, the decoy IS the only wallet.
5. **File size obfuscation** — even without plausible-deniability, our `.age` files include padding so that file size doesn't reveal how many recipients or how much data is inside.

### What plausible deniability is NOT

This is explicitly **not** a defense against:
- **Sustained forensic analysis** — if an adversary takes the device, images the disk, and has time to examine everything, they may find evidence of past real-wallet usage (OS-level caches, keyboard autocomplete, app logs). We mitigate by not writing logs, using `zeroize` aggressively, and avoiding OS conveniences like autofill — but we cannot guarantee perfect hygiene.
- **Multi-passphrase extortion** — an adversary who suspects plausible deniability can demand "the other password too." The defense is *plausibility* of the decoy, not cryptographic unlinkability.
- **Legal compulsion in jurisdictions with key-disclosure laws** — perjury risk. Users in such jurisdictions should use this feature advisedly.

We document these limits explicitly in the feature's explainer.

### Threat models this genuinely defends against

1. **Wrench attack** — mugger demands passphrase; user gives decoy; mugger leaves with the decoy wallet thinking they got it all
2. **Border / checkpoint inspection** — user shows decoy wallet on demand, passes through
3. **Jurisdictional pressure** — user under demand to "show your bitcoin" shows the decoy
4. **Shoulder surfing / non-forensic theft** — attacker with brief physical access gets only the decoy

### User flow

**Enable:**
- Settings → Advanced → Plausible deniability → Enable
- App explains what this is + what it doesn't protect against (verbatim from this ADR's "what it is NOT" section)
- User sets real passphrase (already set during onboarding)
- User sets decoy passphrase (must differ significantly — no typos of real)
- App offers decoy wallet template: "Small spender" (~$20 equivalent, casual txs), "Testing wallet" (small amounts, self-transfers), "Gift received" (one-time incoming from a friend, then small spending). User picks or customizes.
- App generates decoy wallet, pre-seeds from a tiny amount the user sends from their real wallet (needs a real on-chain receipt to be indistinguishable)
- Decoy wallet can be configured to self-transact occasionally to age naturally

**Daily use:**
- App start → passphrase prompt (same UI regardless of whether user has decoy enabled)
- Correct real passphrase → real wallet UI
- Correct decoy passphrase → decoy wallet UI (no hint real exists)
- Incorrect passphrase → standard rate-limited retry (no hint that a specific passphrase matched a decoy)

**Backup export:**
- Real mode → export includes both recipients (real passphrase + decoy passphrase + real-wallet-only extras)
- Decoy mode → export includes both recipients too, so the decoy backup file is indistinguishable from a no-deniability export
- User CAN export from decoy mode; backup is still the same file. This is intentional — if the adversary watches the user export a backup, the file appears normal.

### Biometric interaction

If the user has biometric session caching enabled (ADR-0008), biometric-only launch should bring up a choice screen (subtle, not labeled "real/decoy"):
- Two identical-looking entries → user taps the one corresponding to the wallet they want
- Visual differences are minimal; user internalizes which icon is which
- Under coercion user taps the decoy icon → decoy opens

Alternative for simpler threat models: biometric only unlocks the most-recently-used wallet, and decoy is accessed by passphrase entry only.

## Consequences

**Positive:**
- Tangible defense against wrench attacks and checkpoint coercion — situations that happen to real bitcoin users
- Costs ~30 lines of additional code on top of the multi-recipient age format we're already building
- Optional — doesn't complicate the default onboarding flow
- Indistinguishable from single-wallet mode at the file and server level
- Differentiator vs other family wallets; specifically recognizable by bitcoiners from the BlueWallet reference

**Negative:**
- Decoy wallet requires a real on-chain footprint to be plausible — small UX + fee cost at setup
- Users in key-disclosure jurisdictions may face legal risk for using it — documented prominently
- Two passphrases to remember — mitigated by the feature being opt-in
- Decoy template generation adds some complexity to the codebase
- Testing matrix doubles for any feature that interacts with wallets (every flow must be tested under both real and decoy mode)

## Alternatives considered

- **Hidden wallets via BIP-39 passphrase** (Trezor / Coldcard style) — technically equivalent on the seed level, but requires the user to internalize "a different passphrase produces a different wallet." Our approach is the same mechanism wrapped in a UX that names what it's for.
- **VeraCrypt-style hidden volume** — works for file-level deniability but doesn't integrate with wallet operation. We want deniability that extends through every app interaction, not just at-rest storage.
- **Don't ship it** — considered, rejected. The cost is small enough that users who need it get it, and users who don't are unaffected.

## Implementation notes for Q2/Q3

- Q2: multi-recipient age format already required by ADR-0004. Plausible-deniability is 1 additional recipient type (second passphrase).
- Q3: add decoy-wallet template generation (small amounts, synthetic contacts, age-appropriate transaction history). Add "enable plausible deniability" flow in advanced settings.
- Q3/Q4: test the indistinguishability properties — have an external reviewer (ideally someone who doesn't know the decoy template) try to identify which of two exported backup files contains plausible deniability. Should be indistinguishable.
- Documentation explicitly calls out legal risks in key-disclosure jurisdictions

## References

- BlueWallet source — reference implementation of the pattern in a React Native bitcoin wallet
- ADR-0004 — multi-recipient age backup format; this ADR is an extension
- ADR-0008 — key unlock model; biometric behavior under plausible deniability
- ADR-0009 — UX principles; decoy explainer copy must follow the "reassuring, not threatening" guideline while being clear about limits
