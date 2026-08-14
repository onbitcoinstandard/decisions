# ADR-0008: Key unlock model — age-encrypted file primary, OS keystore optional

Date: 2026-04-15
Status: Accepted (amends ADR-0002 §2) — amended by ADR-0020: limits this model to hot/identity keys; the stored-key "savings/inheritance sovereignty mode" is superseded by the amnesic signer (Class 1)

> **Storage primitive revised by ADR-0014:** since we moved to PWA-first, the age-encrypted key material is stored in IndexedDB (browser origin-isolated) rather than a filesystem path. The core model (age + Argon2id-derived recipient + optional session cache) is unchanged. The session cache uses WebAuthn/passkey in the browser instead of the `keyring` Rust crate's OS keystore API. Every semantic property in this ADR still applies; only the storage backend and biometric-access layer differ.

## Context

ADR-0002 initially committed to "Secure Enclave / Android Keystore / TPM" as the primary on-device key storage. On closer inspection this mischaracterizes what those APIs actually do for Bitcoin keys specifically, and creates an unnecessary dependency on OS-vendor trust for our sovereignty story. We revise.

### The hardware-backed myth for Bitcoin

Consumer Secure Elements (iOS Secure Enclave, Android StrongBox, Windows TPM 2.0, macOS Secure Enclave) **do not support secp256k1**. They support P-256, P-384, RSA — not the curve Bitcoin uses.

Consequently, no consumer Bitcoin wallet achieves "key never leaves the enclave." The standard pattern — used by Specter, Sparrow, Nunchuk, BlueWallet, and conceptually the same as password managers like Bitwarden and 1Password — is:

1. Enclave holds a wrap-key (P-256 or AES, platform-supported algorithm)
2. App stores `E_wrap(secp256k1_private)` on disk or in keystore
3. At signing time: biometric unlocks enclave → enclave unwraps → secp256k1 key enters **app memory** → libsecp256k1 signs in user-space → zeroize memory
4. The Secure Element is a biometric-gated release mechanism, not a signing engine

The secp256k1 key spends time in app memory during every signing operation. Hardware-in-the-loop limits the attack window and raises the bar, but does not achieve "key never in memory." Real "key never in memory" for Bitcoin requires a dedicated hardware wallet (Coldcard / BitBox / Jade) — a separate physical device with a custom secure element programmed for secp256k1. That's a post-PoC integration.

### What OS keystore genuinely gives us

1. **Inter-app isolation** — other apps running as the same user cannot read a Keychain entry. A user-space file is readable by co-user processes.
2. **Native biometric UX** — Face ID / Touch ID / Windows Hello prompts look right.
3. **Smoother App Store review** — Apple occasionally flags apps not using Keychain for credentials.

These are real but not load-bearing for our security model.

### What OS keystore costs us

1. Platform-specific code — per-OS APIs abstracted by the `keyring` crate; still more surface than one pure-Rust path
2. OS vendor trust — Apple, Google, Microsoft in the trust chain for a sovereign bitcoin wallet is a credibility wart
3. Reduced portability — headless Linux, Docker, minimal distros all become edge cases
4. Reproducible-build complexity — platform-specific code paths add variance

## Decision

**Primary: age-encrypted key file, cross-platform.** The signing key is stored as an age ciphertext at a platform-appropriate path. The key is encrypted with a Curve25519 recipient derived from the user's passphrase via Argon2id (ADR-0004 parameters). Decryption happens in app memory at unlock time. After signing, the plaintext key is zeroed using the `zeroize` crate with `#[derive(ZeroizeOnDrop)]` on the key struct.

**Optional convenience layer: OS keystore for session tokens only.** Platforms with biometric APIs (iOS, Android, macOS, Windows Hello) can cache a session token in the OS keystore, gated by biometric. The session token is valid for a configurable timeout (default: 15 minutes for daily wallet, session-only for savings/inheritance). The session token does NOT unlock the age file directly — it lets the app reuse an in-memory-decrypted key for multiple signings within the session without re-prompting the passphrase.

**Absolute fallback: passphrase on every unlock.** Environments with no OS keystore (headless Linux, Docker, minimal distros, or users who disable the convenience layer) require the passphrase on every app start and after every timeout. This mode is available as a setting for users who prefer zero OS-vendor trust.

### Storage layout

```
<data_dir>/obs-family-wallet/
├── keys/
│   ├── <pubkey-1>.age         -- age-encrypted key file (primary)
│   └── <pubkey-1>.meta.json   -- unencrypted metadata (name, created_at, wallet refs)
├── wallets.db                 -- SQLCipher-encrypted SQLite (ADR-0002 §6)
└── session.bin                -- ephemeral session token cache (auto-deleted on app close)
```

Where `<data_dir>` is platform-appropriate:
- Linux: `$XDG_DATA_HOME` (usually `~/.local/share`)
- macOS: `~/Library/Application Support`
- Windows: `%APPDATA%`
- iOS/Android: app sandbox

### File permissions

`keys/*.age` files written with mode `0600` (user-read/write only) on Unix; ACL-restricted on Windows. This is not a security boundary against OS-vendor adversaries but raises the bar for co-user processes.

### Key unlock flow (daily wallet, convenience mode)

```
App start
  → Is there a cached session token? Check OS keystore (biometric-gated)
      ├─ Yes, valid, biometric confirms → unwrap session token → decrypt age file → key in memory
      └─ No / expired / biometric declined → prompt passphrase → Argon2id → decrypt age file → key in memory → optionally cache new session token
Sign transaction
  → Use in-memory key → zeroize working buffers immediately after sign
Session timeout or app close
  → Zeroize in-memory key → clear session token
```

### Key unlock flow (savings / inheritance, sovereignty mode)

```
Signing event
  → Prompt passphrase (always) → Argon2id → decrypt age file → key in memory → sign → zeroize immediately
  → No session token. Next signing re-prompts passphrase.
```

This is the default for savings (2-of-3) and inheritance wallets — these sign rarely, so UX cost is minimal, and the security boost matters more for large amounts.

### Memory hygiene

Use the `zeroize` crate (`#[derive(Zeroize, ZeroizeOnDrop)]` on key structs). Signing happens in a scoped block so the key is zeroed on scope exit. No `Arc<PrivateKey>`, no `Clone` impls on secret types, no serialization that could leak into logs.

## Consequences

**Positive:**
- Identical cross-platform code path for custody
- Zero OS-vendor dependency for the core security model; keystore is optional UX polish
- More defensible sovereignty story — no "trust Apple/Google" caveat
- Simpler reproducible builds — one code path
- Works in Docker, headless Linux, minimal distros
- Matches what every consumer Bitcoin wallet does in practice, without pretending otherwise
- User-controlled: set "sovereignty mode" per wallet and the convenience layer is never consulted

**Negative:**
- Per-signing memory exposure — same as any consumer wallet, but we're explicit about it instead of hiding behind "hardware-backed" marketing
- Other co-user processes could in principle read the age file (mitigated by `0600` perms, by full-disk encryption being a reasonable OS baseline, and by the file being useless without the passphrase)
- iOS App Store review may ask why we don't use Keychain primarily — answer: we use it optionally as a session cache, and we're a bitcoin wallet where Secure Enclave cannot hold the key anyway
- Requires careful memory hygiene (zeroize discipline) — buys us less than hardware would, but still reduces the attack window

## Alternatives reconsidered and rejected

- **OS keystore as primary** (ADR-0002 original): dependency on OS vendor for no real security benefit on Bitcoin keys; the wrap-and-release pattern works the same in a user-space file; decision reversed here.
- **Passphrase on every sign, no session**: usable for savings/inheritance (and chosen as the default mode for those wallets), but hostile for daily-wallet UX with kids. Rejected as the primary mode for daily wallets.
- **Full in-enclave signing via secp256k1**: not supported by any consumer Secure Element in 2026. Rejected because not possible.
- **Dedicated hardware wallet (Coldcard etc.) for daily**: overkill for small-amount daily use; right answer for savings/inheritance (post-PoC).

## Implementation notes for Q2

- Adopt `age` crate for encryption, `argon2` for KDF, `zeroize` for memory hygiene
- Platform paths via `directories` crate
- Optional keystore convenience via `keyring` crate (feature-flagged; can be disabled at build time for pure-Rust-only reproducible builds)
- Unit tests verify: passphrase-incorrect fails cleanly, session-token-tampered fails cleanly, zeroize actually clears memory (best-effort — Rust can't fully guarantee memory semantics but best practice applies)
- Benchmark Argon2id params on Rajesh's weakest target device to set mobile-degraded defaults in Q3

## Amendment to ADR-0002

Replace §2 of ADR-0002 with:

> **2. age-encrypted file for on-device key storage; OS keystore optional as session-auth convenience**
>
> Signing keys are stored as age ciphertext in the app data directory, encrypted with a Curve25519 recipient derived from the user's passphrase via Argon2id. OS keystore is used optionally as a session-token cache gated by biometric, not as the root of custody. See ADR-0008 for detailed rationale, including why the conventional "Secure Enclave" framing oversells what consumer hardware provides for Bitcoin secp256k1 signing.
