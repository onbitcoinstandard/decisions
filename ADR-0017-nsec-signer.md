# ADR-0017: nsec signer sub-app (`nsec.onbitcoinstandard.com`)

Date: 2026-04-16
Status: Accepted

## Context

Every sub-app in our ecosystem needs to sign Nostr events — chore approvals, membership grants, NIP-05 claims, backup-upload events, auth challenges. Asking the user to enter their nsec in each sub-app separately would be:

- **Bad UX.** Re-enter nsec every time they switch between `wallet`, `family`, `savings`, etc.
- **Bad security.** Each app origin holds the nsec, multiplying attack surface.
- **Architecturally wrong.** Violates the "one Nostr identity per person" principle from ADR-0005.

Three existing patterns address this in the Nostr ecosystem:

1. **NIP-07 browser extension** (Alby, nos2x) — `window.nostr` global; extension holds nsec; apps request signatures. Works well but requires a browser extension, which is unavailable on mobile and uncommon among our target audience.

2. **NIP-46 remote bunker signer** — a dedicated signer app (typically on another device or a separate browser origin) holds the nsec; apps request signatures via encrypted messages over Nostr relays; user approves each request in the signer UI.

3. **Bare nsec entry per app** — what we'd be doing without a signer. Worst option; rejected.

nsec.app has proven the NIP-46 pattern works in production for years. We adopt the same architecture, host it ourselves, and make it a first-class sub-app in our ecosystem.

## Decision

**Ship `nsec.onbitcoinstandard.com` as a sixth sub-app.** Standalone Nostr identity holder + NIP-46 bunker for cross-app signing.

### Core properties

| Property | Value |
|----------|-------|
| Storage | Persistent NIP-49 ncryptsec blob in IndexedDB |
| Encryption | Argon2id-derived key from user passphrase (same params as ADR-0008) |
| Memory model | nsec decrypted into memory on unlock; cached for configurable session timeout; zeroed on lock/close |
| Session timeout | Default 15 min inactivity; user-configurable 5 min to 8 hours; biometric (passkey) extends without passphrase re-entry |
| Network | Online by design — must talk to Nostr relays for NIP-46 messages |
| Key rotation | Supported — user can generate new nsec, re-publish NIP-05, retire old nsec |

### Why stored persistently (unlike bitcoin cold signer)

The bitcoin cold signer (`signer.onbitcoinstandard.com`, ADR-0010) is **stateless + airplane-gated** because:
- Low-frequency usage (rare, high-value transactions)
- Bet-the-farm threat model (key loss = funds gone)
- Airplane-mode gate is practical given low signing frequency

The nsec signer has the opposite characteristics:
- **High-frequency usage** (every chore approval, every social post, every auth challenge — potentially dozens per day)
- **Identity compromise ≠ funds loss** (painful, but recoverable via rotation)
- **Airplane-mode impractical** — NIP-46 by definition needs network

These are genuinely different threat models. Persistent ncryptsec storage is correct here despite being wrong for the bitcoin cold signer.

### Session flow

**First-time use:**

1. User visits `nsec.onbitcoinstandard.com` on a device
2. Chooses: *"Generate a new Nostr identity"* or *"Import existing"* (nsec paste, SeedQR-style scan, or NIP-49 import)
3. Sets passphrase (Argon2id tuned for mobile per ADR-0008)
4. ncryptsec written to IndexedDB; plaintext nsec zeroed from memory immediately after
5. Optionally claim NIP-05 alias on `onbitcoinstandard.com` (wildcard, per ADR-0015)
6. Optionally enroll passkey for biometric unlock on this device

**Subsequent use (signing flow):**

1. User (or a delegating sub-app) invokes nsec signer
2. Passphrase or passkey prompt
3. ncryptsec decrypted; plaintext nsec in memory for the session
4. Signer UI shows pending requests (if any) — from delegating apps via NIP-46
5. User approves signatures individually, or auto-approves based on pre-set rules
6. Session auto-locks after inactivity timeout; memory zeroed
7. Any further signing requires re-unlock

### NIP-46 bunker for cross-app signing

Every other sub-app (family, shared, savings, inheritance) needs to sign Nostr events. They delegate to `nsec.onbitcoinstandard.com` via the NIP-46 remote-signer protocol:

```
┌───────────────────────────────┐         ┌──────────────────────────────┐
│ family.onbitcoinstandard.com  │  relay  │ nsec.onbitcoinstandard.com   │
│ (needs to sign a chore event) │  layer  │ (holds nsec, signs on approval)│
└──────┬────────────────────────┘ ◄─────► └──────┬───────────────────────┘
       │                                         │
       │ NIP-46 connection request               │
       │ → relay.onbitcoinstandard.com (default) │
       │                                         │
       │                                         │ User approves in UI
       │                                         │ (or rule auto-approves)
       │ signed event returned                   │
       │ ← relay.onbitcoinstandard.com           │
       ▼                                         ▼
```

**Permission model:**

Per-app trust rules stored in the nsec signer:
- **First request from a new app:** explicit user approval required
- **Trust options after first approval:**
  - Ask for every request (default for new apps)
  - Auto-approve specific event kinds (e.g., "auto-approve kind:1 from `family.onbitcoinstandard.com`, still ask for kind:4 DMs")
  - Auto-approve all (power-user mode; warning on activation)
- **Audit log:** every signature with timestamp, requesting app, event kind, content hash (not plaintext of private DMs)
- **Revoke trust:** user can revoke trust for any app at any time

**Relay selection:**

NIP-46 requires a relay to pass messages. The relay sees metadata (who talks to whom, what event kinds) but not encrypted content.

- Default relay: `relay.onbitcoinstandard.com` (our hosted relay, deployable later once we need it)
- User can configure any Nostr relay they trust (wss://relay.damus.io, wss://nos.lol, etc.)
- For extra paranoia: user can run their own relay and point nsec signer at it

### Fallback: existing NIP-07 and bunker options

Our apps should not require users to use OUR nsec signer. Integration priority:

1. **First:** check for `window.nostr` (NIP-07 browser extension — Alby, nos2x, Flamingo)
2. **Then:** check for existing NIP-46 bunker URI in user preferences (user may already use nsec.app or similar)
3. **Default:** prompt to install our `nsec.onbitcoinstandard.com` if neither exists

A user with an Alby extension never sees our nsec signer. A user with nsec.app continues using nsec.app. A user with nothing gets a turnkey option at our domain.

### What we do NOT do

- **Export nsec to the user's clipboard** — never. Export only to:
  - NIP-49 ncryptsec for backup (passphrase-encrypted, safe)
  - Printed SeedQR / mnemonic for paper backup
- **Sync nsec across devices through our server** — server never sees plaintext. If user wants the same nsec on multiple devices, they import the ncryptsec file on each device.
- **Auto-approve everything** by default — explicit user action required for that mode.
- **Log request content of encrypted events** (DMs, encrypted DMs) — audit log records event kind and hash only, not decrypted content.

### Key rotation path

If a user's nsec is compromised or they want to rotate for any reason:

1. Generate new nsec in nsec signer
2. Publish a NIP-26 delegation event signing the transition (optional — not all clients honor this)
3. Update NIP-05 alias(es) to point to new pubkey (signed update from OLD pubkey per ADR-0015 rules)
4. Notify family graph members via signed membership-update event
5. Re-key each sub-app that held wallet state under the old pubkey (wallet-level rotation; for our sub-apps this is the existing graduation-ritual-style flow from ADR-0004)
6. Archive old nsec for monitoring late-arriving events

This is painful — identity rotation is never easy in Nostr. We make it possible but discourage it unless necessary.

## Consequences

**Positive:**
- Single-sign-on across all our sub-apps — user enters passphrase once per session, signs freely
- Standard NIP-46 protocol means third-party Nostr clients (Damus, Amethyst, Primal) can also use our signer if the user wants
- Familiar ecosystem pattern (nsec.app has validated it for years)
- Per-app permission rules give fine-grained control without UX hell
- Offering turnkey option + fallback to existing tools = broad audience

**Negative:**
- Persistent ncryptsec = more attack surface than stateless
- NIP-46 relay traffic leaks metadata (who's asking for signatures) — mitigated by user-chosen relay
- Identity compromise is possible and painful; recovery path exists but isn't silky
- Adds a sixth sub-app; scheduling pressure on Q2/Q3
- Dependency on NIP-46 spec compliance — Nostr protocol still evolving

**Explicitly accepted risks:**
- Device compromise with decrypted session → attacker can sign until timeout. Mitigation: short default timeouts (15 min), explicit approval for high-risk event kinds (DMs, zaps, publishing to wide relays)
- Persistent ncryptsec attackable offline via passphrase brute-force if device is stolen. Mitigation: Argon2id tuned high (256 MiB / 3 iter / 4 parallel); user education about passphrase strength

## Scheduling

**Ships: late Q2 2026 or early Q3 2026.** Specifically:

- Q2 primary: bitcoin cold signer (`signer.onbitcoinstandard.com`) — non-negotiable
- Q2 secondary if capacity allows: nsec signer basic features (ncryptsec storage, passphrase unlock, own-UI signing for own events)
- Q3 requirement: NIP-46 bunker functional before `family` or `savings` sub-app ships (those delegate signing to it)

Worst case: nsec signer slips to Q3 start; `family`/`savings` sub-app waits a week for it to land. Acceptable.

## Implementation notes

- `apps/nsec/` in the monorepo (alongside `apps/signer/`, `apps/wallet/`, etc.)
- Shared `packages/identity/` module gets Nostr + NIP-05 + NIP-46 helpers that both nsec signer and delegating apps import
- NIP-46 library: likely a hand-rolled implementation on top of `nostr-tools` (the npm package); alternatives exist but adopting a well-maintained one reduces our audit surface
- Relay: `relay.onbitcoinstandard.com` deployment deferred until nsec signer ships; can use public relays (wss://relay.damus.io) in interim
- Session-timeout UI: visible countdown, "stay unlocked" button with biometric confirmation
- Permission-rule UI: per-app cards showing granted permissions + revoke button

## References

- NIP-07 — Web Browser Capability for Bitcoin Wallets: https://github.com/nostr-protocol/nips/blob/master/07.md
- NIP-46 — Nostr Connect (Remote Signer): https://github.com/nostr-protocol/nips/blob/master/46.md
- NIP-49 — Private Key Encryption (ncryptsec): https://github.com/nostr-protocol/nips/blob/master/49.md
- nsec.app — reference implementation: https://nsec.app
- ADR-0005 — multi-family membership (one pubkey per user)
- ADR-0008 — key unlock model (passphrase + Argon2id)
- ADR-0010 — bitcoin cold signer (stateless; contrast with persistent nsec signer)
- ADR-0016 — sub-app architecture (this becomes sub-app #6)
