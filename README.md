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
| [0001](ADR-0001-custody-posture.md) | Custody posture — self-custodial, operator-blind | 2026-04-15 | Accepted |
| [0002](ADR-0002-security-baseline.md) | Security baseline (age, Noise, Argon2id, BDK…) | 2026-04-15 | Accepted |
| [0003](ADR-0003-transport-layer.md) | Transport layer — signed-event envelopes | 2026-04-15 | Accepted |
| [0004](ADR-0004-portable-backup-format.md) | Portable backup format — age multi-recipient | 2026-04-15 | Accepted |
| [0005](ADR-0005-multi-family-membership.md) | Multi-family membership | 2026-04-15 | Accepted |
| [0006](ADR-0006-tauri-stack.md) | Tauri 2.0 shell | 2026-04-15 | ⛔ Superseded by 0014 |
| [0007](ADR-0007-slip39-shamir.md) | SLIP-39 for Shamir sharing | 2026-04-15 | ⚠️ Superseded in part — standard changed to **SSKR** (spec §5/§12) |
| [0008](ADR-0008-key-unlock-model.md) | Key unlock model | 2026-04-15 | Accepted (amends 0002 §2) |
| [0009](ADR-0009-ux-for-non-tech-users.md) | UX for non-technical users | 2026-04-15 | Accepted |
| [0010](ADR-0010-signer-architecture.md) | Signer architecture — stateless, airplane-gated | 2026-04-15 (am. 04-16) | Accepted |
| [0011](ADR-0011-licensing.md) | Licensing — MIT client · AGPL-3.0 server · CC-BY-SA docs | 2026-04-15 | Accepted |
| [0012](ADR-0012-plausible-deniability.md) | Plausible deniability | 2026-04-15 | Accepted |
| [0013](ADR-0013-kid-age-modes.md) | Kid age modes | 2026-04-15 | Accepted (amends 0001) |
| [0014](ADR-0014-pwa-first-shell.md) | PWA-first shell | 2026-04-15 | Accepted (supersedes 0006; store-abstinence amended by 0019) |
| [0015](ADR-0015-identity-and-domains.md) | Identity & domains — NIP-05, subdomain-per-namespace | 2026-04-15 (am. 04-16) | Accepted |
| [0016](ADR-0016-sub-app-architecture.md) | Sub-app architecture (6 sub-apps) | 2026-04-16 | Accepted (am. — daily Arkade wallet dropped) |
| [0017](ADR-0017-nsec-signer.md) | nsec signer sub-app | 2026-04-16 | Accepted |
| [0018](ADR-0018-no-email-accounts.md) | No email accounts | 2026-04-16 | Accepted |
| [0019](ADR-0019-distribution-and-device-tiers.md) | Distribution overlays & signer device tiers | 2026-08-14 | Accepted (amends 0014; resolves spec §12 #3) |

## Current state in one paragraph

Self-custodial and operator-blind (0001, 0002): every family member holds their own
keys; servers only ever see ciphertext. The product is a web codebase shipped
PWA-canonical (0014) with store/APK builds as distribution overlays and a three-tier
signer-device ladder (0019), split into sub-apps (0016) including an amnesic,
airplane-gated Bitcoin signer (0010) and an nsec signer (0017). Identity is a Nostr
pubkey with NIP-05 subdomain-per-namespace (0015, 0018 — no email accounts). Backups
use age multi-recipient envelopes (0004) with SSKR Shamir shares (0007-as-amended),
unlocked per 0008, with plausible deniability (0012). Kids get age-graduated modes
(0013). Client code is MIT, server AGPL-3.0, docs CC-BY-SA-4.0 (0011).

## License

Documentation — [CC-BY-SA-4.0](LICENSE), per ADR-0011.
