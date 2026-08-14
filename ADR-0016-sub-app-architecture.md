# ADR-0016: Sub-app architecture — one focused PWA per wallet concept

Date: 2026-04-16
Status: Accepted (supplements ADR-0014 PWA-first shell) — amended 2026-04-16 to drop daily Arkade wallet sub-app; see § Amendment below

## Amendment 2026-04-16 — daily Arkade wallet dropped; signer-first priority

Original decision listed six sub-apps including `wallet.onbitcoinstandard.com` as a daily single-user Arkade wallet. Amended to **five sub-apps**: daily Arkade functionality is dropped from our scope entirely because **[arkade.money](https://arkade.money) already ships a workable Arkade wallet prototype** and we have no reason to duplicate their work. We interop with arkade.money at the PSBT / coordinator protocol level — users who want a daily Arkade wallet use theirs; our sub-apps handle the family-lifecycle + savings + inheritance + signer functionality that arkade.money doesn't.

**Revised sub-app list (6 — updated again 2026-04-16):**

```
signer.onbitcoinstandard.com       — Bitcoin cold signer, stateless + airplane-gated (Q2 2026 — first)
nsec.onbitcoinstandard.com         — Nostr identity + NIP-46 bunker (late Q2 / early Q3 2026)
family.onbitcoinstandard.com       — Family graph + chores + kid modes (post-research, Q3)
shared.onbitcoinstandard.com       — 1-of-2 partner wallet (Q4 2026)
savings.onbitcoinstandard.com      — 2-of-3 family savings multisig (post-research, Q3 or Q4)
inheritance.onbitcoinstandard.com  — Timelocked inheritance + heirs (Q1 2027)
```

Daily Arkade wallet is still out of scope (arkade.money covers it). The sub-app count went back up to 6 because we added `nsec.onbitcoinstandard.com` per ADR-0017 — this is the Nostr-identity / NIP-46 bunker sub-app, complementary to but distinct from the bitcoin cold signer.

**Why both a bitcoin signer AND a Nostr signer:**

They have opposite operational characteristics despite both being "signers":

| Property | `signer.` (bitcoin cold) | `nsec.` (Nostr identity) |
|----------|---------------------------|---------------------------|
| Stores keys | No — stateless | Yes — NIP-49 ncryptsec in IndexedDB |
| Network posture | Airplane-mode gated | Online (NIP-46 requires relays) |
| Signing frequency | Rare (savings, inheritance) | Frequent (every chore approval, every post) |
| Threat of key loss | Funds lost | Identity lost (recoverable via rotation) |

Conflating them would break both. See ADR-0017 for full rationale.

**Revised priority order:**

1. **Q2 2026 — Signer.** Stateless airplane-gated cold signer (per ADR-0010 amendment). Works with ANY coordinator (Sparrow / Nunchuk / Specter / arkade.money / our own future sub-apps). This is the first community deliverable and it's independent of any family features.

2. **Discovery questionnaire runs in parallel** during Q2. By the time signer ships, we have real market signal about whether families want chores-first or savings-first.

3. **Post-research — Family OR Savings** (whichever has stronger demand signal from the questionnaire). Ship one in Q3, the other in Q4.

4. **Q1 2027 — Inheritance.** Still at the end, since it's the most complex and depends on savings/family landing first.

**Decision rule for post-research direction:**

- Strong family/chores signal → build `family` next
- Strong savings signal → build `savings` next
- Both equal → build `savings` first (simpler architecturally — no multi-user UX, no kid modes, just 2-of-3 multisig + BDK)
- Neither strong → stay in signer-only mode longer; revisit market fit

**Relationship to arkade.money:**

They are **complementary, not competitive**. arkade.money builds the daily Arkade wallet. We build the family coordination, savings, inheritance, and cold-signer layers that plug into Arkade at the protocol level. Our strategic bets table (PLAN.md §0) now reflects "arkade.money's bet, not ours" for Arkade protocol success — we benefit if it succeeds, but our own product value isn't gated on Arkade specifically.

The rest of the ADR below describes the original six-sub-app plan; sections referring to `wallet.onbitcoinstandard.com` as a standalone daily wallet are superseded by this amendment.

---

## Original (pre-amendment) content below



## Context

As scope expanded — daily wallet + shared partner wallet + 3 kid modes + savings multisig + inheritance + graduation + signer + plausible deniability + family graph + inheritance key holders + chores + approvals — the design was heading toward one monolithic PWA doing everything. This creates several problems:

1. **Q2 "MVP" scope is unclear** — too many features compete for the first release, so nothing ships
2. **Audit scope is unbounded** — reviewing security of "the wallet" means reviewing every line including unreleased features
3. **UX personas conflict** — kid chore approval lives in the same app as inheritance setup; opposite ends of the user spectrum
4. **Complexity compounds** — a bug in the inheritance feature potentially affects daily-wallet users
5. **One install for everything** — grandma who only wants to receive birthday sats has to install the full wallet-savings-inheritance-kid bundle

A sub-app architecture fixes all of these. Each wallet concept becomes its own PWA at its own subdomain, shipped independently, audited independently, installed optionally.

## Decision

**Six focused sub-apps, one per wallet concept, each its own PWA at its own subdomain. Users install only what they need.**

```
wallet.onbitcoinstandard.com       — Daily single-user Arkade wallet
family.onbitcoinstandard.com       — Family graph + chores + kid modes (A/B/C)
shared.onbitcoinstandard.com       — 1-of-2 partner wallet
savings.onbitcoinstandard.com      — 2-of-3 family savings multisig
inheritance.onbitcoinstandard.com  — Timelocked inheritance + heir coordination
signer.onbitcoinstandard.com       — Cold signer (offline-first, SeedSigner-parity)
```

Plus the existing support surfaces:

```
onbitcoinstandard.com              — NIP-05 namespace host (wildcard) + OBS content site
api.onbitcoinstandard.com          — Signed-event API + backup blobs + ASP coordinator
media.onbitcoinstandard.com        — Blossom (existing)
```

### Per sub-app scope

**`wallet.onbitcoinstandard.com` — Daily (Q2 2026)**

- Single-user Arkade wallet
- Generate own Nostr keypair, age-encrypted backup (ADR-0004, ADR-0008)
- Receive, send, activity history, contacts, settings
- NIP-05 claim (integrates with identity host)
- This is Q2's shippable product. Adults use it alone. Launch criterion is "Rajesh runs his testnet daily wallet here for 2 weeks."

**`signer.onbitcoinstandard.com` — Signer (Q2 2026 basic, Q1-2027 SeedSigner-parity)**

- Offline-first PWA (service worker refuses network fetches)
- Seed entry (BIP-39 words, SLIP-39 shares, SeedQR, dice entropy)
- PSBT signing via QR (UR animated for large PSBTs)
- Xpub export, address verification, message signing
- Ships Q2 in basic form (sign PSBTs for wallet); full SeedSigner parity in Q1-2027 when savings/inheritance need it
- Works standalone — any Bitcoin coordinator can use it as a cold signer

**`family.onbitcoinstandard.com` — Family + kids (Q3 2026)**

- Family graph (parents, partners, kids)
- Invitations, membership approvals (signed events)
- Kid modes A/B/C (ADR-0013) with three-mode-specific UX
- Chores: creation, completion, approval, auto-payout
- Printable chore charts for Mode A
- Picture-passphrase for Mode B kids
- Graduation ritual (lives here since it's family-lifecycle)

**`shared.onbitcoinstandard.com` — Couples (Q4 2026)**

- 1-of-2 partner wallet (either spends, both see)
- Partner invitation + cosigner onboarding
- Shared wallet descriptor management
- Archive flow for wallets no longer in use
- Small surface — most work is in the existing BDK/descriptor plumbing

**`savings.onbitcoinstandard.com` — Family savings (Q1 2027)**

- 2-of-3 multisig, optional ASP-assist key
- Auto-sign under threshold, manual for above
- Savings-specific UX (less frequent, higher-value flows)
- Large-amount send guards (cooldowns, double-confirm, biometric per signer)
- Recovery kit export (for bitcoin.design Jones-family-style distribution)

**`inheritance.onbitcoinstandard.com` — Inheritance (Q1 2027)**

- Timelocked multisig (primary + recovery + heir key sets)
- Heir onboarding (remote XPUB collection, asynchronous setup)
- Attorney onboarding (inheritance-key-holder role without family membership)
- Maintenance protocol reminders (6-month key check, inspired by Glacier)
- Succession flow for when timelocks expire
- View-only mode with PIN protection
- Plausible-deniability mode (ADR-0012) — though this may also appear in other wallets
- Entirely separate user set from the daily wallet — lawyers, executors, older adult children

### What every sub-app shares

**Identity:**
- User's single master Nostr pubkey (with optional NIP-05 aliases per ADR-0015)
- Each sub-app on first install derives a sub-wallet key via BIP-32 account separation from the user's master seed
- Sub-wallets are linked to the master pubkey via signed `wallet.link` events published to our API
- Family graph sees all of a user's sub-wallets under their master pubkey

**Transport:**
- Signed-event envelopes (ADR-0003)
- HTTPS to `api.onbitcoinstandard.com`
- QR exchange (single or animated UR) for cross-device

**Crypto primitives:**
- Shared `packages/crypto/` module with age, argon2, SLIP-39, noble curves
- Each sub-app bundles what it needs; common code deduplicated at build time

**Backup:**
- Portable `.age` files with multi-recipient unlock (ADR-0004)
- User can back up multiple sub-wallets into a single `.age` file if they wish (age multi-recipient handles this natively)

### What each sub-app isolates

- **Browser origin** = IndexedDB storage isolation (browser-enforced firewall)
- **Per-sub-app signing key** — derived from master seed via BIP-32 account path (`m/84'/0'/<account>'`) so a compromise of one sub-app's key doesn't expose others
- **Service worker lifecycle** — each sub-app updates on its own cadence
- **Audit scope** — `wallet` can be audited + shipped while `inheritance` is still in draft
- **Persona-specific UX** — daily-wallet UX is NOT compromised to accommodate inheritance flows

### BIP-32 account separation

Account-level derivation paths per sub-app:

```
m/84'/0'/0'/...    →  wallet (daily Arkade)
m/84'/0'/1'/...    →  shared (partner 1-of-2)
m/84'/0'/2'/...    →  savings (2-of-3 multisig)
m/84'/0'/3'/...    →  inheritance (timelocked multisig)
```

Kid wallets in family-app use kid-specific paths under parent's master (Mode A) or derive freshly on kid's device (Mode B/C). Signer app has no derivation constraint — it just signs whatever PSBT it's given.

### Revised Q2–Q1-2027 shipping plan

| Quarter | Sub-apps |
|---------|----------|
| Q2 2026 | `wallet` (full), `signer` (basic — sign PSBTs from `wallet`) |
| Q3 2026 | `family` (chores + kid modes) |
| Q4 2026 | `shared` (partner 1-of-2) |
| Q1 2027 | `savings` (2-of-3), `inheritance` (timelocked), `signer` extended to SeedSigner parity |

**Q2 becomes a shippable product**, not a pilot: a working daily bitcoin wallet with cold-signing support. That alone is a legitimate launch.

### Deploy architecture

All sub-apps deploy to Cloudflare Pages as independent projects:

```
wallet-obs-family        → wallet.onbitcoinstandard.com
signer-obs-family        → signer.onbitcoinstandard.com
family-obs-family        → family.onbitcoinstandard.com
shared-obs-family        → shared.onbitcoinstandard.com
savings-obs-family       → savings.onbitcoinstandard.com
inheritance-obs-family   → inheritance.onbitcoinstandard.com
```

Each has its own build pipeline, own version, own release cadence. The monorepo structure becomes:

```
obs-family-wallet/
├── apps/
│   ├── wallet/          — Vite + React PWA
│   ├── signer/          — Vite + React PWA
│   ├── family/          — Vite + React PWA
│   ├── shared/          — Vite + React PWA
│   ├── savings/         — Vite + React PWA
│   ├── inheritance/     — Vite + React PWA
│   └── server/          — Node + Fastify (unchanged)
├── packages/
│   ├── crypto/          — age, argon2, slip39, noble curves (shared)
│   ├── events/          — signed-event envelope types (shared)
│   ├── identity/        — NIP-05 + Nostr helpers (shared)
│   └── ui/              — shadcn/ui customizations (shared)
└── infra/arkade/        — ASP (unchanged)
```

## Consequences

**Positive:**
- Q2 actually ships a shippable product, not a pilot
- Each sub-app is a small, auditable, independently deployable artifact
- User installs only what they need — grandmother gets `wallet` alone, family parents add `family`, wealth holders add `savings` + `inheritance`
- Origin isolation is a real security boundary, not just a principle
- Bugs in one sub-app stay in that sub-app
- Different sub-apps can evolve at different cadences
- Easier to onboard external contributors: "can you work on `signer`?" is a bounded ask
- Marketing + documentation gets simpler per sub-app

**Negative:**
- More subdomains to manage (offset by standard wildcard TLS + DNS)
- More CI/CD pipelines (offset by monorepo tooling — turborepo or nx; even basic npm workspaces handle this)
- Cross-app coordination (launching a savings wallet from the wallet app requires a link-and-install flow) — real but tractable UX work
- Discoverability: users need to know which app does what — hence the short descriptive names (wallet, family, signer, etc.) and a `onbitcoinstandard.com/apps` landing page
- Shared code must stay stable — `packages/crypto/` versioning discipline matters

**Explicitly accepted risks:**
- User confusion on "which app do I install?" — mitigated by a simple onboarding flow on `onbitcoinstandard.com` that recommends starting with `wallet` and adding others over time
- Families needing multiple apps installed — fine, they're all small (PWAs, a few MB each)

## Alternatives rejected

- **Monolithic "do everything" wallet app** — rejected for the reasons in Context; scope creep was strangling the plan
- **Separate app per platform (not per concept)** — considered and rejected. Platform is handled by PWA; concept-level splits give us real product + security benefits
- **Feature-flagged monolith** — considered. Same source tree, features toggled in/out at build time. Rejected because origin isolation is lost — all features share the same IndexedDB and service worker scope, undermining the security boundary
- **Post-PoC re-architecture** — waiting to split until after PoC launches would be more work later; split now while the codebase is small

## Implementation notes for Q2

- Repo restructured to `apps/<sub-app-name>/` + `packages/<shared>/` layout
- Turborepo or similar for build orchestration (or just npm workspaces if that's enough)
- `apps/wallet/` + `apps/signer/` + `apps/server/` are the Q2 deliverables
- Other sub-apps stubbed out with placeholder README files so the structure is visible from day one
- Build pipeline deploys each sub-app to its own Cloudflare Pages project
- Documentation lives in `docs/` at repo root; sub-app-specific docs go in `apps/<name>/docs/` only for sub-app-specific details

## References

- ADR-0014 — PWA-first shell (this ADR applies the shell strategy per sub-app)
- ADR-0015 — Identity + domain architecture (each sub-app is a distinct browser origin, separate from NIP-05 host origin)
- ADR-0010 — Signer architecture (becomes one of the sub-apps)
- PLAN.md — yearly plan, revised to reflect per-sub-app shipping schedule
