# ADR-0002: Day-one security baseline

Date: 2026-04-15
Status: Accepted

## Context

Retrofitting security is painful and rarely complete. We want to pick the non-negotiable security posture at commit #1 and build toward it — rather than ship a v1 on conventional web-app defaults (TLS, passwords, plain SQLite) and bolt on sovereignty later.

## Decision

The following eight elements are **day-one baseline**. Anything that contradicts them needs a new ADR to override this one.

### 1. Nostr keys as identity

A family member's identity in this app is a Nostr keypair (npub). Rajesh already runs Nostr infra; reusing the same key across Nostr + family wallet means one sovereign identity per person. No email, no username, no OAuth.

### 2. age-encrypted key material in browser IndexedDB; WebAuthn/passkey optional as session-auth convenience

**Revised 2026-04-15 (twice) — see ADR-0008 for key unlock rationale; ADR-0014 for PWA storage primitive.**

Signing keys are stored as age ciphertext inside IndexedDB (browser origin-isolated), encrypted with a Curve25519 recipient derived from the user's passphrase via Argon2id. WebAuthn/passkey is used optionally as a session-token cache gated by biometric, not as the root of custody. Installed PWAs request `navigator.storage.persist()` to protect the IndexedDB from browser eviction.

The original framing ("Secure Enclave / Keystore / TPM for on-device key storage") oversold what consumer hardware provides for Bitcoin signing: no consumer Secure Element supports secp256k1, so no consumer wallet actually achieves "key never leaves the enclave" for Bitcoin keys. The wrap-and-release pattern works the same whether the wrap key lives in the OS keystore or a user-space file. Keeping the root of custody in pure Rust removes OS-vendor trust from the sovereignty story. Real hardware-backed Bitcoin signing requires a dedicated hardware wallet (Coldcard / BitBox / Jade) — post-PoC integration.

### 3. age-encrypted E2EE backup blobs

Backups are encrypted with [age](https://age-encryption.org/) before leaving the device. age supports multiple recipients natively, which maps cleanly to "encrypt to passphrase AND to N Shamir shares AND to a parent's public key" — a single ciphertext, multiple recovery paths. Server stores the ciphertext; cannot decrypt.

### 4. Noise protocol for P2P transport

Any direct device-to-device channel (BLE, local-network, Nostr-DM fallback) uses the Noise protocol handshake — the same one Lightning uses. Forward secrecy, no CA dependency, mutually authenticated.

HTTP API between client and our server still uses TLS (practical necessity), but the authentication is Noise-style: client signs a server challenge with their Nostr key; no passwords, no session cookies.

### 5. Passkey / WebAuthn for daily device unlock

Daily app unlock uses the OS passkey (Face ID / Touch ID / Windows Hello / Android biometrics). The passkey unlocks a wrapping key that decrypts the app's local state — the signing key stays in Secure Enclave regardless. Passphrase is only for recovery on a new device.

### 6. Encrypted local SQLite

All on-device state (chore history, contacts, family graph cache, event log) lives in SQLCipher-backed SQLite. Full-disk encryption on the OS is not sufficient — we want the app's data encrypted at rest even on a jailbroken or imaged device.

### 7. Transport-agnostic signed-event envelope

Every state-changing action (membership grant, chore approval, VTXO transfer, limit change) is wrapped in a signed-event envelope. The envelope is transport-agnostic — the same signed blob can traverse HTTP, Nostr relays, BLE, or a QR code between phones. Clients verify the signature, not the transport.

This is what makes future Nostr / BLE / mesh transports additive rather than rewrites.

### 8. Reproducible builds from commit #1

The build pipeline (Nix + pinned Node + pnpm lockfile + deterministic Vite settings) produces bit-for-bit identical JS bundles from a given commit. Users who want to verify can fetch the live bundle from `wallet.onbitcoinstandard.com`, hash it, and compare against the published SHA-256 (released as a Nostr kind:1063 event signed by our project key + GitHub release notes). Retrofitting reproducibility is ten times the work of setting it up now; we do it on day one even though the app is PWA, not native.

## Consequences

**Positive:**
- No passwords anywhere in the system — nothing to leak, breach, or phish
- Server compromise = privacy incident (contact graph, chore history) but not a funds incident
- Every day-one choice is also a sovereignty story we can tell users
- Nostr identity reuse = Rajesh's existing followers / family already have the identity primitive

**Negative:**
- Higher upfront engineering cost in Q2 than a "TLS + passwords + plain SQLite" shell would be
- age and Noise protocol libraries in JS/TS are less battle-tested than OpenSSL — we'll vendor and audit the specific versions we use
- Reproducible builds slow iteration initially (Nix learning curve) — but pays off by mainnet
- Passkey-only unlock means a user who loses all devices AND their passphrase AND all Shamir shares is unrecoverable — by design, but must be communicated clearly

## Alternatives rejected

- **Conventional web-app stack** (TLS + email/password + OAuth + plain SQLite): faster to ship, incompatible with the sovereignty claim, and the retrofit cost is prohibitive.
- **Fully offline / no server**: considered, rejected. We need some server for ASP coordination, push fan-out, chore workflows across parent/kid devices. Keeping the server minimal and operator-blind is the better answer.

## Implementation notes for Q2

Add to the `docs/YEARLY-PLAN.md` Q2 checklist:
- age-encryption binding selected (pure JS `age-encryption.js` or WASM port)
- Nostr key generation + age-encrypted file storage (ADR-0008); optional OS keystore session cache via `keyring` crate (feature-flagged)
- Noise protocol TS library selected (e.g. `@welldone-software/noise-ts` or vendored)
- SQLCipher integration path (Capacitor plugin or community port)
- Nix flake scaffolded alongside the package.json so reproducible builds exist before we have much to build
- Signed-event envelope schema in `packages/shared/`
