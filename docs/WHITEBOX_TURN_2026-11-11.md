# White-Box Turn — WebKit 616.1.27 Reference Tree

> **Date:** 2026-11-11 (00:00)
> **What changed:** the bench now has the era-exact source. Every layout
> question moves from reload-oracle to source-read + compiler offsetof.
> Clone: `C:\Users\renat\Projects\.wk-reference` (shallow @ WebKit-7616.1.27,
> the family cobrajet's drop prints in Version.xcconfig).

## Q1 — CSSFontFace layout (derived + device-anchored)

Declaration order (CSSFontFace.h:183-207), x64 Itanium ABI, NDEBUG arm:

| Offset | Member | Size | Evidence |
|---|---|---|---|
| +0x00 | RefCounted base (m_refCount + bools) | ~0x10 | device rc law (add-time +1) |
| ... | properties/families/ranges/features/... | ... | — |
| +0x58 | `m_clients.m_set.m_table` (8B union) | 8 | **device-measured (T)** |
| +0x60 | `m_operationCountSinceLastCleanup` | 4+pad | derived; **device reads 0 post-adopt — open contradiction** |
| +0x68 | `m_wrapper` (WeakPtr 8B) | 8 | **device-measured (W), BRANCH-B** |
| +0x70 | `m_fontSelectionCapabilities` (3×optional<Range>) | ~0x30 | derived |
| **~+0xA0** | **`m_status` (uint8_t)** | 1 | **derived — replaces +0x82 (falsified)** |
| +0xA1.. | bool pack + Timer | ~0x28 | derived |

NDEBUG arm confirmed in source: HashTable.h:61-66 — CHECK_HASHTABLE_ITERATORS=0
under NDEBUG ⇒ HashTable is ONLY the m_table union (8B). No iterators, no
mutex. The other bench's CHECKED-arm map cannot exist in this era at all.

## Q2 — Status byte: the real consumers

- `setStatus` (CSSFontFace.cpp:571-628): iterates clients with
  `fontStateChanged(*this, m_status /*OLD*/, newStatus)` and commits
  `m_status = newStatus` AFTER notification — the 322233 reentrancy window,
  visible in source, unfixed in this era.
- `pump` (cpp:644+): reads `status()`, early-outs on Failure.
- The `fonts.load()` skip is wrapper-level (p656m [M]) — consistent: the
  load path consults the wrapper's settled promise, not the backing byte.

## Q3 — WeakHashSet in this era

Template `<T, WeakPtrImpl, assertionsPolicy>`; members: `m_set`
(HashSet<Ref<WeakPtrImpl>>) + `m_operationCountSinceLastCleanup` (mutable
unsigned). `add()` calls `amortizedCleanupIfNeeded()` on EVERY add
(WeakHashSet.h:101,188-192) ⇒ opCount must increment on adopt — device
measured 0 twice. **Open contradiction, resolvable by compiling offsetof.**

## Next actions

1. Compile layout_probe.cpp (`.wk-layout/`) with MinGW against the reference
   tree: print sizeof/offsetof for CSSFontFace, WeakHashSet<Client>,
   WeakPtrImpl, HashTable metadata geometry (the 80B table).
2. p664 (self-address forge) with the corrected member map — status now
   ~+0xA0, consumers are setStatus/pump, not load().
3. Watch: WSA-2026-0007 (CVE-2026-43705 mapping), n3onStride7 replies.

## Provenance

Source reads [S@616.1.27]; device anchors [M] from p656b/k/n/p663 sessions.
No compilation performed yet — derivations above are declaration-order
arithmetic anchored on two measured offsets.
