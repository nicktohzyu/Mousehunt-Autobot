# Implementation Plan - Refactor `bwRift` (Bristle Woods Rift Bot)

Refactor `bwRift` in [MouseHunt AutoBot.js](MouseHunt%20AutoBot.js) to modernize timing, improve readability, prevent background tab freezing, and clean up configuration.

## Current Issues

1. **Legacy Timers and Background Throttling**
   - Portal click/confirm/reload uses nested `window.setTimeout` (lines 2050–2072).
   - Pocketwatch toggle uses `setInterval` (lines 2171–2177).
   - Both fail in background tabs due to browser throttling.

2. **Obfuscated Portal Selection Logic**
   - `objPortal` uses parallel arrays (`arrName`, `arrIndex`) instead of structured objects.
   - `Number.MAX_SAFE_INTEGER` used as a sentinel for "exclude this portal".
   - Dead code paths (`storePortalHistory`, frozen portal check, scramble TODO).
   - ~175 lines of interleaved conditionals with no helper decomposition.

3. **Duplicate Config Definitions**
   - `defaultBWRiftConfig` defined inside `bwRift()` (line 1721) with current defaults.
   - Separate fallback config in `saveBWRift` scope (line 12176) with stale/different defaults.
   - `initControlsBWRift` and `saveBWRift` fall back to the stale version.

4. **Magic Number Config Indexing**
   - 32-element arrays indexed by compound chamber + curse + bitmask offsets.
   - `master.weapon[11]` means "weapon for ACOLYTE_CHARGING" — unintelligible.

5. **No Interface Config Options**
   - Settings hardcoded (`//interface does not work; settings must be done in code`).

---

## Phased Approach

### Phase 1: Async Timer Migration ✅

**Scope:** `bwRift()` function signature, portal click sequence, pocketwatch toggle.

| What | Before | After |
|---|---|---|
| Function signature | `function bwRift()` | `async function bwRift()` |
| Portal click → confirm → reload | Nested `window.setTimeout` | Sequential `await bgSleep(1000)` |
| Pocketwatch toggle | `setInterval` + `clearInterval` | `while` loop with `await bgSleep(1000)` |

The caller at line 792 (`bwRift();`) leaves the returned promise unhandled — acceptable for now.

**Debug logging:** Added `if (debug) console.log('[BWRift] ...')` throughout the function at key decision points:
- Function entry
- Chamber detection and state (name, index, loot remaining, time sand)
- Curse/buff index determination
- Priority list selection (normal, enough-sand, cursed)
- Acolyte portal exclusion (insufficient RSC or time sand)
- AL/RL combo resolution (ancient-first vs runic-first)
- GUARD/FROZEN/INGRESS exclusion under curse
- Trap setup resolution (chamber, index)
- Trap arming (full objTemp dump)
- Pocketwatch decision (active, activate, force, toggle)
- Pocketwatch execution (waiting, clicked, retries exhausted)
- Portal click sequences (chosen portal, confirm, reload)

**Untouched:** Portal selection logic (structure), trap resolution logic, config structure, UI code.

**Plus:** Fixed missing `return` after `reloadPage()` in non-acolyte portal entry path — was falling through to trap arming/pocketwatch logic on a dying page (Matches acolyte path which already had `return`).

**Test:** Set `debug = true` (already true by default at line 428). Enter BWRift with a portal open. Check browser console for `[BWRift]` prefixed logs. Verify:
1. Click → confirm → reload sequence completes in active tab
2. Same sequence completes when tab is in background
3. Pocketwatch toggle fires and retries correctly

---

### Phase 2: Portal Selection Refactor

**Scope:** Lines 1905–2081 (~175 lines). Readability refactor only — no behavior changes.

Changes:
1. Replace `objPortal` parallel arrays with `portals[]` array of `{name, priority, excluded}` objects.
2. Extract helper functions:
   - `resolvePriorityList(nIndexBuffCurse, nTimeSand)` — picks normal/enough-sand/cursed priority list.
   - `scoreAcolytePortal(portal, nTotalRSC, nMinRSC)` — returns whether to exclude ACOLYTE.
   - `resolveALRLCombo(arrAL, arrRL, nTotalASC, nTotalRSC)` — decides ancient-first vs runic-first.
   - `chooseBestPortal(portals)` — returns index of lowest-priority, non-excluded portal.
3. Remove dead code: `storePortalHistory` block, frozen portal check (line 2020), scramble TODO (line 2034), commented-out `console.plog` calls.
4. Replace `Number.MAX_SAFE_INTEGER` sentinel with a boolean `excluded` field.

**Plus (from MIMO review):**
5. **M1** — Fix frozen portal check: use `children[i].classList.contains('frozen')` instead of `== 'frozen'` string comparison (always false).
6. **M3** — Add `.catch()` at call site: `bwRift().catch(e => console.error('[BWRift] Error:', e))` at line 792.
7. **Fragile selector** — Use `querySelector('.mousehuntActionButton.small.confirm')` or text-content match instead of `[1]` index.
8. **NEW-2** — Add proper `var bwRiftConfig;` declaration at file scope, remove the commented-out declaration at line 1843.
9. **Consistent declarations** — Convert all `var` in `bwRift()` to `const`/`let`.
10. **Magic number 5** — Extract to `const POCKETWATCH_RETRY_MAX = 5` at top of function.
11. **Commented-out console calls** — Remove all dead `console.plog` and `//console.log` debug lines.

**Test:** Same as Phase 1. Compare `console.log("Choose portal: ", ...)` output before/after to verify identical decision-making.

---

### Phase 3: Single Config Source of Truth

**Scope:** Lines 12176–12241 (stale UI fallback), lines 14136–14140 (`initControlsBWRift` default), lines 14002–14004 (`saveBWRift` default).

`saveBWRift` and `initControlsBWRift` are mostly vestigial — the user runs with inline config defaults and rarely uses the UI panel. Changes:

1. Keep `defaultBWRiftConfig` locally scoped inside `bwRift()`.
2. Delete the duplicate fallback at line 12176.
3. In `initControlsBWRift` (line 14139) and `saveBWRift` (line 14003): when `sessionStorage` has no stored value, use `bwRiftConfig` (the runtime global) as the fallback — which is already assigned `defaultBWRiftConfig` at line 1838.
4. Clean break: existing `sessionStorage` format may become stale; user reconfigures from scratch.

**Test:** Open settings panel → verify controls populate from defaults → save → verify `sessionStorage` written → reload → verify saved values persist.

---

### Phase 4: Config Structure Redesign (deferred)

Replace 32-element arrays with nested structure keyed by chamber name:

```js
// Before: bwRiftConfig.master.weapon[11]
// After:  bwRiftConfig.chambers.ACOLYTE_CHARGING.master.weapon
```

This phase rewrites:
- `defaultBWRiftConfig` definition
- All config reads in `bwRift()` (trap resolution lines 2083–2140, pocketwatch lines 2145–2178)
- `saveBWRift` and `initControlsBWRift` (if still in use)
- Per-chamber overrides (lines 1808–1836)

**Plus (from MIMO review):**
- Re-query `classLootBooster`/`classButton` inside pocketwatch retry loop (stale DOM refs after re-render).
- Add `URL.revokeObjectURL(blobUrl)` in `bgSleep()` to prevent slow blob-URL leak.
- **NEW-4** — Add null check on `guard_chamber.status` before `.split('_')` to prevent crash.

---

### Phase 5: Interface Config Options (deferred)

Expose settings currently hardcoded in `bwRift()` through the UI panel. Depends on Phase 4.
