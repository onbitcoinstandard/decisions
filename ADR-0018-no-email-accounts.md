# ADR-0018: No email-account association — npub + NIP-05 only

Date: 2026-04-16
Status: Accepted

## Context

Asked whether user accounts / credentials should be associated with email addresses (as most mainstream products do) or kept pubkey-only (as arkade.money and nsec.app do). This decision has been implicit in prior ADRs (0002 no-passwords, 0005 pubkey-primary-identity, 0015 NIP-05-as-overlay) but deserves an explicit ADR to lock it in against future drift.

## Decision

**No email addresses, ever.** Identity is a Nostr pubkey (npub) with optional NIP-05 aliases as the human-readable layer. No email field on user profiles, no email in server database, no email-based authentication, no email-based recovery, no email-based notifications.

This matches arkade.money, nsec.app, Damus, Amethyst, and the sovereign Nostr ecosystem generally.

## Rationale

### 1. Recovery-via-email is the exact anti-pattern we avoid

Every email-account-based system eventually implements "I forgot my password" via email. Once we know a user's email, someone with legal compulsion can demand we reset or migrate custody based on email ownership. Operator-blind (ADR-0001) means we genuinely cannot do that — and having no email means we don't even have the material for someone to demand.

### 2. GDPR / PII-surface elimination

Email addresses are PII under every major data-protection regime (GDPR, UK GDPR, CCPA, India DPDP Act). Collecting them creates:
- Right-to-erasure obligations
- Data-breach notification obligations
- Cross-border transfer complications
- Processor agreements with email-service providers
- Subject access request handling

We currently hold zero PII. That's a feature worth protecting.

### 3. "Account" mental model creeps back in

Email = account. Users expect: forgot-password flows, account-merge, email-change flows, unsubscribe links, notification preferences, account deletion confirmation emails, support-via-email. Each of those is support work we'd spend days on.

Pubkey identity avoids this entirely. Users understand "your keys = your account" once they're set up, and there's no forgot-password flow to support because we literally cannot help them recover keys we never had.

### 4. Email-provider dependencies we don't want

- Gmail's spam classifier deciding whether our notifications reach users
- Bounce handling and deliverability ops
- SMTP infrastructure (our own, or a provider like Postmark/SendGrid)
- Sender reputation management
- DMARC/DKIM/SPF configuration

Sovereignty product ↔ email as critical path = mismatch.

### 5. Ecosystem convention

Nostr-native bitcoin products (arkade.money, nsec.app, getalby.com, primal.net) already ship without email. Users in our target audience are used to it. Adding email now would break the convention and signal "we're mainstream-compromised."

## Notifications without email

The one legitimate use of email in most products is notifications. We don't need email for this:

**Option 1 — PWA push notifications** (primary)
- Browser-native Push API + service worker
- iOS 16.4+ supports this for installed PWAs
- User grants permission per origin during install
- Immediate, doesn't require an email inbox

**Option 2 — Nostr DMs** (secondary, user-chosen)
- Standard kind:4 (or kind:1059 gift-wrapped) messages via Nostr relays
- User's existing Nostr client receives notifications
- Decentralized, sovereign, user-controlled

**Option 3 — Nothing** (valid)
- Users who don't want notifications get none
- App works fine without them — all state is queryable in-app

**Not an option:** email, SMS, phone calls. None of these get built.

## What NIP-05 gives us in lieu of email

- Human-readable identity (`mike@ross.onbitcoinstandard.com`) as good as email for user-to-user reference
- Cross-ecosystem verification — every Nostr client can resolve our NIP-05 identifiers
- Multi-alias per pubkey (ADR-0015 amendment) for different contexts

Email's only real advantage is that grandma already has one. For our target audience (sovereignty-aligned families), grandma is either being onboarded via the "family helper mode" flow (ADR-0009) by a tech-savvy relative, or she's participating as a contact-only participant without an account at all (no onboarding needed).

## Consequences

**Positive:**
- Zero PII → no GDPR/CCPA/DPDP overhead
- Zero email infrastructure → zero email-related ops burden
- Operator-blind posture intact → no email-based recovery back-door
- Account mental model avoided → less support burden
- Matches sovereign Nostr ecosystem convention
- Nothing to leak in a breach

**Negative:**
- Onboarding grandma requires a family helper (per ADR-0009 design) or we lose her as a user — accepted tradeoff
- Notifications are PWA-push-only or Nostr-DM-only — works but less universal than email
- No "email my alpha testers" broadcast channel — use Nostr zaps / OBS blog / Geyser updates instead
- Power users who specifically want email notifications have to set up their own relay-to-email bridge — documented as a "here's how if you want it" footnote, not a supported feature

## Alternatives explicitly rejected

- **Optional email, off by default** — creeps into required territory over time; still requires infrastructure; still creates PII; still creates support-via-email expectation. No halfway.
- **Email-only for notifications, no email-login** — still requires SMTP infra, still PII. No meaningful benefit.
- **Use a third-party email provider** (Mailgun, Postmark) — same PII + vendor-dependency problems plus data-processor agreements.
- **Allow NIP-05 claim via email verification** — mixes concepts; NIP-05 works via signed pubkey verification, email doesn't improve it.

## Implementation notes

- User profile schema has zero email field
- Server database has zero email column
- No PII data-subject endpoint needed (we have nothing to subject)
- Privacy policy explicitly states "we do not collect email addresses or any other PII"
- Notification system implements PWA push + Nostr DMs only
- Support channel is Nostr DMs to our support npub, Geyser comments, or public OBS blog comments. No `support@onbitcoinstandard.com` email address.

## References

- ADR-0001 — operator-blind custody (email-based recovery contradicts this)
- ADR-0002 — no passwords (same logic applies to email)
- ADR-0005 — pubkey as primary identity
- ADR-0015 — NIP-05 as human-readable overlay
- arkade.money — reference product that ships without email
- nsec.app — reference product that ships without email
