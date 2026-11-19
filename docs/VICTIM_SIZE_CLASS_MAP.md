# Victim size-class map — object type → size class → post-free reachability

Seeded 2026-11-19 for the #295 cross-bench tri-class effort (cobrajet format: `P693-C<cls>-<HIT|CLEAN>`). BKE = best-known estimate; [S] = source-pinned (616.1.27 clone), [M] = measured on device, [?] = unverified.

## Class 192 — CSSFontFace (CVE-2026-43715 / FontFaceSet::load)

- **Object**: `CSSFontFace` (`RefCounted<CSSFontFace>` @+0x00). Freed while `FontFaceSet::load()` snapshot (non-strong) still iterates.
- **Post-free reachability [M]**: engine continuation writes live pointers into the reclaiming ArrayBuffer — `+0x58` WeakHashSet impl, `+0x68` WeakPtr impl (= L1/L2 on our bench; PR #66 formalized). Readable from JS; templating (rc≥1 @+0x00 + pointers restored) keeps the process alive.
- **Mechanism [M, ours 2026-11-19]**: rc word @+0x00 is the RefCounted counter — engine `deref()` hitting 0 runs `delete this` (clean, measured: `FAM-FR`, no intrusive freelist write). Template rc=2 for one grade of safety margin.
- **Firmware [M-x2]**: alive 14.10; **dead on 14.22** (no 192B reclaim — TFujiwara #67).
- **Era [M]**: per-size-class allocator contest; 37h drought observed on our bench while class-768 stayed golden.

## Class 768 — CanvasRenderingContext2D (CVE-2026-43720) — the row cobrajet asked for

- **Object [S]**: most-derived `CanvasRenderingContext2D` → `CanvasRenderingContext2DBase` → `CanvasRenderingContext` (+ScriptWrappable/RefCounted) + `CanvasPath` mix-in. Freed when the outer `getContext` overwrites `m_context` after the getter's inner call.
- **Read channel owner [S]**: NOT a direct member — the transform/state data lives in **`CanvasRenderingContext2DBase::m_stateStack` (Vector<State,1>)**, a SEPARATE heap allocation from the 768B object. The 768B chunk holds only the Vector header (m_buffer/m_capacity/m_size).
  - Every state read funnels through `state()` = `m_stateStack.last()` (CanvasRenderingContext2DBase.h:310): `getTransform()` → `state().transform` (cpp:847-850); lineWidth/miterLimit/globalAlpha/shadowBlur same pattern.
  - `setTransform` writes `modifiableState()` AND syncs the GraphicsContext; **guards [S]**: `drawingContext()` null → silent no-op; `!isfinite(arg)` → rejected (cpp:852-858).
  - `State` field order [S]: unparsedStrokeColor, unparsedFillColor, strokeStyle, fillStyle (CanvasStyle×2), **lineWidth (double)**, lineCap, lineJoin, miterLimit, shadowOffset (FloatSize), shadowBlur, shadowColor, globalAlpha, 2 enums, **transform (AffineTransform = 6×double)**, hasInvertibleTransform, **lineDash (Vector<double>)**, lineDashOffset, then text/font fields (unparsedFont, FontProxy). ORDER is source truth; byte offsets are MSVC-build-specific → device-measure.
- **Post-free reachability [M]**: YES via the stale inner wrapper — same-type reclaim (sibling context) makes every engine-touched pointer class-identical, so no vtable/rtti hazard; all 12 Canvas2D APIs alias bidirectionally (V630, our Finding 14, cobrajet 6/6 on 14.10, TFujiwara on **14.22**).
- **The ~464B State window [BKE — explains the V650/P682 crashes]**: device model State[8]@buffer+3712 ⇒ sizeof(State) ≈ 464B (8 saves; matches h4len=256-adjacent walk observations). A forged `m_stateStack.m_buffer` makes the engine read a **~464B State window at the target** — any target with less mapped/benign memory around it crashes. This explains: (a) V650 universal crashes (~1400 targets incl. valid bands — targets too small/misaligned for the window), (b) P682 crash at the leaked WeakHashSet impl (~64B object — far smaller than 464B), (c) why `lineWidth` is always the crash point (deepest field read in the window).
- **Correct forge shape [BKE]**: `m_buffer` → attacker-templated region ≥464B with valid Vector-tail fields (`m_capacity`/`m_size` at the header offsets inside the 768B chunk) and sentinel doubles at `State[k].lineWidth/transform` — controlled-content read first, outward steps after. Offset of `m_stateStack` inside the 768B object: **device-measure only** (queued Tuesday: oracle sweep around the alias context; save() N times differential).
- **Firmware [M-x2]**: alive 14.10 **and 14.22** (alias R/W confirmed) — the survives-updates primitive. Needs a pointer source on 14.2x (FontFace leak is patched there; V674 APNG claim unadjudicated).

## Class 80 — (context header?) [?]

Proxy+2d trigger dies by same-size reuse at 80B (CONFIRMED_VULNS note) — the 80B class hosts some context-adjacent allocation; not the primary victim. Unmapped; low priority.

## Measured dead ends (do not re-derive)

- CVE-2026-64783 DataListButtonElement UAF — dead end on pastawithpesto bench (commit 7f2cde46).
- Cross-type reclaim into the 768B canvas slot from ArrayBuffer spray — dead (p554/b/c, different bmalloc size classes).
- Float64 pointer reads through any canvas channel — dead on device (MXCSR DAZ: denormal → 0.0). Write of denormal bit-patterns still reaches memory; read-back of pointer-valued words does not.
- 49 documented dead leak paths — see WEBKIT_FINDINGS.md; oracle (u32-pair) remains our only live arbitrary read.
