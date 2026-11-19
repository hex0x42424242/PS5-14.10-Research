# Cross-bench digest: pastawithpesto/PS5-14.10-Jailbreak-Research (2026-11-19)

Repo commits stale since 10-25, but the ISSUE TRACKER is the live research surface. Verified today.

## 1. Our L1/L2 leak, independently formalized (PR #66, kabangast, OPEN)
- Same leak: freed 192B CSSFontFace reclaimed by ArrayBuffer; engine writes `+0x58` WeakHashSet impl + `+0x68` WeakPtr impl (values match our L1/L2 family exactly; `+0x00` rc=1).
- Survival templating (p564): rc=1 @+0x00 + write leaked pointers back into all 192B buffers → page survives. **Our v34 explains WHY it works**: rc≥1 keeps engine deref from reaching 0 → no delete-this → no free of a live-in-use chunk. We can hand them the exact mechanism (rcw=0 → `delete this` clean run, SESSION §16).
- Constraints measured: one reclaim per WebProcess; write lands ~80ms after getter; family names >256 chars crash; unstable window.

## 2. P682v2 "crash after forge" is a GEOMETRY bug, not a wall (our v31 lesson)
They forged `m_stateStack.m_buffer` → **leaked WeakHashSet impl address**; read crashed ("Canvas2D internal structure at that location doesn't match what lineWidth expects"). Of course: the target is a HashTable, not a State array. Correct design: aim `m_buffer` at a **templated lane we control** (fake State array: valid m_capacity/m_size + sentinel lineWidth @State[8]+64) → controlled-content arb read; then L1/L2-relative engine reads. We can validate on 14.10 with the oracle cross-check — nobody else can.

## 3. Canvas 43720 refinement (docs/CONFIRMED_VULNS.md)
- Correct trigger: **bitmaprenderer outer + 2d inner** (avoids 80B vs 768B size-class reuse that kills Proxy+2d).
- V630: ALL 12 Canvas2D APIs through the alias; `setLineDash(500)/getLineDash` 500-double round-trip PERFECT → the alias dash array is the forge-read vehicle.
- V548–V573: bidirectional alias R/W stable; same-type 768B reclaim works.

## 4. 14.2x intelligence (#63, #67) — strategic
- **14.20 (PS5 Pro, elanonimo832): same engine** (Version/17.0), all APIs present, JIT off. **Offers a free isolated 14.20 bench** to run our probe suite.
- **14.22 (TFujiwara): canvas alias R/W CONFIRMED WORKING** (12 APIs, p553-level). The "14.20 closed Prolapse" scene claim is wrong or partial — the alias primitive survives. FontFace chain PATCHED on 14.22 (no 192B reclaim, no iterateClients crash).
- Consequence: our 14.10-only primitives (oracle u32-pair, engine-driven free) are a **spend-it-or-bank-it** resource on 14.10; the canvas+leak path is the 14.2x road (needs a pointer source — V674 APNG or FontFace leak pre-patch).

## 5. TFujiwara stuck at V650 forge crash-oracle (#67) — we can unstick them
~1400 forged m_stateStack targets, universal crash at the lineWidth read (regions incl. our published bands). Diagnosis: same class as our v31 bug — wrong m_stateStack offset for their build, or target lacking State layout (their all-words forge crashing everywhere says the engine touches the buffer before/independent of the read). **Unique contribution we can make: measure the TRUE Canvas2D m_stateStack offset on 14.10 with the oracle** (read context memory around the alias; save() N times; locate the state array and its allocation size class) → verified offset table + reclaim size class for 14.22.

## 6. V674 APNG overread = unreconciled claim (#70)
README says "ASLR Defeat CONFIRMED — 1,651 heap pointers"; CONFIRMED_VULNS still says "blocker: INFO LEAK". Nobody has adjudicated. Our oracle can: read the region APNG claims to overread and compare content. Worth one load when bench returns.

## 7. Victim-doc input (feeds docs/VICTIM_SIZE_CLASS_MAP.md)
- CSSFontFace: 192B (documented here + #66).
- CanvasRenderingContext2D: 768B (Finding 14 + V573/V630 + their 6/6).
- CVE-2026-64783 DataListButtonElement UAF: **measured DEAD END** on their bench (commit 7f2cde46) — mark it.
- 14.20/14.22 survival table: canvas alive, FontFace dead — per-firmware column.

## Actions queued
1. Review PR #66 from our position (mechanism-level validation of templating via rcw; cross-link SESSION §16). [needs ownerOK]
2. Oracle-measure m_stateStack offset + State array geometry on 14.10 → hand TFujiwara the table.
3. Template-lane arb-read proof (aim m_buffer at our lane) — kills the P682 crash class.
4. Reach out to elanonimo832 for the 14.20 bench (our probe suite, isolated console).
5. V674 adjudication load.
