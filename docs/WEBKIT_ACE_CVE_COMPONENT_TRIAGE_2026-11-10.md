# Post-Cherry-Pick ACE CVEs — Component Triage (desk, no console)

> **Date:** 2026-11-10 (late)
> **Method:** the cobrajet tier-kill triage law (grade by FIX COMPONENT, not
> advisory prose) applied to the 4 candidates in
> `docs/WEBKIT_CVE_POSTCHERRY_INFOLEAKS_2026-11-10.md` (upstream `4b827f90`).
> Component read from the actual WebKit fix commit/PR via GitHub API.
> LLInt-reachable = fix lives in interpreter / C++ runtime / WebCore paths.

## Results

| CVE | Bugzilla | Fix commit | Component | Verdict |
|-----|----------|-----------|-----------|---------|
| CVE-2026-112659 | 321982 | `6f74c58ae058` — "[JSC] Reallocate cloned tuple nodes' slots in DFG" | `dfg/DFGByteCodeParser.cpp`, `dfg/DFGGraph.cpp` | **DEAD — DFG** (compiled out; `OptionsPlayStation.cmake:206`) |
| CVE-2026-86974 | 321609 | `c983da47ed` — "RTT::jsToWasmICEntrypoint should use DoNotHaveTagRegisters" | `wasm/js/JSToWasm.cpp` | **DEAD — Wasm+JIT** (both compiled out) |
| CVE-2026-113903 | 321652 | `be384ac928` — "Wasm tail call to a same-instance JS import should use a restore frame" | `wasm/WasmCallee.*`, `WasmIPIntPlan.cpp`, `WasmBBQJIT.cpp`, `WasmOMGIRGenerator.cpp`, `llint/InPlaceInterpreter64.asm` | **DEAD — Wasm end-to-end** (no WebAssembly on this port) |
| CVE-2026-43705 | none public | none public (Apple advisories only: 127594/127595/127685/128068-70/148287) | unknown | **UNDECIDABLE at desk** — type confusion (CWE-843), Safari 26.5.2 fix (2026-06-29, post-cutoff ⇒ present in 14.10). Needs the fix commit to triage. |

## Consequences

1. The "high-quality OpenAI/Anthropic report" candidate (112659) is a DFG
   register-allocation bug — unreachable on LLInt-only. The advisory prose
   ("arbitrary code execution") said nothing about tier; the component did.
2. Both 27.2 OOB/ACE rows are Wasm — the entire class is free to prune on
   this port (config-level kill, era-independent per the cobrajet law).
3. CVE-2026-43705 is the last live post-cherry-pick candidate. It becomes
   triage-able the moment its Bugzilla opens or a mapping advisory (e.g.
   WSA-2026-0007) names the commit. Watch, do not build PoCs blind.

## Ledger rule reinforced

Three for three, the advisory prose ("type confusion", "memory corruption",
"arbitrary code execution") never named the tier; the fix path always did.
Component-first triage remains the cheapest filter we have: each verdict
above cost minutes and zero console cycles.

Grades: all rows [S] from public WebKit commits/CVE records; no device rows.
