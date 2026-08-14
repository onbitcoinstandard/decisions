# ADR-0003: Transport-agnostic signed-event envelope; BLE as phase-2 transport

Date: 2026-04-15
Status: Accepted

## Context

Family-wallet actions (chore approval, membership grants, VTXO transfers) could travel over many transports: our HTTPS API, Nostr relays, Bluetooth LE between colocated devices, QR codes, NFC, or a mesh messenger. Tying the protocol to HTTPS now forecloses all of them; designing for multiple transports day-one locks in unnecessary scope.

Bitchat, Briar, and Bridgefy demonstrate that BLE mesh is a real transport pattern — especially valuable for in-person family use cases (kid shows phone to parent, parent taps approve, no internet required).

## Decision

**Canonical representation:** every state-changing action is a signed-event envelope (see ADR-0002 §7). The envelope is transport-agnostic and self-verifying — a client checks the signature, not the channel.

**Day-one transports:**
- **HTTPS** to `api.onbitcoinstandard.com` — signed-event envelopes over POST requests
- **QR code exchange** between devices — single QR for small payloads (chore approvals, invitations, nprofile, simple PSBTs), animated UR for large payloads (multi-sig / inheritance PSBTs)

QR is the canonical in-person transport. It works on every platform including iOS Safari (`getUserMedia` + `jsQR`), requires no pairing ceremony, is visually auditable (you can see what's being exchanged), and lines up with the signer-mode UX users already learn. See "QR size reference" below for payload-strategy details.

**Phase-2 transports (designed-for, not built):**
- **Nostr relays** as an event bus for family events. Target phase 2.

**Explicitly out of scope — Bluetooth LE dropped:**
Originally BLE was listed as a phase-2 transport for in-person approvals and PSBT exchange. Removed because:
- Web Bluetooth is not supported in iOS Safari, so the PWA-first strategy (ADR-0014) can't support BLE on iPhones without a native wrapper
- QR exchange handles every BLE use case we were considering — chore approvals, PSBT signing ceremonies, in-person family approvals — and is universal
- Pairing-free QR is arguably better UX than BLE ("show + scan" vs "find device → pair → permission → PIN")
- QR is visually auditable; BLE payload exchanges are invisible
- Dropping BLE removes an entire permission surface, platform fragmentation, and testing matrix

Mesh / long-range transports (Bitchat-style) remain out of scope as originally specified.

## QR size reference

Single static QR (Version 40, highest density):
- 2,953 bytes binary mode
- 4,296 characters alphanumeric
- 7,089 characters numeric

Practical phone-camera scanning ceiling: ~1,000–1,500 characters. Beyond that, use animated QR.

Animated QR standards we support:
- **UR (Uniform Resources)** — default; broad ecosystem support (Sparrow, Nunchuk, Foundation, BitBox02, Keystone)
- **BBQR** — Coldcard's format; we parse it for interop, don't emit it by default

### Payload → transport strategy

| Payload | Typical size | Transport |
|---------|-------------|-----------|
| Signed-event envelope (chore approval, membership grant) | 0.5–0.8 KB | Single QR |
| nprofile / NIP-05 share | 0.1–0.2 KB | Single QR |
| Family invitation code | 0.2 KB | Single QR |
| Simple wallet descriptor | 0.1–0.8 KB | Single QR |
| Simple PSBT | 0.3–0.6 KB | Single QR |
| Multi-sig / inheritance PSBT | 2–15 KB | Animated UR |
| `.age` backup file (multi-recipient) | 5–30 KB | **File export (not QR)** — share-sheet / download / iCloud / Google Drive |

Full wallet backups ship as files, not QRs. Users already understand "save this file somewhere safe" as a concept; QR for multi-KB payloads is slow and error-prone.

## iOS BLE background constraint

iOS throttles BLE peripheral/central modes heavily in background. Our use case is "parent and kid look at their phones together and tap approve" — foreground by definition — so the constraint doesn't bite. Android has fewer restrictions but the same UX guideline applies.

We do NOT attempt always-on BLE discovery. A user launches the app → discovers nearby family devices for ~30 seconds → completes the exchange → goes back to normal.

## Consequences

**Positive:**
- Adding BLE, Nostr, or QR transport later is additive — no protocol rewrite
- Compelling "works without internet" demo: in-person chore approval on airplane mode
- Fits the sovereignty thesis: server is a convenience, not a dependency for core actions

**Negative:**
- Signed-event schema has to be versioned and carefully designed from the start
- Every state-changing action must be expressible as a self-contained signed envelope (no implicit server state)
- Testing matrix grows once we have multiple transports live

## Implementation notes for Q2

- Define signed-event envelope v1 in `packages/shared/`: `{ kind, created_at, pubkey, content, sig, tags }` — Nostr-event-compatible so Nostr transport is a zero-cost add later
- Build the server API as an envelope-processor: endpoints receive signed envelopes, verify, dispatch to handlers
- Write one reference transport test: send a chore-approval envelope via HTTPS, verify it could be serialized to a Nostr event or QR code without modification
