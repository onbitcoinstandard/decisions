# ADR-0005: Multi-family membership and multi-role participation

Date: 2026-04-15
Status: Accepted

## Context

Real families don't fit a single-tree model. Divorced parents, blended families, adult children with their own kids, stepparents, guardians, trusted relatives hosting via Uncle-Jim patterns — all need to coexist in the system. Beyond the family graph itself, trusted outsiders (attorneys, professional executors, sibling-of-parent) may hold inheritance keys without being family members.

A data model that assumes one user ↔ one family forces awkward workarounds (duplicate accounts, separate logins per household). We choose to get this right from schema day one.

## Decision

### Identity is one pubkey. Membership is many.

A user's identity is a single Nostr pubkey (per ADR-0002). That pubkey can simultaneously hold:

- **N family memberships** — each with its own role (`parent`, `partner`, `child`, `adult_member`, `guardian`)
- **M inheritance-key-holder records** — each tied to a specific wallet descriptor in a specific family (may or may not overlap with family memberships)
- **K contact-only relationships** — pubkey appears in a family's contact graph without full membership (grandparent who just wants to send birthday sats)

No duplicate accounts. No separate logins per household. The client app presents a **family switcher** at the top of the UI so the user picks which context they're operating in at any given moment.

### Data model (conceptual)

```
users
  pubkey (primary identity)
  nostr_npub

families
  family_id
  created_by (pubkey)

family_memberships
  family_id
  user_pubkey
  role             -- parent | partner | child | adult_member | guardian
  since            -- timestamp of membership grant
  granted_by       -- pubkey of the parent/admin who approved

inheritance_roles
  wallet_descriptor_id
  family_id        -- the family whose inheritance wallet this is
  holder_pubkey    -- who holds the key
  relationship     -- heir | attorney | executor | trusted_third_party
  added_at
  granted_by
```

A pubkey's full profile is just the union of these rows.

### Role flags are per-membership

A pubkey is a `child` in family A and simultaneously an `adult_member` in family B. The graduation ritual (ADR-0004) flips the role flag **for the family in which graduation occurs** — it does not globally change the user. Their other memberships are untouched.

### Inheritance key holders do NOT see family day-to-day state

Holding an inheritance key for family A does not grant visibility into family A's chores, balances, contacts, or activity feed. The key holder sees only:
- The wallet descriptor they're part of (structural, not financial)
- Maintenance reminders ("time for the 6-month key check")
- Succession-trigger events (if/when they fire years later)

This matches the bitcoin.design Jones family privacy requirement: Christina and David hold keys but can't see balances while parents are alive.

Membership in the social graph and participation in a descriptor are two separate grants, each with its own visibility scope.

### Family switcher UX

- Top of app shows active family name + icon/color (users customize per family per the multi-wallet guide)
- Tap → list of families the user belongs to + option to create a new one or accept an invite
- Inheritance-only relationships appear in a separate section ("You hold keys for: …")
- Each family's wallets, chores, contacts are scoped — no cross-family leakage in the UI

### Uncle-Jim hosting pattern

A tech-savvy relative running infra for multiple households is just "N families all pinned to one ASP + one server instance." Nothing at the family-graph layer changes. Server multi-tenancy (ASP fee splitting, per-family quotas) is a phase-2 infra concern, not a data-model concern.

### Cross-family actions

Sending sats from a wallet in family A to a contact in family B is just a VTXO transfer — no special protocol needed, Arkade handles it. The UI shows "From: Medampudi household wallet → To: grandpa (in Patel household)" so the user understands the source.

Contact discovery across families: opt-in only. A user in family A can share a single address/contact with a user in family B without either family becoming visible to the other.

## Consequences

**Positive:**
- Realistic for actual families (divorced, blended, multigenerational)
- No awkward duplicate-account patterns
- Uncle-Jim hosting is a natural fit
- Inheritance key holders can be true outsiders (lawyers) with no privacy leak into the family
- Contact-only relationships let grandparents participate without full membership

**Negative:**
- UI must make the "which family am I in right now?" context unambiguous — a user accidentally paying from the wrong household is a real failure mode
- Notification fan-out is more complex (events route to the right family context)
- Server multi-tenancy has to be designed in from the start even though we run the only ASP initially

## Implementation notes for Q2/Q3

- Q2: the data model above in the server schema. Family switcher stubbed but with only one family.
- Q3: multi-family switcher becomes real when we ship the first shared partner wallet and the kid use case — test cases to exercise: divorced-parent kid, adult-in-two-families
- Q3-Q4: inheritance-key-holder records land when savings/inheritance wallets ship. External-key-holder (attorney) flow tested with a real professional.
