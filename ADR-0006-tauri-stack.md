# ADR-0006: Tauri 2.0 as the single cross-platform shell

Date: 2026-04-15
Status: **Superseded by ADR-0014 (PWA-first shell)**

> This ADR captured the Tauri-first decision that held for most of April 2026. On further consideration, the sovereignty cost of Apple/Google dependence outweighed the reproducibility and performance advantages for our user base. See ADR-0014 for the revised shell strategy (PWA served from our own domains, no App Store), ADR-0015 for the companion identity + domain architecture. The Tauri signer build is preserved as an optional post-PoC artifact for users wanting compile-time network removal.

## Context

We need one codebase targeting iOS, Android, macOS, Linux, and Windows. The two realistic options in 2026 are:

- **Capacitor** — WebView-based, native plugin ecosystem via JS/TS bridge, mature mobile story, Electron or Tauri for desktop
- **Tauri 2.0** — WebView-based, native plugin ecosystem via Rust bridge, native mobile support (iOS + Android) stabilized in 2024, Rust backend

Our security baseline (ADR-0002) commits us to age, Noise protocol, Argon2id, SLIP-39, BDK, Nostr primitives, and a Bitcoin stack — every one of which has a canonical, audited Rust implementation. The question is whether we put those on the native side (Rust, via Tauri) or behind a JS/WASM shim (via Capacitor).

## Decision

**Tauri 2.0 for all five platforms.** Single codebase, single toolchain.

### Rationale

1. **Native-side audited Rust, not WASM ports.** BDK, age-encryption, snow (Noise), argon2, SLIP-39, nostr-sdk, rust-bitcoin are production-grade Rust. Tauri runs them directly; Capacitor would force WASM compilation, JS ports, or cross-bridge JSON round-trips for every crypto operation.
2. **Reproducible builds are easier in Rust.** Cargo lockfiles + Nix flake give deterministic outputs. Same claim is harder with node_modules + native plugin mix in Capacitor.
3. **Smaller binaries.** Tauri desktop bundles are ~5–15 MB; Electron equivalents are 100+ MB. Mobile gap is smaller but still favors Tauri.
4. **Sovereignty alignment.** Rust-first, no Electron Node runtime, minimal attack surface. Fits the OBS ethos.
5. **One toolchain.** The ADR-0002 reproducible-build pipeline (Nix + Rust + Vite + Tailwind) is the same for all five platforms.

### UI layer stays vanilla web

React + TypeScript + Vite + Tailwind + shadcn/ui — no Tauri-specific or Capacitor-specific APIs in component code. The native bridge is wrapped in `packages/shared/` so a swap of shell technology touches the bridge layer only.

This also means developers can run the UI in a plain browser tab during iteration — no Tauri dev server needed for most UI work.

### Native crates we'll wire in (Q2/Q3)

| Concern | Rust crate | Exposed to JS as |
|---------|-----------|------------------|
| age encryption | `age` | `encrypt_backup(recipients, plaintext) → ciphertext` |
| Argon2id KDF | `argon2` | `derive_kek(passphrase, salt, params) → Uint8Array` |
| SLIP-39 | `slip39` (audit candidate) | `shamir_split(secret, policy) → shares[]` / `shamir_combine(shares) → secret` |
| Noise protocol | `snow` | `noise_session_new(…)` / `noise_encrypt/decrypt` |
| Bitcoin descriptors, multisig (Q1-2027) | `bdk` | `descriptor_parse`, `psbt_sign`, etc. |
| Nostr identity + events | `nostr-sdk` | `npub_generate`, `event_sign`, `event_verify` |
| Secure Enclave / Keystore | `tauri-plugin-biometric` + platform-specific (iOS Keychain via `security-framework`, Android Keystore via JNI) | `secure_store`, `secure_sign` |

### Tauri mobile — the honest caveats

Tauri mobile (iOS/Android) reached general availability in late 2024 and is rapidly maturing through 2025–2026. Specific gaps to validate in Q2:

- **Biometric / Secure Enclave / Keystore plugins** — community plugins exist; may need to write our own thin wrapper if the existing ones don't cover what ADR-0002 requires
- **BLE** — `tauri-plugin-blec` exists but less battle-tested than `@capacitor-community/bluetooth-le`
- **Push notifications** — APNs + FCM plugin less mature than Capacitor's
- **App Store / Play Store submission tooling** — newer than Capacitor's tooling; watch for edge cases

**Mitigation / escape hatch:** if a specific mobile plugin gap blocks shipping, we fall back to Capacitor for mobile only. The React UI layer + the `packages/shared/` bridge abstraction means this is a shell-layer swap, not a rewrite. The Rust crypto crates we're vendoring would still be callable — via Capacitor's WKWebView / Android WebView JS bridge to a small native glue module — though with more friction. Day-one assumption is Tauri throughout; we re-evaluate quarterly.

## Consequences

**Positive:**
- All five platforms from one codebase and one toolchain
- Security-critical code runs as audited Rust, not JS/WASM
- Smaller bundles, easier reproducible builds, better cryptographic perf
- Desktop ships day one (not Q3) because Tauri desktop is essentially free once mobile works
- Story aligns with sovereignty ethos — no Electron, no Node runtime in release binaries

**Negative:**
- Mobile plugin ecosystem is younger; some native features will be DIY
- Tauri mobile churn risk — APIs may shift through 2026; pin versions carefully
- Team must be comfortable with Rust on the backend side (Rajesh's current repo uses JS/TS primarily — Rust learning curve acknowledged)
- If we hit a mobile blocker and must split to Capacitor-for-mobile, we maintain two shell layers temporarily

## Alternatives rejected

- **Capacitor everywhere + Electron for desktop** — more mature today, but keeps us in the JS/WASM crypto world and makes reproducible builds harder. Electron's bundle size alone is a sovereignty smell.
- **Capacitor-for-mobile + Tauri-for-desktop** — two shells, two plugin systems, two native glue layers. Acceptable as a fallback, unacceptable as the day-one plan.
- **React Native** — native views (not WebView) gives best mobile perf but excludes desktop and forces a very different UI model from the web-native story. Not a fit for our case.

## Platform order

**Q2 2026: desktop first (macOS, Linux, Windows).** Tauri desktop is its mature primary platform — no plugin risk, fastest path to Rajesh dogfooding. Mobile is deferred to Q3.

**Q3 2026: mobile (iOS + Android).** Tauri mobile has another quarter of maturity by then; we start wiring Secure Enclave, BLE, and push notification plugins. Mitigation/escape-hatch note above still applies — if any mobile plugin blocker appears, Capacitor-for-mobile is a shell-layer swap, not a rewrite.

Rationale for the order: Rajesh can dogfood daily on Linux/macOS sooner, the reproducible-build pipeline is stood up against desktop first (simpler), and mobile ships with the family-MVP milestone in Q3 where it's genuinely needed (chore approvals on kids' phones). No user-facing feature is blocked by not having mobile in Q2.

## Implementation notes for Q2 (desktop)

- Nix flake includes Rust toolchain (pinned) alongside Node
- `apps/client/` scaffold uses `create-tauri-app` with the React + TypeScript + Vite template
- Desktop targets enabled from commit #1: macOS, Linux, Windows
- Mobile targets stubbed in `tauri.conf.json` but not built in Q2
- `packages/shared/` includes the bridge abstractions — all native calls go through a thin TS interface so the shell is swappable

## Implementation notes for Q3 (mobile)

- Enable iOS + Android builds in Tauri config
- Spike: biometric / Secure Enclave / Keystore plugin — adopt community plugin or write our own
- Spike: BLE plugin for in-person approval flows (ADR-0003)
- Spike: APNs + FCM push notification plugin
- If any spike reveals a blocker we can't resolve in-quarter → trigger the Capacitor-for-mobile escape hatch, document in a new ADR
