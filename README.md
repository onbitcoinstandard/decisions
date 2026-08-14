# OnBitcoinStandard — Architecture Decision Records

The design of the OBS family wallet & signer, decision by decision. Published before
the code ("specs first, then code") so the architecture can be scrutinized on its
reasoning, not its marketing.

## What an ADR is

An Architecture Decision Record captures **one** significant decision: the context that
forced it, the options weighed, what was decided, and the consequences accepted. ADRs
are **append-only** — a decision is never edited into something else. When a decision
changes, a *new* ADR supersedes or amends the old one, and only the `Status:` line of
the old record is updated to point forward. The chain of reversals is the point: it
shows the design was argued, not assumed.

This repo is the authoritative numbering. Proposals discussed in working sessions are
**not** ADRs until accepted and merged here.

## Index

| # | Decision | Date | Status |
|---|---|---|---|
| [0001](ADR-0001-custody-posture.md) | Custody posture — self-custodial, operator-blind, per-user keys | 2026-04-15 | Accepted |
| [0002](ADR-0002-security-baseline.md) | Security baseline (age, Noise, Argon2id, BDK…) | 2026-04-15 | Accepted |
| [0003](ADR-0003-transport-layer.md) | Transport layer — signed-event envelopes | 2026-04-15 | Accepted |
| [0004](ADR-0004-portable-backup-format.md) | Portable backup format — age multi-recipient | 2026-04-15 | Accepted · scope narrowed by 0020 (hot/identity keys + blueprints only) |
| [0005](ADR-0005-multi-family-membership.md) | Multi-family membership | 2026-04-15 | Accepted |
| [0006](ADR-0006-tauri-stack.md) | Tauri 2.0 shell | 2026-04-15 | ⛔ Superseded by 0014 |
| [0007](ADR-0007-slip39-shamir.md) | SLIP-39 for Shamir sharing | 2026-04-15 | ⚠️ Superseded in part — standard changed to **SSKR** (spec §5/§12) |
| [0008](ADR-0008-key-unlock-model.md) | Key unlock model | 2026-04-15 | Accepted · scoped by 0020 (hot/identity only; stored-key savings mode superseded) |
| [0009](ADR-0009-ux-for-non-tech-users.md) | UX for non-technical users | 2026-04-15 | Accepted |
| [0010](ADR-0010-signer-architecture.md) | Signer architecture — stateless, airplane-gated | 2026-04-15 (am. 04-16) | Accepted |
| [0011](ADR-0011-licensing.md) | Licensing — MIT client · AGPL-3.0 server · CC-BY-SA docs | 2026-04-15 | Accepted |
| [0012](ADR-0012-plausible-deniability.md) | Plausible deniability | 2026-04-15 | Accepted |
| [0013](ADR-0013-kid-age-modes.md) | Kid age modes | 2026-04-15 | Accepted (amends 0001) |
| [0014](ADR-0014-pwa-first-shell.md) | PWA-first shell | 2026-04-15 | Accepted (supersedes 0006) · amended by 0019 (store overlays) & 0020 (keys-store scope) |
| [0015](ADR-0015-identity-and-domains.md) | Identity & domains — NIP-05, subdomain-per-namespace | 2026-04-15 (am. 04-16) | Accepted |
| [0016](ADR-0016-sub-app-architecture.md) | Sub-app architecture | 2026-04-16 | Accepted · consolidated by 0022 (sub-apps → internal modules) |
| [0017](ADR-0017-nsec-signer.md) | nsec signer (NIP-46 bunker) | 2026-04-16 | Accepted · amended by 0022 (embedded in wallet; standalone = power-user) |
| [0018](ADR-0018-no-email-accounts.md) | No email accounts | 2026-04-16 | Accepted |
| [0019](ADR-0019-distribution-and-device-tiers.md) | Distribution overlays & signer device tiers | 2026-08-14 | Accepted (amends 0014; resolves spec §12 #3) |
| [0020](ADR-0020-key-location-model.md) | Key-location model — three key classes | 2026-08-14 | Accepted (amends 0004, 0008, 0014) |
| [0021](ADR-0021-operator-blind-infra.md) | Operator-blind infrastructure & chain access | 2026-08-14 | Accepted (extends 0001) |
| [0022](ADR-0022-two-app-surface.md) | Two-app surface — cold signer + nsec-gated wallet | 2026-08-14 | Accepted (amends 0016, 0017) |

## Current state in one paragraph

Self-custodial and operator-blind in both senses — the operator can neither spend
(0001) nor see (0021): every family member holds their own keys, servers hold only
ciphertext, wallets sync by compact block filters, and no OBS-run indexer ever
observes user addresses. Keys live in three classes (0020): cold fund keys exist
nowhere but the amnesic, airplane-gated signer (0010) with physical-only backups;
capped hot spending keys and identity keys live on-device, encrypted. The consumer
surface is two apps (0022): the offline Signer, and one online Wallet gated by a
fresh-by-default Nostr identity (0018 — no email) whose embedded vault is an open
NIP-46 bunker with per-app permissions and per-context identity binding. The web
codebase ships PWA-canonical with store/APK overlays and a three-tier signer-device
ladder (0014, 0019). Backups use age multi-recipient envelopes (0004) with SSKR
shares (0007-as-amended), plausible deniability (0012), kid age modes (0013), and
NIP-05 identity naming (0015). Client code is MIT, server AGPL-3.0, docs
CC-BY-SA-4.0 (0011).

## License

Documentation — [CC-BY-SA-4.0](LICENSE), per ADR-0011.
