# ADR-0001: Operator-blind custody with per-user keys

Date: 2026-04-15
Status: Accepted

## Context

We are building a multi-wallet bitcoin app for families. Two competing custody models were on the table:

**A. Application-layer child accounts** (bitcoin.design recommendation) — parent holds all keys, each child has a reserved balance inside the parent wallet managed via UI-only accounts. Drastically simpler, no per-kid keys, no per-kid backup.

**B. Per-user keys with social family graph** — each family member (parent, partner, each kid) holds their own key. "Family" is a social layer. Parents approve kids *joining* the family, not kids' transactions.

## Decision

We go with **(B): per-user keys, operator-blind**.

Rationale:
- Teaches custody from day one — the kid learns "your key, your coins" by living it, not by eventually graduating to a real wallet.
- No false sovereignty — parent cannot silently drain an adult's (or graduated kid's) funds. Guardrails (limits, allow-lists) are client-side and require the user's own signature.
- Architecturally open — the server does not assume it is the only operator, and the codebase is open source. This preserves the *possibility* of self-hosting without committing to ship packaged install tooling. Unilateral VTXO exit to on-chain remains the guaranteed sovereignty escape hatch.
- Operator protection — if subpoenaed or breached, the operator has only ciphertext; cannot produce plaintext keys or spend on any user's behalf.

### Important amendment — kids below majority

The strict per-user-keys rule applies to every adult in the system. For **kids under majority**, custody scales with age and device access via a three-mode model (see ADR-0013):

- **Mode A (no device, ages ≤~7)** — parent holds a kid-specific HD-derived key on the parent's wallet. Parent has cryptographic access. Temporary and explicitly acknowledged in-app.
- **Mode B (shared family device, ages ~7–12)** — kid holds own age-encrypted key on shared device under picture-passphrase. Parent has physical access to the device but not to the kid's passphrase. Operator-blind vs the parent in spirit, though weaker than separate-device isolation.
- **Mode C (own device, ages ~12+)** — full per-user-keys, identical to adult users.

Every mode transition is a **visible, ceremonial migration** the parent runs with the kid. A → B, A → C, B → C are each explicit events (key generation, VTXO sweep, access shift). C → adult at majority is the graduation ritual (ADR-0004).

The operator-blind guarantee is strongest for adult users and strengthens for kids as they progress through the modes. We communicate this honestly rather than pretending a 6-year-old holds their own key. See ADR-0013 for the full specification.

## Consequences

**Positive:**
- Strong sovereignty story for marketing and for the user
- Reduced operator liability (can't lose what you don't have)
- Legitimate "your keys, your coins" claim — many bitcoin apps claim this while in fact holding a co-signing key

**Negative:**
- Kid recovery is harder. A kid who loses their device and forgets their backup passphrase cannot be bailed out by the operator. Must be mitigated by social recovery / parent-held backup copies (see ADR-0002 when written).
- Each new user needs a real backup flow — the word-reorder ritual can't be skipped for kids.
- UI must be unusually careful about making this model understandable to non-technical users.

## Alternatives rejected

- **Fully custodial** — considered, rejected. Incompatible with the OBS sovereignty thesis; also creates regulatory obligations we don't want (MSB, KYC).
- **Hybrid (parent custodial over kid)** — considered, rejected. Creates the false-sovereignty pattern we want to avoid; kid can never truly graduate without a key-migration ritual that most families will skip.

## References

- `/home/rajesh/personal_social/research/bitcoin-design-wallets/00-multiple-wallets.md` (the application-layer-account recommendation we're deliberately declining)
- `/home/rajesh/personal_social/research/bitcoin-design-inheritance/` (Jones family model — each family member holds real keys)
