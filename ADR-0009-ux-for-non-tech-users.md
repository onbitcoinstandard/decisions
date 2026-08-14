# ADR-0009: UX principles for non-tech users and first-time bitcoiners

Date: 2026-04-15
Status: Accepted

## Context

The app's audience spans tech-comfortable bitcoiners (Rajesh, parents like Bob) and people who will never read documentation: partners, kids, grandparents, friends helping their own parents. The design challenge is that the protocol choices we've made (age encryption, SLIP-39, multisig, timelocked inheritance, graduation rituals, per-user keys) are genuinely complex. Hiding them turns the app into a custodial toy. Surfacing them as-is scares non-bitcoiners away.

We commit to a specific UX posture that threads that needle: **hide the vocabulary, keep the concepts.** Users can still do everything sovereignly — they just don't have to learn "multisig" or "VTXO" to use their wallet.

## Decision

The following principles are binding on every screen, flow, and piece of copy. They override designer preference and engineering convenience when in conflict.

### 1. Concrete metaphors replace jargon

Every technical term has a plain-language equivalent in the primary UI. The technical term may appear on an "advanced details" screen, never on the main flow.

Canonical mapping (expanded in a future `docs/COPYWRITING.md`):

| Technical | User-facing |
|-----------|-------------|
| VTXO | Balance |
| Multisig | Shared wallet / Extra-protected savings |
| Descriptor | Wallet recipe |
| Mnemonic / seed phrase | Backup words |
| Xpub export | Share your wallet info |
| PSBT | Pending transaction |
| Timelock | Waiting period |
| Address | Payment code |
| Transaction | Payment |
| Confirmation | Settling |
| Signing | Approving |

Inheritance wallet explanation (canonical phrasing):
> "A safe that opens automatically if we're not around to use it for a year. Christina, David, and the lawyer together can open it — like a will, but the money moves itself."

Graduation ritual:
> "Today your wallet grows up with you. You're making a new key that's just yours."

### 2. Progressive disclosure — delay scary things

Timeline of asks:

| When | Ask | Rationale |
|------|-----|-----------|
| First run (~minute 0) | Name + family + avatar | Concrete, human, zero crypto vocab |
| After first receive (~minute 10) | Back up wallet | User has skin in the game |
| After 1 week | Set a spending limit | Habits formed |
| After 1 month | Add a recovery helper | Wallet concept is now real |
| After 3 months | Consider savings wallet | Ready for multi-wallet |
| After 1 year | Consider inheritance wallet | Long-term context |

No pre-funded wallet sees its first scary word. Backups, recovery, multi-sig setup — all land after funds are present.

### 3. Safety-by-default, not by-configuration

Default answers work for the 90% case. Configuration is optional.

Examples:
- New kid wallet → default 100 sats/day spending limit, parent-only allow-list
- New savings wallet → default 2-of-3 with ASP assist
- New inheritance wallet → default Jones-family policy (2-of-3 primary + 6-month recovery + 6-month heirs)

The questionnaire asks *human* questions, not *technical* ones:
- "Who should be able to spend from this wallet?" → derives multisig policy
- "What's this wallet for?" → derives wallet type
- "Who helps if something goes wrong?" → derives backup recipients

### 4. Concrete UI over abstract data

- Contacts show photos and names, not addresses
- Wallets are **cards** with names, emoji, colors
- Transactions show counterparty names and chore descriptions, not hashes
- Hashes / addresses / descriptors are one tap away ("Show details") but not default-visible

### 5. Fiat-first for the first month

Every amount displayed with fiat prominent, sats secondary, for the first ~30 days of app use. Gentle opt-in nudge at day 30 to flip to sats-first. Expert toggle in settings for sats-only from day one.

Rationale: people need fiat for scale-checking until they have sat-intuition.

### 6. Mistake prevention before mistake

Every send flow includes:

- **Large-amount detector** — "This is more than you've spent in the last 30 days."
- **New-recipient detector** — "You haven't sent here before. Double-check the name."
- **Round-trip typing** for over-threshold sends — re-type the amount
- **Cooldown** for very large sends — "Wait 5 minutes and tap Send again."
- **Biometric confirmation** for every non-trivial send — user presence becomes the signal
- **Undo** within 10 seconds for internal family transfers (server holds briefly before broadcast)

### 7. Reassuring copy, not threatening copy

- NOT: "⚠️ WARNING: If you lose this phrase you will lose all your bitcoin forever"
- YES: "Let's make a safety net together. We'll write down a few words so your sats are safe even if this phone breaks. It takes 2 minutes."

Framing: backups are a milestone achieved, not a calamity avoided.

### 8. First-time training wheels, dismissable

One-time in-flow explanations for first-ever actions. Each is a single screen, one paragraph, dismiss-on-read.

- First send: what a bitcoin transaction is, irreversibility, confirmation timing
- First receive: what a payment code is, safety of sharing, why no one can drain it
- First backup: what backup words are, where to keep them
- First chore approval: what happens when you approve

After dismissal, never shown again. Help content is contextual (tap "?" icons), not a separate doc section.

### 9. Kid-specific patterns

- **Picture passphrase** for under-14s — 4-emoji sequence (from a 16-emoji grid) derives the KEK via Argon2id, same age-file scheme. No typing required.
- **Animated chore payouts** — visible, satisfying, proves the transfer happened
- **Goal thermometers** — "Saving for Lego: 2,300 / 10,000 sats" with a progress bar
- **Human-voice copy** — "Mom is sending 500 sats!" not "Incoming transaction"
- **Parent-pre-configured defaults** — kid's first hour is purely consumption; configuration by the parent beforehand

### 10. Family helper mode

A guided co-setup flow: tech-savvy family member scans a QR link with their phone to join a helper session on another person's phone. The helper sees the onboardee's screen, can tap-advance, but **cannot sign for them**. Onboardee's biometric / passphrase is required at every actual custody action.

This is the dominant onboarding path in practice: Rajesh onboards his parent, an OBS member onboards their partner, Uncle Jim onboards his sister-in-law.

### 11. Offline-tolerant and latency-honest

- No "syncing..." spinner blocks the home screen
- Recent balance and contacts work immediately from local cache
- Server latency is background; UI acts optimistically and rolls back on failure with a clear explanation
- Error states are actionable: "Couldn't reach server. Try again?" big button, not a stack trace

### 12. Graduation + inheritance framing is human, not technical

- Graduation screen: "Today your wallet grows up." Not "Key rotation ritual."
- Inheritance screen: "A gift that waits for when it's needed." Not "Timelocked multisig recovery path."
- Attorney onboarding: "Edward's job is to hold a spare key." Not "External inheritance-role-only pubkey."

## Consequences

**Positive:**
- Non-tech users can use the app without learning a vocabulary
- First-time bitcoiners onboard with confidence, not confusion
- Kids actually use the app because it speaks to them
- OBS audience can credibly onboard their parents, partners, and neighbors
- Reduces support burden — fewer "what does this mean?" questions

**Negative:**
- Design + copy review becomes a gate on shipping features — every screen runs through this doc
- Slight onboarding-flow complexity for the one-time explainers
- Some bitcoiners may object to "dumbed down" vocabulary; mitigated by advanced-mode toggle for technical details
- Two-language maintenance burden (plain + technical) — accepted

## Implementation notes

- Write `docs/UX-PRINCIPLES.md` in Q2 as the living expanded version of this ADR
- Write `docs/COPYWRITING.md` as the canonical vocabulary mapping, updated with every new string
- Write `docs/ONBOARDING-SCRIPT.md` as the full first-run walkthrough text, screen-by-screen, reviewable by a non-bitcoiner proxy
- Every PR that adds user-facing copy is reviewed against these docs
- Alice-level user test every quarter: hand the latest build to someone non-technical, watch them use it silently, fix what they trip on before the next sprint

## Alternatives rejected

- **Expose technical terms with explanations** — "Multisig (a wallet that needs multiple keys to spend)" — rejected because it creates unnecessary cognitive load on the primary flow; advanced mode exposes the technical terms for users who want them.
- **Two separate apps (tech + non-tech)** — rejected because family members mix, and we'd duplicate effort.
- **Wizard-only, no free navigation** — rejected because it infantilizes experienced users; progressive disclosure within a free-navigation UI works better.

## References

- `docs/FAMILY-LIFECYCLE.md` — the graduation and inheritance framings
- `docs/CUSTODY-MODEL.md` — what's actually happening under the friendly copy
- bitcoin.design — "delay backup until after first funds" and "no balance on receive screen" are adopted from their guide
