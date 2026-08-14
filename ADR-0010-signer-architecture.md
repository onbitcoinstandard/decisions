# ADR-0010: Split-mode signer architecture with SeedSigner feature parity

Date: 2026-04-15 (amended 2026-04-16)
Status: Accepted — stateless + airplane-gated model per 2026-04-16 amendment below

> **Deployment revised by ADR-0014:** ships as installed PWA at `signer.onbitcoinstandard.com` with offline-first service worker.

> **Stateless + airplane-gated amendment (2026-04-16):** the signer **does not persist keys on device**. Each signing session is stateless — user enters seed fresh each time, seed is held in memory only for the signing, zeroed on exit. The PWA **refuses to function unless the device is in airplane mode** (network state polled continuously; page locks if network reappears). This raises the strength from "~85% of Tauri signer" to "functionally equivalent air-gapped hardware signer" for the threat model we care about. Rationale + flow in § Stateless + airplane-gated (amendment) below.

## Context

ADR-0008 established that consumer Secure Elements can't hold secp256k1 keys, so "hardware-backed" Bitcoin custody in software is an approximation, not a true property. For daily-wallet amounts on a phone this is acceptable. For savings and inheritance wallets holding larger value, users deserve better — and buying a dedicated hardware wallet (Coldcard, BitBox, Jade) is a $100+ barrier we don't want to force.

The answer: emulate hardware-wallet security properties in software using a split-device architecture. The same codebase, built with different feature flags, produces a wallet app (online) and a signer app (offline). Together on two devices they approximate a SeedSigner setup — air-gapped, PSBT-QR transport, no network path to the key.

## Decision

### One codebase, two compile-time feature flags

```
cargo build --features wallet       # full wallet build
cargo build --features signer       # signer-only build
```

The `signer` feature flag:
- Compiles OUT all network code (no HTTP client, no WebSocket, no Nostr relay, no ASP coordinator, no push notifications, no chore system, no family graph)
- Compiles IN key management, age encryption, signed-event signing, PSBT parsing + signing, QR scan + display, SeedQR + SLIP-39 card handling
- Result: ~40% smaller binary, genuinely network-incapable (not just "we promise not to call the network — we removed the code that could")

Single source of truth = no drift between modes on critical crypto code. Rust's feature-flag system makes this clean. Chosen over runtime-mode-toggle because compile-time removal is genuinely stronger than "we're disabled right now."

### Feature parity with SeedSigner

The `signer` build matches SeedSigner's feature set, with better UX afforded by a touchscreen + full CPU.

**Seed entry:**
- BIP-39 word entry with full autocomplete keyboard (vs SeedSigner's 4-letter joystick typing)
- SLIP-39 share entry (SeedSigner doesn't support this)
- SeedQR scan (standard)
- CompactSeedQR scan (standard)
- Dice entropy (99 / 50 / 77 rolls with visual progress counter)
- Camera image entropy mixing (optional)
- BIP-39 passphrase entry

**Seed export / backup:**
- SeedQR display
- CompactSeedQR display
- BIP-39 words display (with anti-screenshot hints where platform supports)
- SLIP-39 shares display
- Printable paper templates (PDF, SeedPlate-compatible layouts)

**Cards:**
- Printed SeedQR cards — produced by this signer, scannable by any SeedSigner-compatible device (standard format)
- SLIP-39 printed mnemonic cards
- SD card seed files — on platforms with removable storage (Android, Linux laptop). SeedSigner's standard air-gap transport.
- NFC cards (Tapsigner / Satochip style) deferred — post-PoC hardware-signer integration

**PSBT signing:**
- PSBT via QR — animated UR or BBQR for PSBTs that exceed single-QR capacity
- PSBT via SD card file
- Canonical transaction review screen — amount, fee, all outputs, unusual-output warnings, change-output detection
- Multisig coordination — sign as one-of-N, pass PSBT along to next cosigner

**Coordinator interoperability:**
- Xpub export (QR or file, Sparrow / Specter / Nunchuk / Electrum JSON formats)
- Wallet descriptor import (scan QR from coordinator)
- Output descriptors (BIP-380) native throughout
- Address verification — derive from xpub, confirm displayed address is ours

**Capabilities SeedSigner lacks:**
- SLIP-39 throughout
- Nostr event signing (zaps, membership grants on the cold device)
- Stateless mode AND age-encrypted persistent seed — user chooses
- Graduation ritual (kid-specific wallet migration)
- Inheritance descriptor signing with our Jones-family-style policy templates
- Plausible-deniability mode (see ADR-0012)

### Airplane mode enforcement — platform tier table

| Platform | Enforcement level | Implementation |
|----------|-------------------|----------------|
| **GrapheneOS** | Truly enforced | App requests "Network permission: deny" at install; OS blocks all network regardless of airplane-mode toggle |
| **Android Device Owner (kiosk)** | Truly enforced | MDM-style deployment sets network permissions; power-user territory |
| **Standard Android** | Nudged | Launch-time network check; full-screen warning if connected; refuse-to-sign with cooldown bypass |
| **iOS** | Nudged | Network reachability check at launch; same warning pattern |
| **Tails Linux** | Enforced externally | Tails has network controls; our app assumes offline |
| **Desktop Linux** | Nudged | `nmcli radio all off` documented; banner on launch if online |
| **Desktop macOS / Windows** | Nudged only | Banner instruction; OS-level WiFi + Ethernet disable by user |

Recommended default for the signer build:
1. First launch: explainer for the detected platform — how to go offline, why it matters
2. Every launch: network state check. If connected, full-screen warning requiring explicit "I understand, sign anyway" with 10-second cooldown
3. GrapheneOS: auto-prompt for network permission denial during onboarding
4. Setting: "Refuse to run if online" — hardcore mode, no bypass

We can't force airplane mode on iOS or standard Android from inside an app. We can make it loud, explicit, and hard to ignore — which captures ~90% of the practical benefit. GrapheneOS closes the remaining gap.

### Distribution channels

| Platform | Path | Sovereignty grade |
|----------|------|-------------------|
| Android | F-Droid (preferred) + direct APK sideload + Play Store | Excellent via F-Droid |
| GrapheneOS | F-Droid + network permission denied + verified boot | Best on consumer hardware |
| iOS | App Store "Signer" variant + TestFlight + AltStore (EU DMA sideloading) | Weakest — Apple reviews all |
| Linux desktop | AppImage + `.deb` + `.rpm` direct download (SHA256 + signed) | Good |
| Tails | AppImage on persistent volume + custom Tails variant documentation | Excellent for air-gap |
| macOS | Notarized `.dmg` direct | Decent |
| Windows | Codesigned `.msi` direct | Decent |

All distributions are produced from the same Nix flake and git commit. Published SHA256 + signed release notes for every binary.

### User mental model

- **Daily wallet** — wallet-mode app on main phone / laptop. Hot wallet. Convenience model.
- **Savings / inheritance** — recommended gold standard: a used Pixel running GrapheneOS, F-Droid, permanently in airplane mode, our signer build installed. Alternative: a laptop booted to Tails with our signer AppImage on the persistent volume.
- Same UX on both devices (sign what's shown, confirm amount + address). Only the device posture differs.

## Scope impact

Originally described in ADR-0008 notes as a "~2-3 weeks of work" item. With full SeedSigner parity the realistic scope is **6-8 weeks in Q1-2027**:

- Dice entropy entry UX (~1 week)
- SeedQR + CompactSeedQR bidirectional (~1 week)
- SLIP-39 card flow (~1 week)
- Animated UR + BBQR for large PSBTs (~1 week)
- Coordinator interop testing matrix (Sparrow + Nunchuk + Specter) (~1 week)
- GrapheneOS + Tails + F-Droid distribution setup (~1-2 weeks)
- SD card / USB file flow on platforms that support it (~1 week)

This partially **replaces** the previously-deferred "native bitcoin wallet interop" scope — SeedSigner-parity IS interop since coordinators already speak BBQR and UR PSBT. The distinct "native wallet interop" line stays post-PoC for the cases where coordinators want native descriptor JSON / Sparrow-specific formats, but the core transactional interop ships with the signer.

## Consequences

**Positive:**
- A real "hardware-wallet-class" sovereignty answer for savings + inheritance without forcing a $100+ hardware purchase
- Existing SeedSigner users can scan our SeedQR backups and vice versa — meaningful interop
- GrapheneOS recipe gives state-level-adversary-resistant sovereignty on consumer hardware
- Transport (PSBT-QR) is the same one hardware wallets use — paves the road for post-PoC Coldcard / BitBox / Jade integration
- Learning curve — users who start with daily signing naturally upgrade to split-mode for savings

**Negative:**
- Q1-2027 scope grows by ~6-8 weeks; other post-PoC items may shift
- Two build artifacts per release (wallet + signer) on each platform — multiplies CI + signing + publishing effort
- Documentation burden — GrapheneOS recipe, Tails recipe, F-Droid submission process
- Can't truly emulate display trust (OS controls the screen; malware could lie) — mitigated by "signer device is ONLY a signer" discipline

## Alternatives rejected

- **Two separate repos** — drift risk on crypto code, doubled maintenance; rejected
- **Runtime mode toggle** — signer binary still contains wallet code (inert but present); weaker than compile-time removal
- **Force hardware wallet for savings / inheritance** — $100+ barrier, alienates non-technical users; hardware wallet integration stays available post-PoC as a bonus option
- **Ship only wallet mode, recommend Specter/Sparrow for cold** — loses the UX / family-lifecycle integration (graduation, inheritance descriptors, SLIP-39); rejected

## Implementation notes for Q1-2027

- `apps/client/Cargo.toml` uses feature flags: `default = ["wallet"]`, `wallet = [...]`, `signer = [...]`
- Tauri config produces two bundle IDs: `com.obs.familywallet` and `com.obs.familysigner`
- CI builds both per platform; reproducible-build verification runs on both
- F-Droid submission uses the signer bundle with its own metadata
- GrapheneOS profile = documented app permission set + network-denied install
- Tails integration = AppImage + persistent volume config template
- Testing matrix includes: Sparrow as coordinator, Nunchuk as coordinator, Specter as coordinator, our wallet-mode as coordinator, air-gap-via-QR round trip, air-gap-via-SD round trip

## Stateless + airplane-gated (amendment 2026-04-16)

### Core rule
**No keys stored on device.** The signer PWA does not use IndexedDB, localStorage, or any persistent storage for seed material. Each signing session is entered fresh by the user. Memory is zeroed on session end (page close, navigation, tab switch).

### Airplane-mode gate
**The PWA refuses to function unless the device has no network connectivity.**

- On page load, the service worker checks network state (`navigator.onLine`) and runs a lightweight connectivity probe (HEAD request to a canary URL with short timeout). If connectivity is detected, the entire UI is replaced with a full-screen gate:

```
  ┌─────────────────────────────────────────────┐
  │                                             │
  │         ⚠️  Network detected                │
  │                                             │
  │   This signer is designed to run offline.   │
  │                                             │
  │   Please enable airplane mode to continue.  │
  │                                             │
  │   [ I'm in airplane mode — retry ]          │
  │                                             │
  │   Why? Your keys never touch this device's  │
  │   storage. The only defense is the network  │
  │   being off. We check continuously.         │
  │                                             │
  └─────────────────────────────────────────────┘
```

- The gate blocks all interaction. No "dismiss" button, no "proceed anyway."
- After the user enables airplane mode and taps retry, the probe runs again. Only if offline is confirmed does the signer UI load.
- During operation, the connectivity probe runs every 5 seconds in the background. If network reappears (airplane mode disabled, WiFi reconnects, cellular resumes), the UI immediately locks back to the gate and any in-memory seed is zeroed. User must re-enter seed after returning to airplane mode.

### Session flow

1. User navigates to `signer.onbitcoinstandard.com` on a dedicated device
2. Airplane gate displays — user enables airplane mode on the device
3. Gate passes → signer UI loads
4. User chooses input method:
   - BIP-39 words (autocomplete keyboard, no paste from clipboard)
   - SLIP-39 shares
   - SeedQR scan
   - Dice entropy (fresh key generation)
5. Seed materialized in a scoped in-memory object with zeroize-on-reference-drop semantics
6. User pastes PSBT (QR scan or text input)
7. Signer reviews transaction, user confirms, signs
8. Signed PSBT displayed as QR (animated UR for large)
9. User scans signed QR onto the online coordinator device
10. Signer auto-clears memory after 60 seconds of idle OR immediately on navigation/close
11. Next session requires fresh seed entry

No "remember me." No biometric shortcut. No passphrase cache. Stateless by design.

### Why this is stronger than the original model

Original design: install once online → save encrypted seed to IndexedDB → go airplane → use offline with stored key.

Threats that original design exposed:
- Device compromised while online during install → attacker steals seed (unlikely but possible)
- User accidentally disables airplane mode mid-session → seed exposed during network window
- Physical device theft → age-encrypted file attackable offline given weak passphrase
- Malicious service worker update (if online somehow during update window) → could exfiltrate

Revised design closes all four:
- No key at rest → nothing for attacker to steal during install or later
- Network state enforced continuously → no "accidental online" window
- Physical theft → device has nothing on it beyond the cached PWA shell
- Service worker updates blocked → strict cache-only fetch handler + airplane gate means no update path even if online reappears

### Device posture per user tier

- **Casual use (small amounts):** any phone, airplane mode, stateless PWA — adequate for daily / small-value
- **Savings wallet signings (Q1-2027):** dedicated device (ideally cheap used Pixel) running GrapheneOS + Vanadium with network permission denied entirely, signer PWA cached in airplane mode
- **Inheritance signings (rare, high-value):** same as savings, plus physical device kept in a safe between uses. Heirs inherit the device or reconstruct from paper backup

### Memory hygiene in JS

JavaScript doesn't give us zeroize semantics as cleanly as Rust, but we can approximate:

- Seed stored in `Uint8Array` that we overwrite with zeros + nullify reference on session end
- No string representations of seed retained (work in `Uint8Array` throughout; words decoded only for UI display, immediately re-wiped)
- Clipboard forbidden — no `navigator.clipboard.writeText(seed)` ever; same-origin paste from clipboard warning if user pastes seed material
- `window.onbeforeunload` + `visibilitychange` triggers zeroize
- Post-zeroize, force GC pressure via large-buffer allocation burst to flush the old memory pages (best-effort)

### Mobile-native seed-entry capabilities

SeedSigner's UI is constrained by its hardware — tiny screen, 4-way joystick or sparse buttons, camera as sole input channel. Our signer runs on real phones with full touchscreens, good keyboards, high-res cameras, and meaningful CPU. We explicitly free ourselves from those hardware limitations while preserving bidirectional compatibility with SeedSigner's output formats.

**BIP-39 seed entry (12 / 15 / 18 / 21 / 24 words):**
- Full 2048-word BIP-39 autocomplete keyboard — type `abl` → see `ability`, `able`, `about`. Tap to select.
- Direct word-picker — scrollable list of all 2048 words with fuzzy search, for users who prefer tap over type
- Native support for all valid seed lengths; format chosen explicitly or auto-detected
- **Live checksum validation** — BIP-39's last word encodes a checksum of the prior words. We verify as soon as the final word lands and show a clear valid/invalid indicator. This catches typos immediately instead of after a failed derivation.
- "Did you mean…?" suggestions for obvious typos (Levenshtein distance 1 against the wordlist)
- Progress indicator — *"7 of 12 words entered"*
- Paste from clipboard **explicitly disabled** on seed fields — prevents clipboard-sniffer attacks on compromised devices
- Speech-to-text **explicitly disabled** — some mobile keyboards offer voice input; we refuse it for seed material

**SLIP-39 share entry (Shamir reconstruction):**
- Full 1024-word SLIP-39 autocomplete keyboard (different wordlist from BIP-39; keyboard switches when SLIP-39 mode selected)
- Share-progress UX — *"Share 2 of 3 entered, need 1 more"*
- Cross-share consistency validation — shares must belong to the same group; warn if user mixes shares from different wallets
- Threshold auto-detection — shares self-describe their group; user doesn't type "this is 2-of-3" metadata

**SeedQR + CompactSeedQR (bidirectional):**
- Both formats supported for scan AND display
- Camera scanning with real-time frame feedback — *"3 of 5 frames captured"* for animated QR
- Live validation as frames arrive — invalid seed reported immediately, not after the final frame
- Display with adjustable QR density for printed backup generation

**Dice entropy:**
- Auto-count rolls (99 for 256-bit, 50 for 128-bit, 77 for 192-bit)
- Visual progress ring + remaining-roll count
- Camera-based dice photo optional — pure pixel hash mixed into entropy pool as extra source

### BIP-39 passphrase (25th word) support

Every seed entry flow offers an optional passphrase. Standard BIP-39 passphrase semantics: same seed + different passphrase = entirely different wallet (different keys, different addresses, different fingerprint).

**Flow:**
1. After entering seed (via any method), the UI offers: *"Add a passphrase? Optional, but unlocks a separate hidden wallet."*
2. If checked, user enters passphrase in a masked input field
3. Re-type confirmation (no clipboard paste, no voice input)
4. One-time warning overlay (first use of passphrase in a session): *"Your passphrase cannot be recovered. Lose it and these funds are gone forever. Write it down somewhere safe."*
5. Master fingerprint displays immediately after passphrase entry — user confirms they typed what they intended (fingerprint is unique per seed+passphrase combination, so correct passphrase = expected fingerprint)

**Security properties this unlocks:**
- Infinite hidden wallets per physical seed backup
- Plausible deniability extends naturally (decoy wallet = no passphrase; real wallet = passphrase only the owner knows)
- Stolen seed alone is insufficient to spend if passphrase isn't also compromised
- Passphrase **never persisted** — held in memory only for the signing session, zeroed alongside the seed (per ADR-0010 stateless amendment)

### Master fingerprint across address types

The **master fingerprint** (root xpub fingerprint at depth 0) is a function of the seed+passphrase pair, not of any derivation path. We display it explicitly as confirmation that a given seed really does produce a consistent set of xpubs across all the standard paths.

**UI after seed + passphrase confirmed:**

```
Master fingerprint:  a1b2c3d4
(identical across every address type below — confirms one wallet)

  Native SegWit   (BIP-84)  m/84'/0'/0'   xpub…1abc
  Nested SegWit   (BIP-49)  m/49'/0'/0'   ypub…2def
  Taproot         (BIP-86)  m/86'/0'/0'   xpub…3ghi
  Legacy          (BIP-44)  m/44'/0'/0'   xpub…4jkl  [hidden by default]
```

Users see the same `a1b2c3d4` under all four derivations — proof they're looking at one wallet, four script types, same underlying keys.

The fingerprint is what coordinators (Sparrow / Nunchuk / Specter) use in PSBTs to identify which key a signing request belongs to. Displaying it gives users a stable identifier they can visually verify against.

### Arbitrary account number selection

BIP-32 allows account indexes from `0` to `2^31 - 1` (≈2.15 billion). Most wallets lock users to sequential indexes (0, 1, 2…) for UX simplicity. We don't — users can select any valid account number they want.

**UI:**

```
Account number:  [  0  ]  ← editable, defaults to 0

[ Use another number ]

┌───────────────────────────────────────────┐
│ Enter any number from 0 to 2,147,483,647  │
│                                           │
│ Account: [_______]                        │
│                                           │
│ Paths that will derive:                   │
│   m/84'/0'/<n>'  (native segwit)          │
│   m/49'/0'/<n>'  (nested segwit)          │
│   m/86'/0'/<n>'  (taproot)                │
│                                           │
│ ⚠️  Remember this number.                  │
│    Without it, this wallet is             │
│    unreachable even with the right        │
│    seed + passphrase.                     │
└───────────────────────────────────────────┘
```

Validation: integer, `0 ≤ n < 2^31`, no decimals, no scientific notation, no negative numbers.

**Why this matters:**
- **Plausible deniability / obscurity** — wallet at account `#99999` isn't found by scanners that check standard accounts 0-10
- **Memorable compartmentalization** — account `#1990` for a birth year, `#42` for personal preference, `#1337` etc.
- **Recovery against partial compromise** — if account 0 is tainted (someone knows about it), abandon and restart at an arbitrary index
- **Date-encoded accounts** — `#20260416` for a specific deposit date, self-documenting

**Security note:** arbitrary account numbers don't add cryptographic security — derivation is deterministic regardless of index. They add *obscurity*, which combined with BIP-39 passphrase makes the hidden-wallet space effectively infinite. Someone who steals the seed but not the passphrase AND the account number has zero shot at the real funds.

### Combined wallet-space math

Three independent unknowns per wallet:

| Unknown | Size |
|---------|------|
| Seed (24 words from 2048) | ~256 bits of entropy |
| Passphrase (arbitrary UTF-8) | effectively unbounded |
| Account number (0 to 2^31) | ~31 bits |

A thief who steals two of three has zero computational advantage finding the third — passphrase alone is unboundedly large, account space is 2 billion options. All three are required to recover funds. This is genuinely what hardware-wallet power users expect and what most mainstream mobile wallets don't expose.

### Packaging later

As the user noted in the 2026-04-16 direction: "then we can directly package it when ever we want and how ever we wnat once it is stabilised."

Packaging paths once the PWA is stable:
- Tauri shell wrapping the same UI → real binary, compile-time-removed network code (the originally-designed Tauri signer)
- Dedicated hardware (Raspberry Pi Zero + display + camera, SeedSigner-style) running the PWA on a stripped-down browser
- F-Droid APK for GrapheneOS distribution
- Tails persistent-volume AppImage

All of these become natural evolutions once the PWA is audited and stable. The stateless + airplane-gated model translates to each packaging without changing the threat model.

## References

- ADR-0008 — key unlock model (signer amendment overrides §2 for the signer sub-app specifically — no IndexedDB for keys)
- ADR-0009 — UX principles (signer UX is bound by same principles)
- ADR-0014 — PWA-first shell (signer as one of the sub-apps)
- ADR-0016 — sub-app architecture (signer = one of six sub-apps)
- `https://seedsigner.com` — feature parity reference
- BBQR spec, UR spec — QR payload formats
- BIP-174 (PSBT), BIP-380 (Output descriptors)
