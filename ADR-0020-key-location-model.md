# ADR-0020: Key-location model — three key classes

Date: 2026-08-14
Status: Accepted (2026-08-14, reviewed and accepted by Rajesh)
Amends: ADR-0004 (portable backup format), ADR-0008 (key unlock model), ADR-0014 (PWA storage primitive). Reconciles the ADR set with the signer-first pivot (2026-05-18 commits) and the Master spec (§2, §5, §9).

## Context

The April ADRs were written before the signer-first pivot. They describe signing keys
stored on the online device (age-encrypted in IndexedDB, unlocked by passphrase or
biometric — ADR-0008, ADR-0014), and age-encrypted key backups in the vault
(ADR-0004). The Master spec (2026-08) says the opposite for the funds that matter:
the Wallet **holds no private key** (§2) and the seed is **never stored
electronically, not even encrypted** (§5).

Both are right — for different keys. The Ark family layer (kid allowances, shared
household funds) cannot round-trip a QR to an offline signer for every small payment;
some on-device key is inherent to that product, and the spec accepts it (§9: the
daily profile is "kept small on purpose"). Every device also signs transport
envelopes (ADR-0003), which requires an identity key. What was never written down is
**which rule governs which key**. This ADR is that record.

## Decision

Every key in the system belongs to exactly one class. Each class has one storage rule.

| Class | What | Where the key lives | Governing rules |
|---|---|---|---|
| **1 · Cold fund keys** | savings, inheritance, cold storage — any balance that would hurt | **Nowhere.** Amnesic signer only: seed entered per session, zeroed on exit. Backups are physical (SeedQR / SSKR cards / metal) — never electronic, never in the vault | spec §2, §3, §5; ADR-0010 |
| **2 · Hot spending keys** | Ark daily / kid / shared-partner balances — deliberately small | On-device, age-encrypted, unlocked per ADR-0008 (passphrase / biometric session). Electronic backup via ADR-0004 permitted | ADR-0004, 0008, 0013, 0014 |
| **3 · Identity keys** | Nostr / device keys signing transport envelopes — control no funds | On-device, age-encrypted (ADR-0008 pattern). The *master* nsec may additionally be held amnesic via the nsec signer | ADR-0003, 0017 |

Class boundaries:

- **A key's class is declared at creation and never silently changes.** Promoting funds
  from hot to cold is a *transaction* (spend to the cold descriptor), not a key change.
- **Hot balances are capped by policy** (per-wallet limit surfaced in UX, family-
  configurable). Breaching the cap prompts a sweep-to-cold flow. The cap is what makes
  "on-device key" an acceptable risk — Class 2 is spending money, not savings.
- **The vault (ADR-0004 recipients) may hold:** Class 2/3 ciphertexts, descriptors,
  blueprints, the encrypt-then-SSKR ciphertext C, and the gated fiduciary package
  (§14). **It may never hold Class 1 seeds in any encoding, encrypted or not.**

## Effect on prior ADRs (status-line pointers added, texts untouched)

- **ADR-0004** — format unchanged; scope narrowed: applies to Class 2/3 keys and
  non-secret blueprints. Its "backup of signing keys" language does not apply to
  Class 1.
- **ADR-0008** — unlock model unchanged for Class 2/3. Its "savings / inheritance
  sovereignty mode" (stored key, session unlock) is **superseded**: savings and
  inheritance are Class 1 and never have a stored key to unlock.
- **ADR-0014** — storage primitive unchanged; the IndexedDB "keys object store"
  holds Class 2/3 ciphertexts only. The wallet PWA remains watch-only with respect
  to Class 1: descriptors, xpubs, PSBTs, transaction history — no cold keys, ever.
- **ADR-0001 / ADR-0013** — unchanged. Kid-mode keys are Class 2 by definition
  (small, capped); graduation cere­monies and per-user custody read naturally onto
  the class model.

## Consequences

- A reviewer reading ADR-0008 beside spec §5 no longer finds a contradiction; the
  class table says which rule wins where.
- The wallet's marketing claim is now precise: *watch-only for savings; spending
  money lives on the phone like cash in a pocket — capped, and never your stack.*
- Cost: one more concept (key class) in docs and UX copy. The UX consequence is a
  visible one-time choice ("spending money" vs "savings") users already understand.
