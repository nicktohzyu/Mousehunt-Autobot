# Implementation Plan — Refactor `bwRift` (Bristle Woods Rift Bot)

**File:** `MouseHunt AutoBot.js` (single-file userscript, ~15k lines)
**Working directory:** `/home/nicho/mousehunt`
**Branch:** `refactor/bwrift-async`

Refactor `bwRift()` to modernize timing, improve readability, prevent background tab freezing, and clean up configuration.

---

## Codebase Conventions (from AGENTS.md)

| Convention | Detail |
|---|---|
| **Timer/async** | `bgSleep(ms)` (Web Worker) for background-resistant sleep; `sleep(ms)` for standard. Prefer `bgSleep` for game automation to avoid tab throttling. |
| **Variable naming** | Hungarian-like prefixes: `g_nTimeOffset`, `nIndex`, `bToggle`, `objTrapList` |
| **Game state** | Scraped via `getPageVariable()` / `JSON.parse()` from window-level vars |
| **Persistent state** | `localStorage` / `sessionStorage` via `setStorage()` / `getStorage()` wrappers |
| **DOM interaction** | `fireEvent(element, 'click')` for triggering game UI clicks |
| **Debug logging** | `if (debug) console.log(...)` — `debug` is `true` by default at line 429 |
| **Hooks/comments** | Skipping lint, typecheck, build steps — this is a no-bundle single-file script |
| `console.plog` | Does not exist as a real function — typo for `console.log`. All calls are commented-out in bwRift. |

---

## Current Issues

1. **Legacy Timers and Background Throttling**
   - Portal click/confirm/reload uses nested `window.setTimeout` (pre-Phase 1).
   - Pocketwatch toggle uses `setInterval` (pre-Phase 1).
   - Both fail in background tabs due to browser throttling.
   - **Phase 1 status:** ✅ Converted to `bgSleep`/`async`.

2. **Obfuscated Portal Selection Logic**
   - `objPortal` uses parallel arrays (`arrName`, `arrIndex`) instead of structured objects.
   - `Number.MAX_SAFE_INTEGER` used as a sentinel for "exclude this portal".
   - Dead code paths (`storePortalHistory`, frozen portal check, scramble TODO).
   - ~175 lines of interleaved conditionals with no helper decomposition.
   - **Phase 2 status:** ⬜ Not started.

3. **Duplicate Config Definitions**
   - `defaultBWRiftConfig` defined inside `bwRift()` (line 1722) with current defaults.
   - Separate fallback config in `saveBWRift` scope (line 12200) with stale/different defaults.
   - `initControlsBWRift` and `saveBWRift` fall back to the stale version.
   - **Phase 3 status:** ⬜ Not started.

4. **Magic Number Config Indexing**
   - 32-element arrays indexed by compound chamber + curse + bitmask offsets.
   - `master.weapon[11]` means "weapon for ACOLYTE_CHARGING" — unintelligible.
   - **Phase 4 status:** ⬜ Deferred.

5. **No Interface Config Options**
   - Settings hardcoded (`//interface does not work; settings must be done in code`).
   - **Phase 5 status:** ⬜ Deferred.

---

## Review History

These findings came from two independent code reviews:
- **`code-reviewer-deepseek-v4-flash`** — Found C1 (critical: `activate` default was `VACUUM_CHARMS` array instead of `false`), M1 (dead frozen portal check), M2 (dual configs out of sync), M3 (unhandled promise), plus minor items.
- **`code-reviewer-mimo-2.5`** — Confirmed C1 fix, found NEW-1 (missing `return` after `reloadPage()`), NEW-2 (implicit global `bwRiftConfig`), NEW-3 (`nIndex += 16` confusing ordering), NEW-4 (guard `status.split` can throw). Rated all prior issues as ACCEPT/DEFER.

See [issues.md](issues.md) (if present) for full review transcripts, or re-run the reviews with:
```bash
# DeepSeek review:
task --subagent code-reviewer-deepseek-v4-flash --prompt "Review bwRift() in MouseHunt AutoBot.js" 

# MIMO review:
task --subagent code-reviewer-mimo-2.5 --prompt "Review bwRift() in MouseHunt AutoBot.js"
```

---

## Phased Approach

### Phase 1: Async Timer Migration ✅ DONE

**Goal:** Convert legacy `setTimeout`/`setInterval` patterns to async/await with `bgSleep` for background-tab resilience.

#### Changes applied

| Scope | Before | After | Location (current lines) |
|---|---|---|---|
| Function signature | `function bwRift()` | `async function bwRift()` | Line 1716 |
| Portal click → confirm → reload (acolyte) | Nested `window.setTimeout` | `await bgSleep(1000)` × 2 + `reloadPage(); return;` | Lines 2061–2071 |
| Portal click → confirm → reload (regular) | Nested `window.setTimeout` | `await bgSleep(1000)` × 2 + `reloadPage(); return;` | Lines 2080–2090 |
| Pocketwatch toggle | `setInterval` + `clearInterval` | `while` loop + `await bgSleep(1000)` + retry counter | Lines 2192–2201 |
| Caller (unawaited) | `bwRift();` | `bwRift();` (unchanged — acceptable per plan) | Line 792 |
| Debug logging | None | 28× `if (debug) console.log('[BWRift] ...')` at all decision points | Throughout lines 1718–2201 |

**Bug fixes applied during Phase 1:**
- **C1 (critical):** `activate: new Array(32).fill(VACUUM_CHARMS)` → `activate: new Array(32).fill(false)` at line 1729. Was causing pocketwatch toggle on every chamber (truthy array).
- **NEW-1 (major):** Added missing `return;` after `reloadPage()` in non-acolyte portal path (line 2090). Was falling through to `checkThenArm()`/pocketwatch on a dying page. Acolyte path already had `return`.

**Remaining (known, deferred):**
- Caller at line 792 is unawaited — unhandled promise rejections are swallowed. Will be fixed in Phase 2 (item 6) with `.catch()`.
- `nLootRemaining = Number.MAX_SAFE_INTEGER` at lines 1863/1867 still used as sentinel — will be replaced in Phase 2.

#### Verification checklist
- [ ] `async function bwRift()` at line 1716
- [ ] No `window.setTimeout` or `setInterval` calls remain in `bwRift()` (only `bgSleep` + `setTimeout` inside `bgSleep`'s fallback)
- [ ] Portal click → confirm → reload completes in active tab (check console for `[BWRift] Reloading after ...`)
- [ ] Same sequence completes when tab is in background
- [ ] Pocketwatch toggle fires and retries correctly (console: `[BWRift] Pocketwatch toggle: ...`)
- [ ] `activate` default is `false` at line 1729 (not `VACUUM_CHARMS`)

---

### Phase 2: Portal Selection Refactor ⬜ TODO

**Goal:** Readability refactor of portal selection (~lines 1909–2097). No behavior changes — preserve all decision logic.

#### Step-by-step changes

**1. Replace `objPortal` parallel arrays with structured array** (lines 1922–1924)
```js
// Before:
var objPortal = {
    arrName: new Array(n).fill(''),
    arrIndex: new Array(n).fill(Number.MAX_SAFE_INTEGER)
};

// After:
var portals = Array.from({length: n}, () => ({
    name: '',
    priority: Number.MAX_SAFE_INTEGER,
    excluded: false
}));
```
- `arrName[i]` → `portals[i].name`
- `arrIndex[i]` → `portals[i].priority`
- `Number.MAX_SAFE_INTEGER` sentinel → `portals[i].excluded = true`
- `Number.MAX_SAFE_INTEGER - 1` / `Number.MAX_SAFE_INTEGER - 2` sentinels → Boolean `excluded` + a secondary `priority` bump

**2. Extract helper functions** inside `bwRift()` or as module-level (prefer inside for now to limit scope):

| Helper | Purpose | Inputs | Output |
|---|---|---|---|
| `resolvePriorityList(nIndexBuffCurse, nTimeSand)` | Picks normal/enough-sand/cursed priority list | `nIndexBuffCurse`, `nTimeSand` | `arrPriorities` |
| `scoreAcolytePortal(nTotalRSC, nMinRSC, nTimeSand, minTimeSand)` | Check if ACOLYTE should be excluded | RSC count, RSC threshold, sand counts | `excluded: boolean` |
| `resolveALRLCombo(arrAL, arrRL, nTotalASC, nTotalRSC)` | Decide ancient-first vs runic-first | Ancient/Runic cheese totals | Which set to exclude |
| `chooseBestPortal(portals)` | Return index of lowest-priority non-excluded portal | `portals[]` | `nMinIndex` (or -1 if none available) |

**3. M1 — Fix frozen portal check** (line 1954)
```js
// Before (always false — HTMLElement compared to string):
if (classPortalContainer[0].children[i] == 'frozen')

// After:
if (classPortalContainer[0].children[i].classList.contains('frozen'))
```

**4. M3 — Handle unhandled promise** (line 792)
```js
// Before:
bwRift();

// After:
bwRift().catch(e => console.error('[BWRift] Error:', e));
```

**5. Fragile confirmButton selector** (lines 2063, 2082)
```js
// Before:
document.getElementsByClassName('mousehuntActionButton small')[1]

// After:
document.querySelector('.mousehuntActionButton.small.confirm')
// or fallback: Array.from(document.querySelectorAll('.mousehuntActionButton.small'))
//   .find(b => b.textContent.includes('Confirm'))
```

**6. Remove dead code:**
- `storePortalHistory` block (lines 1912–1919) — always `false`
- `if (storePortalHistory)` block at lines 2040–2043
- Commented-out frozen portal check (lines 2031–2038)
- Commented-out scramble TODO (lines 2045–2052)
- Commented-out `console.plog` calls (lines 1869, 2115, 2187)
- Commented-out `//console.log` debug lines throughout

**7. NEW-2 — Proper `bwRiftConfig` declaration**
- Add `var bwRiftConfig;` at file scope near line 1715 (before `bwRift()`)
- Remove the commented-out `var bwRiftConfig = getStorageToObject(...)` at line 1843
- The assignment `bwRiftConfig = defaultBWRiftConfig;` at line 1839 now assigns to the declared variable

**8. Consistent declarations in `bwRift()`**
- Change `var` → `const` for: `defaultBWRiftConfig`, `chambers`, `VACUUM_CHARMS`, `objUser`, `objPortal`/`portals`, `objTemp`, `arrPriorities`, `arrIndices`, `arrAL`, `arrRL`, etc.
- Change `var` → `let` for: `nIndex`, `nLootRemaining`, `strChamberName`, `nIndexBuffCurse`, `nMinIndex`, `nMinRSC`, `bToggle`, `bForce`, `bPocketwatchActive`, etc.

**9. Magic number 5 → named constant**
```js
// At top of bwRift(), near line 1720-ish:
const POCKETWATCH_RETRY_MAX = 5;

// Line 2190:
// Before: var nRetry = 5;
// After:  let nRetry = POCKETWATCH_RETRY_MAX;
```

**10. Remove leftover `nLootRemaining = Number.MAX_SAFE_INTEGER`** at line 1863, 1867 — the sentinel is no longer needed once portals use `excluded` boolean.

#### Verification checklist
- [ ] Before/after: `console.log("Choose portal: ", objPortal)` at line 2039 produces equivalent portal choices
- [ ] Acolyte portal exclusion still works (insufficient RSC / time sand)
- [ ] AL/RL combo resolution still works
- [ ] Cursed-mode portal exclusions still work
- [ ] Frozen portals are actually excluded (not dead code)
- [ ] No unhandled promise rejections from `bwRift()`
- [ ] Confirm button selector still finds the right element
- [ ] No `console.plog` or stale `//console.log` remain in `bwRift()`

---

### Phase 3: Single Config Source of Truth ⬜ TODO

**Goal:** Eliminate duplicate `defaultBWRiftConfig` and make the UI panel reflect runtime defaults.

#### Key locations

| What | Current line | Notes |
|---|---|---|
| Runtime `defaultBWRiftConfig` | 1722 | Inside `bwRift()` — correct defaults |
| Stale fallback `defaultBWRiftConfig` | 12200 | Inside `bodyJS()` — different defaults |
| `saveBWRift` fallback | 14028 | References the stale version |
| `initControlsBWRift` fallback | 14164 | References the stale version |

#### Changes

1. **Keep the runtime config** at line 1722 (inside `bwRift()`) as the single source of truth.

2. **Delete the stale fallback** (lines 12200–12241). Remove the entire `var defaultBWRiftConfig = { ... };` block in `bodyJS()`.

3. **Update `saveBWRift`** (line 14028): when `sessionStorage` has no stored value, reference `bwRiftConfig` (the runtime global assigned at line 1839) instead of the deleted `defaultBWRiftConfig`.

4. **Update `initControlsBWRift`** (line 14164): same change as above.

5. **Clean break:** The existing `sessionStorage` format under key `BWRift` may be stale. Users should reconfigure from scratch via the UI panel after this change. Consider clearing `sessionStorage.removeItem('BWRift')` on first load after this change.

#### Verification checklist
- [ ] Only one `defaultBWRiftConfig` definition exists in the file
- [ ] `saveBWRift` uses runtime config when `sessionStorage` is empty
- [ ] `initControlsBWRift` uses runtime config when `sessionStorage` is empty
- [ ] UI panel populates with values matching runtime behavior
- [ ] Save → reload → settings persist correctly

---

### Phase 4: Config Structure Redesign ⬜ DEFERRED

**Goal:** Replace opaque 32-element arrays with named chamber keys.

**Dependency:** None — the `QuestRiftBristleWoods` page variable structure is already known from existing field accesses in the code (see `objUser.*` usage throughout `bwRift()`). No need to wait for user input.

#### Before → After
```js
// Before:
bwRiftConfig.master.weapon[11]       // "11" means ACOLYTE_CHARGING
bwRiftConfig.master.activate[11]     // unintelligible without the chambers map

// After:
bwRiftConfig.chambers.ACOLYTE_CHARGING.master.weapon
bwRiftConfig.chambers.ACOLYTE_CHARGING.master.activate
```

#### Rewrites

This phase rewrites all config reads/writes:

| Section | Current lines | Scope |
|---|---|---|
| `defaultBWRiftConfig` definition | 1722–1787 | Full rewrite from flat arrays to nested chambers |
| Per-chamber overrides | 1808–1836 | Move into `chambers.X` structure |
| Config reads in trap resolution | 2113–2158 | Replace `nIndex` lookups with `chambers.STRCHAMBERNAME` |
| Config reads in pocketwatch | 2173–2201 | Same |
| `saveBWRift` / `initControlsBWRift` | 13997 / 14129 | Update to new structure (if not already removed in Phase 3) |

**Accepted extra items (from MIMO review):**
- **Stale DOM refs:** Re-query `classLootBooster`/`classButton` inside the pocketwatch retry loop (lines 2192–2200) instead of caching outside. The DOM may re-render during the 1–5s wait window.
- **bgSleep blob URL leak:** Add `URL.revokeObjectURL(blobUrl)` in `bgSleep()` after the Worker is created, or in the `onmessage` handler.
- **NEW-4:** Add null check on `objUser.minigame.guard_chamber.status` before calling `.split('_')` at line 2130.

#### Verification checklist
- [ ] Config reads produce identical values for all chambers (compare console output before/after)
- [ ] Per-chamber overrides still apply
- [ ] No `master.weapon[numericIndex]` patterns remain
- [ ] Pocketwatch loop re-queries DOM each iteration
- [ ] `bgSleep` no longer leaks blob URLs
- [ ] Guard chamber `status.split` is null-safe

---

### Phase 5: Interface Config Options ⬜ DEFERRED

**Goal:** Expose currently hardcoded settings in `bwRift()` through the UI panel.

**Depends on:** Phase 4 (config structure redesign) — the new named-chamber structure will make UI binding straightforward.

**Likely candidates for exposure:**
- `minTimeSand` array (per-curse-index)
- `minRSC` / `minRSCType`
- `enterMinigameWCurse`
- `choosePortalAfterCC`
- Per-chamber: weapon/base/trinket/bait/activate overrides

**Verification checklist:**
- [ ] All settings reflect in `sessionStorage` on save
- [ ] `bwRift()` reads from `sessionStorage` on next invocation
- [ ] Defaults match the runtime config when no UI save exists
