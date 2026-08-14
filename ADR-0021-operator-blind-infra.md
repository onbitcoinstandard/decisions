# ADR-0021: Operator-blind infrastructure & chain access

Date: 2026-08-14
Status: Accepted (2026-08-14, reviewed and accepted by Rajesh)
Extends: ADR-0001 (operator-blind custody). Resolves PLAN.md open item "electrs vs esplora". Companion to spec §16 (universality) and §17–§18 (Master v2).

## Context

ADR-0001's operator-blind guarantee covers **custody**: the operator holds only
ciphertext of key material and cannot spend or produce keys. It says nothing about
**observation**. The planned stack (PLAN.md, ARCHITECTURE.md) puts an OBS-hosted
electrs/esplora indexer in every wallet's sync path — an Electrum-protocol server
learns every connecting wallet's addresses, balances, and transaction graph. The
operator would hold no keys yet possess a complete financial record of every user:
maximum surveillance liability with zero custody. The blindness principle must
extend from "cannot spend" to "cannot see", and each infra component needs an
explicit classification.

## Decision

### 1. Blindness matrix — every component OBS operates is classified

| Component | Classification | Mechanism |
|---|---|---|
| Backup vault / sync API | **Blind** | age E2EE (ADR-0004); ciphertext only |
| Transport / relay (PSBT + event shuttling) | **Blind (content)** | NIP-44 / gift-wrapped envelopes; operator sees ciphertext + minimal routing metadata |
| Push notifications | **Blind** | content-free wake pings; payload fetched E2EE |
| Family ASP (arkd + its bitcoind + electrs) | **Not operated by OBS** | each family self-hosts (spec §16.2); OBS ships config, never runs it |
| Inter-family registry | **Deliberately seeing — the documented exception** | sees family-boundary events only (spec §16.3); this is its designed function and matches the reportable-event boundary |
| Chain indexer for user wallets | **OBS never operates one** | see §2 below |

New components default to **Blind**; any exception requires its own ADR.

### 2. Chain access — blind by protocol, not by policy

- **Wallet default sync: compact block filters (BIP-157/158).** The wallet downloads
  per-block filters and matches its own scripts locally; no server ever learns which
  addresses it watches. BDK supports CBF natively. This holds even against
  OBS-operated or malicious nodes — blindness is structural.
- **electrs lives inside the family stack only.** arkd needs an indexer; it gets one
  in the family's self-hosted compose (bitcoind + electrs + arkd + postgres) where
  it sees only that family's own addresses — private by topology.
- **Power-user option:** the wallet may be pointed at the family's own electrs
  (faster, still self-observed). Third-party public Electrum servers are supported
  but the UX labels the privacy cost honestly.
- **Broadcast:** via the family node when present; otherwise over the CBF P2P
  connection — never through an OBS API that would link user → transaction.
- This resolves PLAN.md's open "electrs vs esplora" item sideways: the deciding
  question was never *which* indexer but *whose*. Answer: the family's; never OBS's.

### 3. Honest liability framing (documentation rule)

E2EE + non-custodial + observation-blind means the operator **cannot produce what it
never had** — keys, plaintext, or an address graph. That is liability
*minimization by architecture*, and it is the strongest structural position
available. It is **not** legal immunity: obligations attach per jurisdiction
(spec §16.4 overlays) and require counsel per market. OBS documentation and
marketing must never claim E2EE as immunity — only as minimization. The registry
is the deliberate, documented seeing-point precisely because its events are the
reportable ones.

## Consequences

- No OBS service can answer "which addresses belong to this family" — under
  subpoena, breach, or insider threat — because the knowledge never exists
  server-side. The registry answers only what it was built to record.
- CBF sync is slower than Electrum queries (minutes-scale first sync, seconds
  thereafter); accepted as the default-privacy cost, with family-electrs as the
  fast path.
- OBS runs less infrastructure, not more: no public indexer fleet to scale,
  secure, or answer for.
- PLAN.md / ARCHITECTURE.md must be updated: the shared `electrs/esplora` box moves
  out of the server stack and into the per-family compose.
