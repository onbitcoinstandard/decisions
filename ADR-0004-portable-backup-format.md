# ADR-0004: Portable backup format — age with multi-recipient recovery paths

Date: 2026-04-15
Status: Accepted — amended by ADR-0020: narrows this format to hot/identity keys and non-secret blueprints; cold seeds are physical-only per spec §5

## Context

Every user needs a backup they fully control — storable on their own drive, NAS, cloud, USB stick, or Blossom/Nostr relay, without depending on our server. The backup must:

- Be decryptable years into the future by anyone holding the right secret — even if this company no longer exists
- Support multiple independent recovery paths so a single lost artifact doesn't mean total loss
- Be modern enough that security auditors nod rather than wince (ruling out legacy formats)
- Handle the kid→adult transition cleanly (recovery paths that were appropriate for a 10-year-old should dissolve by default when they turn 18)

## Decision

### Format: age (not PGP)

We use [age](https://age-encryption.org/) for the portable backup file. Files end in `.age`, produced with the standard `age` CLI or our embedded library, decryptable on any platform with the open-source `age` binary.

Why not PGP: outdated UX, weak default primitives (RSA legacy), brittle key-server dependencies, complex trust model, and the audit community has moved on. age is what Filippo Valsorda built to replace it — small tool surface, Curve25519 + ChaCha20-Poly1305, multi-recipient native, no "web of trust" baggage.

### Passphrase stretching: Argon2id, not scrypt

age's built-in passphrase recipient uses scrypt. We override with **Argon2id** (higher memory cost, stronger GPU/ASIC resistance) via a small wrapper: the user passphrase runs through Argon2id to produce a 32-byte key, then that key is wrapped as a native age Curve25519 identity. Result: a standard `.age` file that any `age` binary can decrypt, but offline brute-forcing is meaningfully harder than scrypt defaults.

Parameters (tunable, benchmarked on mid-tier mobile at export time):
- `memory = 256 MiB` baseline; degrade gracefully on memory-constrained devices to 128 MiB
- `iterations = 3`
- `parallelism = 4`

### Multi-recipient backup structure

A single `.age` file contains one ciphertext body and N recipient stanzas. Any one valid secret unlocks the body.

Default recipients:

| Recipient | Enabled by default | Purpose |
|-----------|---------------------|---------|
| **Passphrase (Argon2id-wrapped)** | ✅ always | Primary recovery |
| **Own Nostr pubkey** | ✅ if user has one | Seamless restore if nsec still accessible |
| **Family-member pubkey** | ⚪ opt-in, per export | Helper recovery via parent/partner/guardian |
| **Paper Shamir shares** | ⚪ opt-in, per export | Air-gapped physical backup |

The file format is standard age — no custom parsing, no proprietary envelope. The recipient choice is purely which stanzas the export flow includes.

### Paper Shamir shares — on-device only

Paper recipients are generated on the user's device using Shamir's Secret Sharing over a Curve25519 identity. Shares are rendered as printable slips (QR + BIP-39 or Codex32 text fallback). They are never transmitted to the server. The server sees only the resulting age ciphertext — it cannot tell whether "paper" is one of the recipients.

Default paper split: **2-of-3** for adults, **1-of-1** (single paper backup) for kids under a configurable age, because a single printed slip held by a parent is realistic for a 10-year-old; a 3-slip ceremony isn't.

### Graduation ritual (kid → adult) — a role change, not a family exit

**What graduation IS:** a ceremonial key-rotation and role-flag flip for a growing child. The user's family membership persists — you're still your parents' kid at 19, 40, 60. See `docs/FAMILY-LIFECYCLE.md` for how graduation, family membership, and inheritance interact.

**What graduation is NOT:** removal from the family, and not the trigger for inheritance. Inheritance is a separate wallet whose keys are only activated by parent death or timelock expiry — typically decades after graduation. See ADR-0005 for the data model separating membership from inheritance-key-holder records.

On the kid's configured graduation date (default 18, parent-adjustable during family setup), the app prompts the ritual:

1. Generate a fresh signing key on the device
2. Sweep all VTXOs from the old Arkade wallet to the new key
3. Re-export the portable backup file — **parent recipient dropped by default** (the parent-as-safety-net relationship dissolves; the graduated adult can opt back in)
4. New paper Shamir split (adult defaults: 2-of-3)
5. Old key archived for monitoring any late-arriving funds; archived backup retained as cold record
6. New passphrase prompt — the adult picks their own, independent of any the parent may have helped set up
7. Family-membership role flag flips `child → adult_member` in this family (does not affect memberships in other families)
8. Optionally: parent grants the graduated adult one of the inheritance wallet keys (bitcoin.design Jones family pattern — Christina receives her inheritance key at ~24)

The adult can optionally re-add the parent as a backup recipient. But the default is: parent-as-safety-net dissolves. The app presents this as a deliberate, ceremonial transition — not an auto-silent key rotation.

Mechanically steps 1–2 are the same "making changes" flow used for any key replacement (see bitcoin.design `/guide/inheritance-wallet/making-changes/`). Graduation is the ceremonialized UX-layer trigger for it.

### Portable restore

Restoring from the backup file is transport-agnostic. User can:
- Import the `.age` file in our app on a new device → app prompts for passphrase or accepts Nostr nsec / paper shares
- Decrypt the file outside our app entirely — `age -d backup.age > seed.json` with the `age` CLI — then inspect / import elsewhere
- Keep the encrypted file on iCloud, Google Drive, Nextcloud, Umbrel, Nostr Blossom, a USB stick, or printed as a QR code for the ciphertext itself

Any future operator, or no operator at all, can read it. The file does not depend on our server being alive.

## Consequences

**Positive:**
- Users have a backup artifact they fully own, in a standard format, with multiple independent recovery paths
- Passing the "ten-year test" — the file is decryptable in 2036 with just the `age` binary + the user's secret
- Parent-as-safety-net is expressible without granting custody — the parent is a recipient, not a key holder
- Graduation is a first-class product moment rather than an afterthought
- Paper backups give us the Glacier-Protocol-grade air-gapped option without bringing Glacier's ceremony complexity to every user

**Negative:**
- Argon2id Node bindings need audit; we'll vendor a specific version
- Shamir-over-Curve25519 is less battle-tested than Shamir-over-BIP39; need to pick and audit a library (consider SLIP-39 as an alternative that's used in production by Trezor)
- Graduation ritual requires an on-chain or VTXO sweep — small but nonzero cost at the transition. Must be communicated during family setup ("your kid's graduation will cost ~X sats in fees")
- Multi-recipient export files leak the *count* of recipients (though not their identities). Privacy-conscious users can export single-recipient files

## Implementation notes for Q2

Add to the `docs/YEARLY-PLAN.md` Q2 checklist:
- `age-encryption.js` or WASM port vendored and version-pinned
- Argon2id wrapper implemented with tuned mobile params
- Multi-recipient export UI (the checkbox screen described in this ADR)
- Portable restore flow (import `.age` file → prompt for secret)
- Shamir library decision: SLIP-39 (Trezor-compatible, BIP-39 mnemonic shares) vs. custom Curve25519 Shamir — pick one, write a micro-ADR if not immediately obvious
- Graduation ritual is a Q3/Q4 feature (can't graduate until we have family-graph + kid keys), but the backup file schema must support the "rotate and re-export" flow from day one

## Alternatives rejected

- **PGP / GPG** — dated, poor UX, ecosystem drift. Rejected above.
- **Custom AES-GCM envelope** — gives us nothing age doesn't and removes the "decryptable with a standard tool" property.
- **NIP-44 (Nostr encryption)** — excellent for messaging, not designed as a file-at-rest format with multi-recipient support. Use alongside for event transport (ADR-0003), not instead.
- **Server-side backup with key escrow** — violates operator-blind posture (ADR-0001).
