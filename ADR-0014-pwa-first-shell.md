# ADR-0014: PWA-first shell strategy

Date: 2026-04-15
Status: Accepted (supersedes ADR-0006) — store-abstinence amended by ADR-0019; keys-object-store scoped by ADR-0020: Class 2/3 ciphertexts only, wallet is watch-only for cold funds

## Context

ADR-0006 committed us to Tauri 2.0 as the single cross-platform shell. On closer reflection — informed by the success of nsec.app, Mutiny Wallet, and other bitcoin/Nostr PWAs in production — the sovereignty case for a **PWA-first architecture** is stronger than the convenience case for Tauri:

1. **App Store independence.** Apple and Google have pulled bitcoin self-custody apps in the past. With a PWA served from our own domain, they have no leverage. This alone is load-bearing for the sovereignty thesis.
2. **Zero App Store fees.** No $99/yr Apple Developer Program, no $25 Play Console, no EV codesigning ($400/yr). Cost envelope drops.
3. **Instant updates.** No review delays, no rejection cycles, no TestFlight bottleneck.
4. **One artifact, all platforms.** A single PWA install works on iOS, Android, desktop (Chrome / Firefox / Safari), Linux, Windows, macOS. No per-platform native code.
5. **Precedent exists.** nsec.app has held user nsecs as a PWA for years without incident. Mutiny ran a full self-custody Bitcoin + Lightning wallet as a PWA until business wind-down in 2024 — the model works.
6. **Installed-PWA storage is reliable** in 2026 — `navigator.storage.persist()` grants protected storage to installed PWAs on iOS 16.4+, Android, Chrome desktop, Firefox. The 7-day-eviction horror story applies to non-installed tabs, not installed PWAs.

The original concern — "PWA trust is weaker than native codesigned binary" — is real but addressable with published bundle hashes, immutable Nostr release notes, and service-worker-controlled update discipline.

## Decision

**Ship the wallet as an installed PWA.** Same Vite + React + TypeScript UI code we've been planning, minus the Tauri Rust shell. The PWA is served from our own domains and installs via "Add to Home Screen" on every major platform.

### Three-subdomain architecture

See ADR-0015 for identity details. High-level:

- `wallet.onbitcoinstandard.com` — wallet PWA, installed
- `signer.onbitcoinstandard.com` — signer PWA, installed with strict offline service worker
- `onbitcoinstandard.com` — NIP-05 + (future) LUD-16 identity host + OBS content site
- `api.onbitcoinstandard.com` — server API (signed-event endpoints, backup blobs, ASP coordinator)

Each subdomain is its own **browser origin** — storage is isolated by browser security policy. The signer's IndexedDB cannot be read by the wallet's code even if the wallet is compromised.

### Stack

- **UI:** Vite + React + TypeScript + Tailwind + shadcn/ui — unchanged from ADR-0006
- **Shell:** PWA (service worker + manifest + installable)
- **Crypto (WASM):**
  - `age-encryption.wasm` — portable backup (ADR-0004)
  - `argon2.wasm` — passphrase KDF
  - `slip39.wasm` or JS port — Shamir shares
  - `@noble/curves` — secp256k1 signing (pure JS, audited, fast)
  - `bdk-wasm` — on-chain descriptors + PSBT (Q1-2027 savings/inheritance)
- **Storage:** IndexedDB for encrypted blobs + `navigator.storage.persist()` requested at install time
- **Auth:** WebAuthn/passkey for biometric session unlock (ADR-0008 pattern, browser-native)
- **Transport:** `fetch` with signed-event envelopes (ADR-0003)
- **QR:** `@zxing/library` for scan, `qrcode` for display

Bundle size target: <3MB total after code-split, ~800KB critical path. WASM modules lazy-loaded on first use.

### Storage primitive — revised

The age-encrypted key material lives in **IndexedDB**, not a filesystem path. Logical structure:

```
IndexedDB: wallet-obs
├── keys object store
│   ├── pubkey-1: { ciphertext: Uint8Array, meta: {...} }
│   └── pubkey-2: { ... }
├── events object store       -- cached signed events
├── contacts object store
└── ui_prefs object store
```

IndexedDB ciphertext is still age-encrypted (passphrase-derived Argon2id recipient). The browser's origin isolation + encryption provides equivalent security to the filesystem-and-OS-permissions model in ADR-0008 for PWA deployment.

**Key difference from ADR-0008:** no OS keystore convenience layer for the wallet PWA. WebAuthn/passkey handles biometric-unlocked session caching (browser-native, cross-platform, no `keyring` crate dependency). The age file remains the source of truth.

### Reproducible builds for PWAs

Deterministic JS bundling is well-understood:

1. Nix flake pins Node version + pnpm lockfile
2. Vite build runs with deterministic settings (no timestamps in output, stable module ordering)
3. Same source tree → identical bundle bytes
4. Published artifacts: bundle SHA-256 hash, signed by our Nostr key, published as Nostr kind:1063 event + GitHub release
5. Users (or auditors) can `curl wallet.onbitcoinstandard.com/app.<hash>.js | sha256sum` and compare

**Subresource Integrity (SRI)** locks external resources — our `index.html` references our own bundle by hash, so a swapped bundle breaks the page visibly rather than silently.

**Service Worker update policy:** checked once on page load when online, updates fetched in background, user prompted to apply. No silent version swap — user sees "update available" banner before anything runs.

### Signer PWA — separate subdomain, offline-first service worker

`signer.onbitcoinstandard.com` serves a **stripped-down signer-only build** of the same codebase:

- No wallet UI; only seed entry, PSBT signing, QR exchange, backup export
- Service worker refuses all `fetch` requests that aren't cache hits — no network even when available
- User installs once (while online), then goes airplane forever
- Updates require explicit reinstall — no silent JS injection

```js
// signer-sw.ts
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) =>
      cached ?? new Response('Offline-only mode', { status: 503 })
    )
  );
});
```

Target platforms for signer:
- **GrapheneOS + Vanadium** — browser-level network permission denied; strongest mode available on consumer hardware
- **Tails Linux** — Firefox with our PWA cached to persistent volume; airplane-mode default
- **Airplane-mode Pixel with Chrome** — standard user-accessible fallback
- **Airplane-mode iPhone with Safari** — supported but iOS Safari SW has had quirks; document caveats

### Optional Tauri signer build for paranoid users

The signer PWA is strong (~85% of Tauri signer strength). For users who specifically want compile-time network removal — savings/inheritance users, state-level-adversary threat models — we **can** ship a Tauri shell wrapping the same UI code as a side artifact, post-PoC.

This isn't part of v1. It's kept as an affordance because the UI is 100% web-native; the Tauri wrapping cost is small when we want it. Decision: defer to post-PoC, revisit if a specific user or pilot family requests it.

## Consequences

**Positive:**
- Apple + Google have no leverage — we ship when we want, what we want
- Significant cost savings (~$500/yr in fees eliminated)
- Faster Q2 velocity — no Rust/Tauri learning curve, no platform-specific native code
- Q3 mobile platform spike collapses to zero — the PWA already works on iOS + Android from day one
- One codebase, one deployment, one test matrix
- Genuinely cross-platform with real parity — Linux, Windows, macOS, iOS, Android, ChromeOS all work the same way
- Origin isolation between wallet + signer subdomains is browser-enforced security, not code discipline

**Negative:**
- Supply-chain trust is weaker than native codesigned binary — mitigated by published bundle hashes + Nostr-attested releases + service worker update discipline, but not erased
- Bundle size considerations (WASM crypto adds ~1-2MB) — acceptable on modern connections
- iOS Safari has historically lagged on PWA features — we accept known-good subset, document workarounds
- Web Bluetooth not in Safari — acceptable since we dropped BLE from transports entirely (see amended ADR-0003); QR is the canonical in-person transport and works universally
- Web NFC not in Safari — NFC cards already deferred post-PoC, no change
- Performance on low-end Android worse than native Rust for heavy crypto — acceptable for our load

**Explicitly accepted risks:**
- Malicious-push supply-chain attack: mitigated by SW update discipline + published hashes, not eliminated
- Users on untrusted networks visiting install URL: mitigated by HTTPS + certificate pinning via CAA records, SRI on bundle
- Apple/Google changing PWA support policies: possible but low probability in 2026; lived through iOS 16.4 PWA improvements already

## Amendments to existing ADRs

- **ADR-0006 (Tauri stack)** — superseded by this ADR; marked as such
- **ADR-0002 §2 storage** — IndexedDB backend for age ciphertext instead of filesystem paths
- **ADR-0002 §reproducible builds** — JS deterministic build instead of Nix+Rust binary build; Nix still pins the toolchain
- **ADR-0008 key unlock** — WebAuthn/passkey replaces OS keystore as the biometric session cache layer; age file unchanged
- **ADR-0010 signer architecture** — separate PWA subdomain with offline-first SW replaces `--features signer` compile flag; optional Tauri signer shell deferred post-PoC

All amendments done in separate commits immediately after this one.

## Implementation notes for Q2

- Vite PWA plugin (`vite-plugin-pwa`) for service worker generation + manifest
- `@noble/curves`, `@noble/hashes`, `@noble/ciphers` for pure-JS crypto (audited, fast, small)
- WASM bundles lazy-loaded: age via `age-encryption.js`, Argon2 via `argon2-browser` or `hash-wasm`, SLIP-39 via JS port or WASM
- `@zxing/library` for QR scan; `qrcode` for display
- IndexedDB wrapper: `idb` (small, Promise-based)
- WebAuthn via browser native API; fallback to passphrase entry
- Service worker uses Workbox for standard patterns; custom fetch handler on signer domain for strict offline
- Reproducible build: Nix flake + pinned pnpm + SHA256 in release notes + Nostr-signed kind:1063 event

## Alternatives reconsidered and rejected

- **Tauri-first (ADR-0006 original)** — superseded. Reason: sovereignty cost of Apple/Google dependence outweighs the reproducibility + performance advantages for our user base and budget.
- **PWA wrapped in Tauri for desktop only** — possible but doubles the maintenance surface. PWA alone covers desktop adequately.
- **Native React Native for mobile + PWA desktop** — two codebases, no unified build; rejected.
- **Electron for desktop + PWA for mobile** — Electron is heavier than Tauri and everyone knows it; worse on every axis.

## References

- ADR-0002 — security baseline (amended below this ADR)
- ADR-0003 — transport-agnostic signed-event envelope (unchanged)
- ADR-0004 — portable `.age` backup (unchanged, format works identically whether storage is IndexedDB or filesystem)
- ADR-0006 — Tauri stack, superseded
- ADR-0008 — key unlock model (amended for WebAuthn)
- ADR-0010 — signer architecture (amended for PWA subdomain approach)
- ADR-0015 — NIP-05 identity + domain architecture (companion to this ADR)
- nsec.app — production PWA Nostr signer, reference implementation
- Mutiny Wallet — PWA Bitcoin + Lightning self-custody wallet (now defunct, but proved the model)
