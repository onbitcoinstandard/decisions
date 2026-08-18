# ADR-0019: Distribution overlays & signer device tiers (PWA stays canonical)

Date: 2026-08-14
Status: Accepted (decided in working session with Rajesh, 2026-08-14)
Amends: ADR-0014 (PWA-first shell). Resolves spec §12 open decision #3 (which hardware tier ships first).

## Context

ADR-0014 chose PWA-first partly on grounds that no longer hold, and partly on grounds
that still do. What changed since April:

1. **Store accounts now exist.** Apple individual ($99, rajesh@medampudi.com) and Google
   Play (ID 5392182772804468106) were verified 2026-08-05..13; an app-factory pipeline
   (React-Native/store tooling) exists for other product lines. "Zero store fees" is moot.
2. **GTM targets non-technical families.** Store distribution is how they find and trust
   apps; PWAs are weak on iOS push/background — real limits for the *online* wallet
   (a signing request from a family member should be a push notification).
3. **Long-offline PWA fragility.** The signer PWA lives in a service-worker cache; on a
   dedicated device offline for months, iOS/Android can purge it — bricking the signer
   until it goes online again (an air-gap break) to reinstall.
4. **Apple guideline 3.1.5** requires crypto-wallet apps to come from developers enrolled
   as an **organization**; our Apple account is individual. iOS store distribution of the
   wallet is blocked until/unless we enroll an org.

What still holds from ADR-0014: stores have pulled Bitcoin self-custody apps before; a
PWA on our own domain is the channel nobody can revoke. That argument is load-bearing
and is retained.

## Decision

### 1. PWA is canonical; everything else is an overlay

The web codebase (Vite + React + TS per ADR-0014) remains the single source. The PWAs at
`wallet.` / `signer.onbitcoinstandard.com` are the canonical, unpullable distribution.
Store and APK builds are **distribution overlays** built from the same code — convenience
channels, never the foundation.

### 2. Wallet (online, watch-only)

- **Android:** native store app via Google Play (app-factory rails, package
  `com.onbitcoinstandard.wallet`). Watch-only = no funds hostage if ever pulled; PWA
  remains the fallback channel.
- **iOS:** installed PWA. Revisit App Store only if we enroll an Apple organization
  (guideline 3.1.5). Watch-only may arguably fall outside "storage" but review risk is
  not worth the critical path.

### 3. Signer (offline, amnesic) — device-tier ladder

| Tier | Artifact | Runs on | Integrity model |
|---|---|---|---|
| **1 · Commodity** | PWA + signed reproducible **sideload APK** (thin wrap of the same build) | any phone, airplane mode | software-level; APK survives SW-cache eviction on long-offline dedicated devices |
| **2 · Dedicated** | **De-radioed GSI** (LineageOS/TrebleDroid-based, one image): connectivity framework, telephony, Wi-Fi/BT/NFC stacks and package-install paths removed | any Treble (Android 9+) device with unlockable bootloader (₹3–8k used) | OS-level: no connectivity or install code exists; vendor radio blobs dormant; bootloader stays unlocked → ritual defenses (spec §18) |
| **3 · Flagship** | Full per-device build: stripped kernel (no radio drivers), deleted radio firmware, **custom AVB key + relocked bootloader** | curated shortlist (used Pixel a-series first; must support custom AVB relock) | hardware-enforced: only our-signed image boots; unlock = forced wipe + warning |

Runtime lockdown for Tier 2/3 builds: no package installer, no store client, no ADB, no
recovery sideload, no networking code. Inputs = power, touch, camera; output = screen.
The QR parser is the only data ingress — keep it minimal (strict UR / BIP-174 only).

### 4. Web installer — `flash.onbitcoinstandard.com`

WebUSB-fastboot installer (fastboot.js; the Android Flash Tool / GrapheneOS-installer
pattern): detects the model, tells the user which tier their device gets, flashes, and
on Tier-3 devices sets our AVB key and relocks. It also serves the **re-flash ritual**
(spec §18) — restoring a known-good, hash-verified image in ~10 minutes.

**The installer is a door, not a lock.** Fastboot is an open protocol; "only installable
via our page" is convenience, not enforcement. Enforcement exists only where the
bootloader is relocked under our key (Tier 3).

### 5. Sequencing

- **Now (event scope, Oct–Dec 2026):** Tier 1 only — signer PWA + sideload APK,
  wallet PWA. Play-store wallet build when ready (starts the account's 12-tester/14-day
  gate; not on the demo critical path).
- **Post-event:** web installer + Tier-2 GSI (first build est. 2–6 weeks part-time);
  Tier-3 flagship after device-support research shortlists relockable models.

## Consequences

- ADR-0014's architecture (origins, stack, storage, WASM crypto) is unchanged; only its
  store-abstinence is superseded.
- Tier 2/3 are a maintained OS artifact — a real ongoing cost, accepted because an
  offline device doesn't chase monthly patches; rebuilds track our releases, not Android's.
- Spec §10 gains the ladder; §12 #3 is resolved; §17–§18 (install channel, physical
  adversary model) are added in Master v2.

## Amendment (2026-08-14, Rajesh): Tier-1 APK declares no INTERNET permission

The signer's Android APK (the Capacitor wrap of the PWA build) ships **without
`android.permission.INTERNET` in its manifest — and without any other network
permission.** Android enforces network access at the OS level per-app: an APK
that never declares the permission cannot open a socket, on **any** device,
stock or custom, rooted or not. This upgrades Tier 1 from "airplane-mode
discipline" to an **OS-enforced air gap on any phone**, with zero flashing.

- The wrap must not include plugins or WebView settings that require network;
  the app loads exclusively from bundled local assets.
- Applies to all native builds (Tier 1 APK, and the signer as baked into
  Tier 2/3 images). The **browser PWA cannot make this claim** — a browser has
  the internet permission; there the §18 rituals and the airplane gate remain
  the control. UX copy must keep the two claims distinct.
- Verification: the reproducible-build hash plus a manifest dump
  (`aapt dump permissions`) lets anyone confirm the APK asks for nothing.

## Amendment (2026-08-14, Rajesh): the signer is Android-only

The signer ships natively on **Android only** — no iOS build, ever. The dedicated-
device story depends on capabilities iOS structurally lacks: sideloading without a
store, dropping the network permission at the OS level, and the Tier-2/3 ROM path.
The signer PWA remains reachable from any browser as the try-it channel. The
**wallet** (and other online apps) do ship iOS from the same web package — this
amendment scopes only the signer.

## Amendment (2026-08-15, Rajesh): no hosted signer — APK-only distribution

The hosted signer PWA is withdrawn. signer.onbitcoinstandard.com is a **landing
page only** (what it is, release verification, card templates, APK download).
Rationale: a signer running in an online browser invites exactly the behavior the
product forbids — seed entry on a networked device. Distribution is the **signed
APK** (and later Tier-2/3 images) exclusively; the web build remains in-repo for
development and as the APK's source, but is never deployed as a runnable app.
The PWA-canonical rule (ADR-0014) continues to apply to the online apps (wallet);
this amendment scopes only the signer. A decommissioning service worker was
deployed at the old /app/ scope to clear cached copies.

## Amendment (2026-08-18, Rajesh): Tier 3 = GrapheneOS, not a custom build; GSI scoped to cheap phones

Two changes to the device-tier ladder, from the realisation that GrapheneOS already
delivers Tier 3 better than we could build it — and only on Pixels.

**Tier 3 is now GrapheneOS + the OBS Signer APK — we build no custom relocked image.**
GrapheneOS (Pixel-only) is a maintained, hardened OS with verified boot + bootloader
relocking under its own keys (hardware-enforced integrity) and a per-app network
permission toggle that denies the app all network at the OS layer. The signer is just
an Android app: flash GrapheneOS → install the OBS Signer APK → deny it network →
airplane mode → relock. This is cleaner and more secure than a home-rolled per-device
build, with zero build/maintenance burden. So the earlier "Tier 3 = full per-device
build + custom AVB key + relock" plan is **withdrawn**; the recommendation is Graphene.
Honest nuance: GrapheneOS keeps the radios physically present (neutralised by verified
boot + no-network-permission + per-app denial + airplane), rather than stripping them
from the build. For every realistic threat model that is sufficient; physical removal
on a Pixel is diminishing returns.

**Tier 2 (the de-radioed GSI) is now explicitly scoped to CHEAP commodity phones only.**
GrapheneOS cannot run on non-Pixel hardware, and the whole OBS thesis is "a $20 old
phone becomes an air-gapped signer." That accessible tier is exactly what the GSI serves
and Graphene cannot. The GSI base is **pure AOSP (aosp_arm64), de-radioed** — minimal and
auditable, deliberately NOT TrebleDroid (its hundreds of vendor-compat patches are audit/
attack surface a signer shouldn't carry, and most of what they fix is the connectivity we
delete). Validated against a **curated device shortlist** we QA ourselves (near-stock
brands — Motorola G-series that unlock officially, Nothing/CMF), not "any phone." For a
signer the only hardware needs are display, touch, and camera; on MediaTek/near-stock
devices those are the parts that work. Tier-2 devices are unlocked-bootloader only
(ritual-secured per spec §18), never relocked — relock lives in Tier 3 (Graphene/Pixel).

**Resulting ladder:**
| Tier | Base | Device | Integrity |
|---|---|---|---|
| 1 · Universal | no-network APK | any phone | software (OS denies the socket) |
| 2 · Accessible | de-radioed pure-AOSP GSI | cheap unlockable (Moto G-official, Nothing/CMF) | OS-level; unlocked → ritual |
| 3 · Hardware-enforced | **GrapheneOS + APK** | Pixel | verified boot + relock |

Supersedes the Tier-3 rows of the original §3 table and the §3.4 flagship-build plan.
