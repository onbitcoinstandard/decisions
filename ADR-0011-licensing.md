# ADR-0011: Licensing — MIT client, AGPL-3.0 server

Date: 2026-04-15
Status: Accepted

## Context

We're open-sourcing the project (rationale: no defensible IP in the code, trust / audit / sovereignty benefits far outweigh closed-source's theoretical protection, bitcoin-community norms). The remaining question is which open-source license for each component.

Two components with different incentive structures:

- **Client (apps/client/)** — runs on users' devices. Forking it doesn't let anyone compete with us; they'd still need users, trust, and an ASP. Permissive licensing maximizes adoption and trust.
- **Server (apps/server/)** — runs our hosted ASP + API. A permissive license would let a competitor clone, run it as a closed-source SaaS at scale, benefit from our code without contributing back. AGPL addresses this.

## Decision

**Client code: MIT License.** Anyone can use, fork, modify, distribute, sublicense, or rebrand. Attribution required. No requirement to share modifications.

**Server code: AGPL-3.0.** Anyone can use, fork, modify, distribute. BUT: running a modified version as a network service requires publishing the source of modifications to users of that service. Prevents "take our code, run a closed-source hosted clone" patterns while still allowing legitimate self-hosting (Uncle Jim-style, even if we aren't packaging it in v1).

**Shared packages (`packages/shared/`, `packages/shared-rust/`):** MIT, matching the client — they're the cryptographic primitives and event schemas that we want maximally reusable.

**Documentation (`docs/`):** CC-BY-SA-4.0 so anyone can fork the docs with attribution and share-alike, but can't proprietize them.

## Component license summary

| Directory | License |
|-----------|---------|
| `apps/client/` | MIT |
| `apps/server/` | AGPL-3.0 |
| `packages/shared/` | MIT |
| `packages/shared-rust/` | MIT |
| `infra/arkade/` | AGPL-3.0 (configs + operator scripts travel with the server code) |
| `docs/` | CC-BY-SA-4.0 |
| `tools/` | MIT |

Each directory gets its own `LICENSE` file. Root `LICENSE` file documents the split and references the per-directory files.

## Why MIT for client (not Apache-2.0)

Apache-2.0 has an explicit patent-grant clause, stronger legal protections. MIT is shorter, more widely understood, and has no patent language.

For our specific case:
- We hold no patents we're granting
- We're not concerned about patent attacks from forks (we're the smaller party; nothing to defend)
- MIT's brevity and familiarity in the bitcoin ecosystem matters more than Apache's additional clauses
- Many bitcoin-adjacent projects use MIT (Bitcoin Core, BDK, rust-bitcoin, nostr-sdk — though licenses vary)

If patent concerns arise later, we can relicense future client versions (though the existing MIT grant is permanent).

## Why AGPL-3.0 for server (not MIT, not SSPL, not BSL)

**Not MIT:** would let a well-funded entity clone, run our server closed-source at scale, erode our operator position without contributing upstream.

**Not SSPL:** MongoDB's license, has reputation issues (OSI rejected it, treated as non-open-source in many contexts), would alienate the open-source community without meaningfully strengthening our position over AGPL.

**Not BSL (Business Source License):** converts to open after N years. In the interim it's source-available, not open-source. The bitcoin-sovereignty audience treats BSL with suspicion ("not really free"). Works for databases (CockroachDB, Sentry), doesn't fit here.

**AGPL-3.0:** the standard copyleft-for-network-services license. Widely understood. Tested. OSI-approved. Used by Nextcloud, Mastodon, GitLab Community, Mattermost, many self-hosted ecosystem projects. Exactly the right match.

**Concern with AGPL:** some companies have policies prohibiting AGPL use. Doesn't matter here — we don't want closed-source companies integrating our server code anyway. The policy exclusion is a feature, not a bug.

## Contributions

Any contributions to the repo are licensed under the component's license by default (Developer Certificate of Origin — `Signed-off-by` line in commits). No CLA. Contributors retain copyright; we don't hoard it.

Rationale: CLAs add friction, suggest future relicensing ambition, and are unnecessary for a project that isn't planning an exit. DCO gives us enough legal footing.

## Re-licensing protection

Neither license allows us to unilaterally relicense existing contributions under a different license without every contributor's consent (the permanent-grant nature of both MIT and AGPL). This is desirable — it means a future acquirer or pressured decision cannot silently move the project to a closed license.

If we ever need to relicense (e.g. fixing an incompatibility), we either get contributor consent or rewrite the affected code.

## Trademark — separate from license

The code is open. The name "obs-family-wallet" and OBS branding are ours. Forks may copy the code under the license terms but not the brand, the domain, or the wordmark. We register the relevant trademarks before public launch.

A fork can exist as "Alice's Family Wallet" or whatever else; it cannot call itself obs-family-wallet or imply our endorsement.

## Third-party dependencies

Before first public push, run `cargo deny` + `license-checker` to ensure no dependency is incompatible with our license grants:
- MIT / Apache-2.0 / BSD-* compatible with both MIT and AGPL-3.0 outputs
- GPL-3.0 (without the network clause) is compatible with AGPL-3.0 but contaminates MIT components — would need isolation
- Any proprietary or ambiguously-licensed dependencies get replaced

Reproducible builds (ADR-0002) already require dependency pinning, so license audit is a natural add.

## Public-push checklist

Before the repo goes public (planned: Q3 2026 before the Geyser campaign):

- [ ] Root `LICENSE` file with split explanation
- [ ] `LICENSE` files in each directory
- [ ] License headers in source files where required (AGPL components must have them; MIT components may)
- [ ] `NOTICE` file attributing major upstream projects
- [ ] `CONTRIBUTING.md` explaining DCO and which license contributions fall under
- [ ] `cargo deny` passes in CI
- [ ] Trademark registered

## Consequences

**Positive:**
- Trust / auditability / sovereignty benefits of open source in full
- AGPL on server deters closed-source commercial forks while allowing legitimate use
- MIT on client maximizes reach + trust
- No CLA, no acquirer-friendly license structure — aligned with "not planning an exit"

**Negative:**
- Some organizations' policies exclude AGPL — they'll not use our server code (acceptable; they're not our target)
- AGPL compliance when we run our own modified server is our own obligation (we publish our fork source — easy since our repo IS our fork source)
- Trademark registration costs a few hundred dollars per jurisdiction; budgeted

## References

- MIT License text: https://opensource.org/licenses/MIT
- AGPL-3.0 text: https://www.gnu.org/licenses/agpl-3.0.en.html
- CC-BY-SA-4.0: https://creativecommons.org/licenses/by-sa/4.0/
- DCO: https://developercertificate.org/
