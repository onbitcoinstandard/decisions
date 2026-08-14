# ADR-0007: SLIP-39 for Shamir Secret Sharing

Date: 2026-04-15
Status: **Superseded in part (2026-08-13):** the Shamir standard was changed to **SSKR** (`ur:crypto-sskr`, QR-native) during the signer-spec sessions — see spec §5 and §12 (Master v1+). The group-policy rationale below still informs the SSKR share design; the age-recipient bridge carries over.

## Context

ADR-0004 committed us to paper Shamir shares as one of the multi-recipient backup paths. The library/standard choice was deferred. The two realistic options are:

- **SLIP-39** — Trezor's published standard, production-proven since 2017, BIP-39-style word mnemonics, native support for group-of-groups policies
- **Custom Curve25519-native Shamir** — simpler integration with age recipients, but less battle-tested, no ecosystem interop

## Decision

**SLIP-39.** All Shamir splits in the app use the SLIP-39 standard.

### Coverage check — all our use cases

| Use case | SLIP-39 native support |
|----------|------------------------|
| Adult 2-of-3 paper backup | ✅ Simple threshold |
| Kid 1-of-1 single paper slip | ✅ Trivial case |
| Jones-family inheritance: 2-of-3 groups, each a k-of-n subgroup | ✅ Groups-of-groups is a first-class SLIP-39 feature |
| Passphrase-strengthened shares | ✅ SLIP-39 passphrase extension |
| Handwritten by a kid | ✅ BIP-39-style words (kid writes 20–33 words per share) |
| Recoverable outside our app with an open-source tool | ✅ Multiple audited open-source implementations |

**Note:** we are NOT pursuing Trezor or any specific hardware-wallet interop as a goal. SLIP-39 happens to be a published standard, which we value for longevity and for the existence of open-source recovery tools — not for cross-vendor interop. Our users are not expected to move shares between devices.

### How we bridge SLIP-39 to age recipients

SLIP-39 splits a 128-bit or 256-bit master secret. age recipients are Curve25519 x25519 identities. We bridge them with a HKDF step:

1. Generate random 256-bit master secret
2. Derive x25519 identity via `HKDF-SHA256(master_secret, salt="obs-wallet-age-v1", info="recipient")`
3. Use that identity as an age recipient when encrypting the backup
4. Shamir-split the master secret using SLIP-39 for paper shares

To recover: combine enough SLIP-39 shares → get the master secret → HKDF to the x25519 identity → decrypt the age file.

This bridge is documented in code and tested; the computation is deterministic.

### Rust crate selection

The Rust SLIP-39 ecosystem has multiple implementations as of 2026. We evaluate on:
- Audit status (Trezor's internal reference implementation is the ground truth)
- Test vectors covered
- Maintenance activity

Candidate: `slip39` crate on crates.io. Exact version pinned after a Q2 audit spike. If no satisfactory Rust crate exists, we port Trezor's Python reference (`python-shamir-mnemonic`) to Rust with test-vector parity.

### Why not custom Shamir

Building our own Shamir implementation — even directly over Curve25519 — means:
- No ecosystem interop (user can't recover on Trezor, Keystone, etc.)
- Audit burden we must carry ourselves
- No shared test vectors with prior art
- Re-inventing a solved problem

SLIP-39 is the industry standard for Bitcoin Shamir backups in 2026. Using it is a write-once, benefit-forever call.

## Consequences

**Positive:**
- Open, published standard with audited open-source reference implementations
- Handles our most complex case (Jones-family group-of-groups inheritance) natively
- Well-understood security properties, audited in production since 2017
- Kid-readable mnemonic words
- Backup recoverable with open-source tooling even if our app disappears

**Negative:**
- BIP-39-style words produce longer per-share writedowns than a raw hex or Codex32 format (20–33 words per share vs ~48 hex chars)
- Extra HKDF step to bridge from SLIP-39 master secret to age Curve25519 identity — documented, deterministic, small code surface
- SLIP-39 Rust ecosystem less mature than SLIP-39 Python/Java ecosystems — may need to port or vendor carefully

## Alternatives rejected

- **Custom Curve25519 Shamir** — no interop, audit burden we don't want
- **Codex32** — newer, mathematically elegant, but less ecosystem support in 2026 than SLIP-39
- **Plain BIP-39 + split into N pieces by position** — not real Shamir, no threshold property

## Implementation notes for Q2

- Audit spike: select SLIP-39 Rust crate (candidate: `slip39`), validate against Trezor test vectors
- If crate is unsatisfactory: port `python-shamir-mnemonic` to Rust in `packages/shared-rust/`
- HKDF bridge: 10-line helper with unit tests verifying deterministic output
- Export UI: user picks split policy (default: adult 2-of-3, kid 1-of-1) and device generates shares
- Restore UI: user enters words from enough shares to meet threshold
