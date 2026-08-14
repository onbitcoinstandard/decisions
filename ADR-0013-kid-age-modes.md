# ADR-0013: Kid age modes and migration

Date: 2026-04-15
Status: Accepted (amends ADR-0001)

## Context

The earlier custody model assumed every family member — including kids — has their own device and their own signing key from day one. That assumption doesn't fit how real families work:

- Kids doing chores are typically 6–12 years old
- Many families (reasonably, per their parenting values) don't give kids phones until 13+
- A 6-year-old cannot meaningfully manage a private key
- Shared family devices (iPads, household tablets) are common and a perfectly sensible intermediate step

Forcing every kid to have their own device to participate in the chores loop would exclude the dominant use case. Pretending a 6-year-old has sovereign custody when really the parent is signing everything is dishonest. We define an explicit three-mode model that scales custody with the kid's age and device access.

## Decision

### The three modes

**Mode A — Parent-held custody (ages ≤~7, or any kid without device access)**

- No kid profile, no kid passphrase, no kid device
- Parent's wallet contains a "Children" section; each kid is a virtual account with name, photo, balance, goal, chore history
- Sats held under a kid-specific BIP-32 derivation path from the parent's signing key (e.g. `m/44'/0'/42'` for kid #42)
- Parent signs all transactions — incoming (chore payouts) and outgoing (spending on kid's behalf or for kid-managed gifts)
- Kid interacts with their "wallet" through:
  - Printable chore chart (app generates weekly PDF)
  - Physical milestone cards (printed redemption tokens for goal achievements)
  - Weekly "money meeting" — parent shows kid their balance on the parent's device
- **Custody property:** parent has cryptographic access. This is explicit, time-boxed, and shown in the UI ("You are holding Ayla's sats for her")

**Mode B — Shared family device (ages ~7–12, most common)**

- Family iPad / shared tablet / family desktop
- Kid has own profile with picture-passphrase (4-emoji sequence — ADR-0009)
- Kid's age-encrypted key file stored on the shared device; decryptable only with kid's picture passphrase
- Kid unlocks own UI, sees own balance, picks goals, requests purchases
- Parent approves all outgoing transactions (though kid signs with their own key — app builds parent-countersigned multi-recipient Arkade transaction)
- **Custody property:** parent has physical access to the device but not to the kid's picture passphrase. Operator-blind vs the parent in spirit, though weaker than separate-device isolation. Equivalent to any shared-computer security model.

**Mode C — Own device (ages ~12+)**

- The standard per-user-keys model originally specified in ADR-0001
- Kid's key on kid's phone
- Kid approves own transactions within parent-configured spending limits
- Parent approves limit raises, new allow-list merchants, large sends
- **Custody property:** full operator-blind, identical to adult users

### Migration points (all explicit + ceremonial)

| Transition | Trigger | Steps |
|------------|---------|-------|
| A → B | Parent toggles "Kid can access the family iPad now" in kid settings | Generate fresh kid signing key on shared device → sweep parent's kid-derived-path balance to kid's new key → parent loses cryptographic access to kid's sats (still holds physical device access) → role migration event published to family graph |
| A → C (skip B) | Kid gets own phone | Same migration, target is kid's new phone; parent signs one last transaction moving sats, then loses access |
| B → C | Kid gets own phone | Export kid's age file from shared device → import on new device via QR or helper mode → wipe age file on shared device → confirmation that shared device no longer has kid's key |
| C → adult | 18th birthday (configurable) | Graduation ritual per ADR-0004 — new adult key, VTXO sweep, backup re-export, role flag `child` → `adult_member` |

Each migration is presented as a **visible flow** the parent initiates with explanation. Sats move (with visible fees), custody tightens, the kid's role flag in the family membership record progresses. The kid sees the change too — "Your wallet is now on your iPad, not Dad's phone. Dad can't spend it anymore."

### Mode selection during kid onboarding

Parent answers two questions:

1. **How old is this kid?** — pre-fills recommended mode based on age bands
2. **Does this kid have a device?**
   - "No device" → Mode A (override to Mode B if parent plans to let them use a family tablet)
   - "Shared family device (iPad, family tablet)" → Mode B
   - "Own phone / tablet" → Mode C

Parent can override the recommendation in either direction (e.g. a mature 8-year-old in Mode C, or a cautious parent keeping a 13-year-old in Mode B). The mode is a property of the kid's family-membership record, not a global app setting.

### HD derivation path convention (Mode A)

Children under a parent's wallet get hardened derivation paths using a monotonic counter:

- Kid #1: `m/44'/0'/<account>'/0'/<kid-counter>'`
- Kid #2: `m/44'/0'/<account>'/0'/<kid-counter>'`
- etc.

Each kid's derived account has its own xpub, its own address set, its own VTXO pool inside Arkade. Parent signs transactions specifically for that kid's derivation path, which cleanly scopes access for accounting and migration.

When the kid migrates to Mode B or Mode C, the parent's wallet **still knows** the kid's derivation path but no longer holds the post-migration key. The derivation path becomes archival metadata; a new independent key lives on the kid's device.

### UX honesty about custody

In Mode A, the parent's UI shows **"You are holding sats for [kid name]."** Not "Kid's wallet." Not "Child account." The parent is the custodian. When the kid visits Mode A screens, they see "Dad is holding your sats until you're ready for your own wallet."

In Mode B, kid's UI says **"Your wallet on the family iPad."** Parent sees a list of kids with "On the iPad — you can physically access but not spend."

In Mode C, kid's UI says **"Your wallet on your phone."** Parent sees "Kid's phone — you approve big sends."

These honesty labels matter. They teach both the parent and the kid what the custody property actually is, and they make the migration transitions feel like real upgrades rather than cosmetic changes.

### What this does to the operator-blind claim

The strict "operator-blind for every family member from day one" claim is softened for kids in Modes A and B, and stated honestly in marketing and UI. Specifically:

- **Mode A kids:** parent has cryptographic access (acknowledged, time-boxed)
- **Mode B kids:** parent has physical device access; picture-passphrase isolation is a real but imperfect wall
- **Mode C kids:** full operator-blind, same as adults
- **Graduated adults:** full operator-blind, independent

The operator-blind property strengthens as the kid matures. This is more honest than "every family member holds their own key from day one" and arguably a better product story — we're modeling custody maturation, not pretending a 6-year-old is a sovereign bitcoin user.

## Consequences

**Positive:**
- The product works for the dominant use case (young kids without phones)
- Custody scales with kid maturity — realistic, honest, and teachable
- Mode migrations become product moments that reinforce "your keys, your coins" as the kid grows
- No awkward "application-layer child accounts forever" pattern from the original bitcoin.design recommendation — Modes A and B are explicitly time-limited
- UI honesty labels make parents and kids both understand what's happening

**Negative:**
- Three modes = three UI flows = three testing surfaces
- Mode migrations are non-trivial — fees, VTXO sweeps, coordination between devices
- "Operator-blind for every family member" isn't quite true for kids below majority; marketing must be careful
- Parents in Mode A have more power than we'd like if they use it badly (they could spend kid's sats for themselves) — mitigated by UI transparency + family financial meetings + chore history receipts that kid sees
- Edge case: kid loses shared-device picture passphrase in Mode B (parent helps via backup flow; same as adult backup recovery)

## Implementation notes for Q3

- Mode A is the FIRST kid mode to implement (Q3 family MVP) — simplest, no kid device required
- Mode B follows in Q3/Q4 — requires multi-profile support on shared devices (picture-passphrase + age-file-per-profile)
- Mode C = the originally-designed kid flow; Q3
- Migration flows (A→B, A→C, B→C) in Q4 when families naturally hit them; simulated transitions tested in dev
- Printable chore chart PDF generation in Q3 (Mode A)
- Physical milestone reward cards PDF generation in Q3 (Mode A)

## Alternatives rejected

- **Require kid to have own device** — excludes the majority use case; rejected
- **Application-layer child accounts forever** (original bitcoin.design pattern, parent always custodies) — rejected because it blocks graduation to real custody; our three-mode model keeps the same simplicity for young kids but adds explicit migration
- **Single "kid mode" with configurable options** — confusing UI; users can't tell at a glance what custody property applies. Explicit modes with clear labels are better.
- **Custodial service for kids' sats** (we hold them) — violates ADR-0001; never considered seriously

## References

- ADR-0001 — amended to point here for kids below majority
- ADR-0004 — graduation ritual (Mode C → adult)
- ADR-0009 — UX principles; picture-passphrase for kids under 14
- bitcoin.design inheritance wallet — the "weekly money meeting" idea is borrowed from their family-financial-meetings pattern
- `docs/FAMILY-LIFECYCLE.md` — updated to show mode progression alongside graduation + inheritance
- `docs/PARENT-GUIDE.md` — practical parent-facing walkthrough of all three modes
