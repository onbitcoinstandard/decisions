# ADR-0015: NIP-05 identity overlay, Nostr-pubkey primary, domain architecture

Date: 2026-04-15 (amended 2026-04-16)
Status: Accepted (amended — subdomain-per-namespace promoted to primary, multi-alias support added, generalized to communities)

## Amendment note (2026-04-16)

Original decision shipped flat namespace (`firstname.familyname@onbitcoinstandard.com`) as primary with subdomain-per-family as a deferred option. Amended to make **subdomain-per-namespace the primary pattern**, supporting both biological families and arbitrary communities, with **multi-alias-per-pubkey** as a first-class feature. Rationale below in "§ Subdomain-per-namespace" and "§ Multi-alias per pubkey". The flat namespace becomes one namespace among many (`obs-members.onbitcoinstandard.com` or similar), not the privileged default.

## Context

Pubkeys are our primary identity (ADR-0002, ADR-0005) — the project has no accounts, no passwords, no emails. But `npub1abc…xyz` is not a thing a human shares at dinner, and it's not a string a 10-year-old remembers. We need a human-readable overlay that preserves the no-accounts property.

NIP-05 (Nostr Implementation Possibility 05) is the standard solution already used across the Nostr ecosystem: a DNS-based, static-file-served mapping from `user@domain` to a pubkey. Compatible with every Nostr client. No account required on the lookup server — just HTTP GET returning JSON.

We also need to commit to a domain architecture before we start coding the server, since origins are load-bearing for PWA isolation (ADR-0014).

## Decision

### Domain architecture

```
onbitcoinstandard.com                ← OBS content site + NIP-05 + future LUD-16
├── /.well-known/nostr.json          ← NIP-05 directory (static or dynamic)
└── /.well-known/lnurlp/<user>       ← LUD-16 Lightning addresses (post-PoC)

wallet.onbitcoinstandard.com         ← Wallet PWA
signer.onbitcoinstandard.com         ← Signer PWA (offline-first SW)
api.onbitcoinstandard.com            ← Signed-event API + backup blob storage + ASP proxy
media.onbitcoinstandard.com          ← Blossom (existing, per project CLAUDE.md)
```

Rationale for the split:
- **Origin isolation** — wallet IndexedDB cannot be read by signer code, signer keys cannot be accessed by wallet code. Browser-enforced, not code-enforced.
- **NIP-05 on root domain** — simplest URL for users to share: `alice.ross@onbitcoinstandard.com` not `alice.ross@identity.onbitcoinstandard.com`
- **API on subdomain** — clean CORS story, easy to move to a separate host if needed, doesn't pollute root for content
- **Signer on subdomain** — forces user to install separately, reinforces "this is a different app on a different device"

### NIP-05 identifier format

Pattern: `<firstname>.<familyname>@onbitcoinstandard.com`

Examples:
- `mike.ross@onbitcoinstandard.com`
- `ayla.medampudi@onbitcoinstandard.com`
- `bob.smith@onbitcoinstandard.com`

The NIP-05 spec permits local parts matching `/^[a-z0-9-_.]+$/`. Our `firstname.familyname` format is spec-valid and doesn't require any protocol extension.

**Collision resolution:**
- First-come-first-served claim via signed request
- Two users with identical `firstname.familyname`: second user prompted to choose a disambiguator
  - `mike.ross.2@onbitcoinstandard.com` (numeric)
  - `mike.t.ross@onbitcoinstandard.com` (middle initial)
  - `mike.ross.ca@onbitcoinstandard.com` (region/state initial)
- Reserved list: `admin`, `support`, `security`, `abuse`, `postmaster`, `webmaster`, `info`, single-word common names (`alice`, `bob`, `charlie` — to prevent cybersquatting)
- Known-person impersonation (e.g. `satoshi.nakamoto`): moderated by us, revocable per ToS

**Claim and revocation:**
- Claim: user's client posts a signed request `{name: "mike.ross", pubkey: "..."}` to our API. Server verifies signature, checks availability, writes to `nostr.json`.
- Update (migrating to new pubkey, e.g. after graduation or device compromise): signed request from the OLD pubkey authorizes the change to a new pubkey. Prevents hijacking.
- Delete: signed request from the current pubkey removes the entry. Server's obligation is immediate.
- Revocation by us: only under explicit ToS violations (impersonation, trademark, clearly malicious). Documented process.

### Privacy defaults by role

| Role | NIP-05 default |
|------|----------------|
| Parent / partner / adult_member | Opt-in; shown as suggestion during onboarding |
| Kid in Mode A (no device) | Not offered — kid doesn't have a device to claim from |
| Kid in Mode B (shared device) | Opt-in by parent; default **off** (privacy protection for young kids) |
| Kid in Mode C (own device) | Opt-in with parent approval (under 18) |
| Graduated adult | Opt-in, fully user's choice |
| Inheritance-only key holder (attorney) | NOT offered via our NIP-05 — they should use their own professional identity |

Rationale: publicly listing a 10-year-old at `ayla.medampudi@onbitcoinstandard.com` on a globally-searchable Nostr directory is a vector for stranger contact. Default-off for minors; parent consciously opts in per kid.

### nprofile — the shareable artifact

NIP-19 `nprofile1...` strings bundle:
- pubkey
- relay hints (where to find their events)
- optional NIP-05 verification

When a user wants to share their identity, the app generates a single nprofile string + QR. One scan imports all three into any Nostr-compatible client — our app, Damus, Amethyst, Primal, etc.

UI shows `alice.ross@onbitcoinstandard.com` as the friendly label; the nprofile is the artifact actually exchanged.

### Subdomain-per-namespace (PRIMARY — amended 2026-04-16)

Every family or community owns a subdomain of `onbitcoinstandard.com`. Users within that namespace have local-part identifiers:

```
mike@ross.onbitcoinstandard.com            (Ross family)
ayla@medampudi.onbitcoinstandard.com       (Medampudi family)
alex@btcinvestors.onbitcoinstandard.com    (Bitcoin investors group)
sam@obs-members.onbitcoinstandard.com      (OBS community — our flagship namespace)
```

**Benefits:**
- Namespace-scoped — no cross-namespace collisions (two Mike Rosses in different families each get clean identifiers)
- Natural human-readable identity: *"Mike from the Ross family"* is how people actually think about it
- Portable: family migrates to `ross.family` if they buy their own domain — local parts unchanged
- Multiple communities: one pubkey can claim aliases across namespaces (see § Multi-alias)
- Per-namespace moderation: the namespace creator is admin; we're not the global nanny

**Communities, not just families:**

"Namespace" generalizes beyond biological families. Any group can register one:
- Biological families (`ross`, `medampudi`, `smith-jones`)
- Friend groups (`collegefriends-2009`, `cypherpunk-dinners`)
- Professional communities (`btcinvestors`, `cyoa-devs`, `nostr-builders`)
- Formal communities (`obs-members`, `stacker-news-club`)

The pattern is agnostic to what "group" means. If you have a cohesive set of people who want a shared identity namespace, you can register one.

### Namespace registration & admin model

**Claiming a namespace:**
- Any user can register a new namespace by posting a signed request to `api.onbitcoinstandard.com`
- First-come-first-served for non-reserved names
- Reserved list (small, static): `admin`, `www`, `api`, `signer`, `wallet`, `family`, `shared`, `savings`, `inheritance`, `media`, `support`, `security`, plus trademark-risky strings (`amazon`, `google`, `coinbase`, etc.)
- ToS-based revocation for clear impersonation, trademark violation, or malicious use
- Registration itself is free (no cost to us beyond static file serving)

**Namespace admin rights:**
- Creator becomes admin by default
- Admin can: approve/reject membership requests, remove names, transfer admin rights
- Admin cannot: change another user's pubkey (that requires signed request from the pubkey itself)
- Namespace can be configured as **open** (anyone can add themselves, like an OBS community) or **invite-only** (admin approves each addition, like a family)

**Namespace-level privacy:**
- Namespace can publish its member list or not
- Individual users opt in/out of appearing in the namespace directory

### Multi-alias per pubkey — a user's identities

**Core principle: one pubkey, many aliases.** A user's Nostr keypair is their canonical identity; NIP-05 aliases are the human-readable addresses pointing to that pubkey. Multiple aliases can point to the same pubkey, each in a different namespace.

Example: Mike has pubkey `npub1abc…xyz`. He claims:
- `mike@ross.onbitcoinstandard.com` (his family)
- `mike@btcinvestors.onbitcoinstandard.com` (investment club)
- `mike.r@obs-members.onbitcoinstandard.com` (OBS community)

All three resolve to `npub1abc…xyz`. Mike picks one as **primary for display in his profile**; others remain resolvable aliases.

**Why this matters:**
- Divorced parents whose kids belong to both households: kid has one pubkey, aliases in both namespaces
- Blended families: same logic
- Professional vs personal identity separation
- Community membership without dissolving biological family membership

**Data model:**

```
nip05_entries
  namespace_domain     -- "ross.onbitcoinstandard.com"
  local_part           -- "mike"
  pubkey               -- same pubkey allowed across many rows
  is_primary_for_pubkey
  admin_pubkey         -- who controls this namespace's roster
  registered_at
```

Primary-key constraint: `(namespace_domain, local_part)` unique; `pubkey` may repeat.

**Profile UX:**

```
Your identities (primary shown in your profile):

  ● mike@ross.onbitcoinstandard.com          [Ross family, primary]
  ○ mike@btcinvestors.onbitcoinstandard.com  [Bitcoin investors]
  ○ mike.r@obs-members.onbitcoinstandard.com [OBS community]

  [+ Claim another identifier]
  [+ Create a new namespace (family or group)]
```

Switching primary is instantaneous; other aliases continue to resolve.

### Decoupling from family wallet graph

Critical architectural note: **NIP-05 namespace and family wallet graph are independent features.**

- A user can claim an alias in a namespace without using any wallet feature — they get a free NIP-05 identity, nothing more.
- A family can run a private wallet graph (chores, approvals, shared wallets) with zero public NIP-05 presence.
- A user can belong to a namespace they have no wallet relationship with — e.g. joining `btcinvestors` for social identity only, while their actual wallet family stays private.

**Loosely coupled at the UX layer:** when creating a namespace, the app may offer "want to set up chores + shared wallet too?" — but either feature works standalone. The family wallet graph (ADR-0005) continues to be a private, signed-event-based construct on our server; NIP-05 is the public identity directory.

### Infrastructure

- DNS: wildcard `*.onbitcoinstandard.com` pointing at our server (Cloudflare DNS)
- TLS: Let's Encrypt wildcard cert via DNS-01 challenge, renewed automatically
- Web: single handler at all subdomains responds to `/.well-known/nostr.json?name=<local>` queries, looking up `(Host, name)` in the `nip05_entries` table
- Caching: edge cache with short TTL (5 min) so revocations propagate quickly

One table, one handler, one DNS record, one cert. Operationally simple.

### Relation to original flat namespace

The old flat namespace (`firstname.familyname@onbitcoinstandard.com`) is no longer privileged. It becomes **one namespace among many**, most likely published as `obs-members.onbitcoinstandard.com` for the core OBS community. Users who originally claimed flat-namespace identifiers (during early development) migrate to a chosen namespace; legacy flat-namespace records remain resolvable to avoid breaking any early adopters.

Pattern becomes: no global naming authority. Every identifier is namespaced.

### LUD-16 Lightning addresses (post-PoC)

NIP-05 format equals LUD-16 format. Same identifier `alice.ross@onbitcoinstandard.com` can double as a Lightning-receive address if we publish `/.well-known/lnurlp/alice.ross` that returns an LNURL-pay callback.

Implementation requires:
- LNURL-pay → Arkade-receive bridge (our server converts an LNURL-pay request into an Arkade VTXO receive for the user)
- Payment routing decisions (user online? route to their VTXO. Offline? Bounce or hold?)
- Fee accounting (LNURL-pay has fee expectations; Arkade has ASP fees)

Meaningful complexity. Defer to post-PoC. When shipped, it gives users a single `alice.ross@onbitcoinstandard.com` identifier that works for:
- Nostr social identity
- Lightning zaps
- Our family wallet contact discovery

## Implementation notes for Q2 / Q3

**Q2:**
- Static `nostr.json` served from `onbitcoinstandard.com`
- Claim API endpoint on `api.onbitcoinstandard.com`: signed-event-based, verifies pubkey ownership
- First implementation: flat namespace, manual reserved list, simple collision flow

**Q3:**
- Privacy defaults for kids per mode
- UI integration: contact cards show NIP-05 verified badge, family invitations include NIP-05 in nprofile
- Dynamic `nostr.json` generation from Postgres (instead of static file) when user count grows

**Post-PoC:**
- Subdomain-per-family option behind a config switch
- LUD-16 Lightning address bridge via Arkade

## Consequences

**Positive:**
- Human-readable identity without introducing accounts or passwords
- Already-trusted format across the entire Nostr ecosystem — our users are "verified" in every Nostr client the moment we publish their entry
- Brand ties to OBS without locking users in (they can always use raw npub)
- Clean domain architecture — each subdomain has its own origin for browser security
- Kid-safe defaults acknowledge the privacy concern
- Future-proof for LUD-16 unified identity

**Negative:**
- We host a name directory — small moderation burden (impersonation complaints, trademark issues)
- Users who lose access to their key lose the NIP-05 identifier (expected, documented, honest consequence of operator-blind)
- Subdomain-per-family deferral means one-word family name collisions possible in v1 (accepted, manageable)
- We must keep the `/.well-known/nostr.json` endpoint operational indefinitely — any downtime breaks NIP-05 lookup for all users; staleness tolerated by cache but not unbounded

## Alternatives rejected

- **Raw npub only** — too unfriendly for non-tech users and kids
- **Custom username format** — NIP-05 already exists, standard, interoperable; no reason to invent
- **Email addresses as identifiers** — requires an email server, PII storage, verification flows, GDPR obligations; NIP-05 avoids all of it
- **Username + password (Nunchuk style)** — defeats no-accounts posture from ADR-0002

## References

- NIP-05: https://github.com/nostr-protocol/nips/blob/master/05.md
- NIP-19: https://github.com/nostr-protocol/nips/blob/master/19.md (nprofile format)
- LUD-16: https://github.com/lnurl/luds/blob/luds/16.md
- ADR-0002 — no passwords, signature-based auth
- ADR-0005 — pubkey is primary identity; multi-family membership
- ADR-0014 — PWA-first, three-subdomain architecture
