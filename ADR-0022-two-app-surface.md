# ADR-0022: Two-app surface — cold signer + nsec-gated wallet

Date: 2026-08-14
Status: Accepted (2026-08-14, reviewed and accepted by Rajesh)
Amends: ADR-0016 (sub-app architecture), ADR-0017 (nsec signer). Builds on ADR-0020 (key classes). Origin: Rajesh, 2026-08-14 — "I have a bitcoin cold signer that also generates the keys, like SeedSigner. But I need a simple wallet gated by Nostr key identity."

## Context

ADR-0016 split the product into six installable sub-apps on separate origins for
storage isolation. Under the ADR-0020 class model that isolation only earns its cost
at one boundary: Class 1 (the amnesic signer) versus everything else. Family, shared,
savings and inheritance all hold at most Class 2/3 material — the same trust level as
the wallet — so their separate origins add install sprawl and naming confusion
("wallet vs shared vs savings — which one has our money?") without a matching
security gain. Six icons is not explainable to a family; two is.

## Decision

The consumer surface is **exactly two apps**:

### 1. OBS Signer (offline / dedicated device) — unchanged
The amnesic, air-gapped signer per ADR-0010 and the Master spec: generates seeds,
signs PSBTs, stores nothing. SeedSigner-class. Its separate origin/device *is* the
product feature.

### 2. OBS Wallet (online phone) — one app, gated by the nsec
- **Identity = the Nostr key.** First launch creates or imports an nsec, held
  encrypted at rest in the wallet (Class 3; passphrase + biometric session per
  ADR-0008). No email, no password, no server-side account (ADR-0018). NIP-05 gives
  it a human name.
- **The former sub-apps become sections of this one app:** Daily (Class-2 hot,
  capped), Family & chores, Shared, Savings (watch-only), Inheritance (watch-only +
  readiness status). They are views over descriptors + signed events — one install,
  one mental model: *"the app where I see all our money."*
- **Bitcoin side is watch-only for cold funds:** descriptors/xpubs imported from the
  signer by QR; sync via compact block filters (ADR-0021); PSBTs round-trip to the
  signer for anything cold.
- **Everything hangs off the npub:** family membership, event auth, and the E2EE
  vault — backup blobs age-encrypted to the user's own npub (ADR-0004 recipient),
  so restore is "import nsec → your world comes back," and the operator stays blind.

### Identity provenance — fresh key by default
Onboarding **generates a new nsec per person**; importing an existing Nostr identity
is the advanced path, behind a warning. Rationale: the OBS nsec decrypts the vault
(age-to-npub), carries in-family authority, and is the relay-visible routing address —
a social nsec with years of paste-history into clients/extensions has unknown
exposure and links family financial coordination to a public identity. Social and
OBS identities stay **unlinked by default**; public linking is explicit opt-in with
the metadata cost stated. (Amends the spirit of ADR-0004's "own Nostr pubkey"
recipient: that option now means the *OBS* npub, not a reused social one. Composes
with the offline root-identity ceremony for the paranoid tier.)

### Multiple identities — per-context binding, no ambient identity
The vault holds **N identities** (label + color + avatar each). Binding rules:
- Every NIP-46 connection and every internal context binds to **exactly one
  identity at pairing**; there is no ambient "current identity" a request can
  silently inherit. Rebinding is an explicit ceremony.
- Inside the OBS wallet the binding unit is the **family context**: all modules
  (wallet/family/savings/…) of family X sign with the identity that joined X.
  Multi-family membership (ADR-0005) may use a **different identity per family** —
  families stay mutually unlinkable.
- The active identity's badge is visible on every approval prompt and screen; the
  user never has to remember which key is in play.
- **Cross-context guard:** a request from an app bound to identity A that references
  another identity's context (tags, family IDs) escalates to manual approval —
  standing grants never span identities. Rationale: a wrong-identity signature is
  published to relays permanently; misfire is irreversible in public.
- Per-identity vault namespaces and **one identity card per identity** (backup
  ritual below applies to each key, including ones created later).

### Identity disclosure & backup — abstract the string, never the fact
The raw `nsec` never appears in the happy path; onboarding presents "your family
identity" as a named concept. But the key's existence is never hidden, and creation
ends with a mandatory backup ritual: a printed/written **identity card** (QR of the
passphrase-protected ncryptsec + word-phrase form, NIP-06 style) — mirroring the §13
seed-card ritual, one household behavior: keys live on cards. Optional soft path:
the identity backup additionally encrypted to parent/partner npubs, so a family
quorum can restore a lost member (generalizes ADR-0013 modes A/B). An advanced
screen reveals nsec/ncryptsec for export/pairing — extraction is a sovereignty
requirement, not a power feature. **Design constraint driving all this: the vault
cannot back up the nsec (it is encrypted to it) — the recovery artifact must exist
out-of-band, at creation time.**

### The gate rule (honesty requirement)
**The nsec gates the app, never the money.** Funds are controlled solely by Bitcoin
seeds whose backups are physical (Class 1). Losing the nsec costs identity
continuity (rotation + NIP-05 re-publish), not sats. Losing the phone costs nothing
that the nsec import + physical seed cards cannot restore. The two key systems fail
independently, and UX copy must never blur them.

### Effect on ADR-0017 (nsec signer)
The NIP-46 bunker stops being a consumer-visible sixth app. Its engine ships
**embedded in the wallet** as the identity layer (same NIP-49 storage, same session
model). The standalone bunker at `nsec.onbitcoinstandard.com` remains as a
**power-user deployment** of the same code — for people who want their identity key
on a different device/origin than their wallet. ADR-0017's threat-model analysis
(online-by-design, rotation as recovery) is unchanged.

**Open bunker (ecosystem scope).** Because NIP-46 is a standard, the vault serves
**any** Nostr client ("Login with bunker": `bunker://` / `nostrconnect://` strings,
QR pairing) — the nsec.app category, self-hosted and MIT. Conditions of opening up:
- **Per-app permission grants** — each connected app gets an explicit kind whitelist;
  sensitive operations (kind-0 profile edits, DMs/encryption ops, delegation and
  rotation events) always require manual approval; a connections screen lists and
  revokes apps.
- Both deployments (embedded and standalone) expose the same open bunker — the
  embedded one is how a wallet user's identity serves their *other* Nostr apps too.
- Strategic role: the vault is the adoption wedge — a free, self-hostable Nostr key
  vault that installs OBS identity infrastructure before the user ever sees the
  wallet, and the first brick of the one-identity architecture (wallet, MailZ,
  future OBS products on one nsec).

### Effect on ADR-0016 (sub-app architecture)
Sub-apps survive as **internal modules** (code boundaries, lazy-loaded routes) rather
than separate installs/origins. Two origins remain: `wallet.` and `signer.`
(+ optional standalone `nsec.` for power users). The engineering benefits of the
module split (independent development, clear ownership) are kept; the install
sprawl is not.

## Consequences

- Onboarding is one sentence: *"Install the Wallet on your phone; the Signer lives
  on the offline phone in the drawer."*
- One origin for all Class-2/3 surface means one storage domain to protect; the
  compensating control is the Class-1 boundary staying on its own origin/device,
  which is where the irreversible risk lives.
- The web installer / store overlays (ADR-0019) now have exactly two artifacts to
  ship, which simplifies the whole distribution matrix.
- Cost: less origin-level compartmentalization between wallet modules. Accepted:
  those modules already share a trust class, and a compromised wallet was always
  contained by the signer's independent verification (spec §2, §7).
