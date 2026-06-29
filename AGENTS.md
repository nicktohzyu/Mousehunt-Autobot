# MouseHunt AutoBot

Single-file userscript (Tampermonkey/Greasemonkey) that automates MouseHunt gameplay.

## Setup

- Install via Tampermonkey/Violentmonkey on `mousehuntgame.com` or `apps.facebook.com/mousehunt`
- Edit settings inline in the script (`// == Basic/Advanced User Preference Setting ==` sections)
- Reload the game page for changes to take effect

## Key facts

- **Single file**: `MouseHunt AutoBot.js` (~15k lines, no modules, no bundler)
- **No build, test, lint, or typecheck** — just edit and reload
- **Entrypoint**: the IIFE at the bottom runs automatically at `document-idle`
- **All configuration is JavaScript variables**, not external files or a UI

## Timer / async patterns

- `bgSleep(ms)` — background-resistant sleep using Web Worker (prefer this for game automation to avoid browser tab throttling)
- `sleep(ms)` — standard `Promise`-based sleep (line 536)
- The codebase is mid-migration from nested `setTimeout`/`setInterval` to async/await + `bgSleep`

## Notable conventions

- Variable naming: Hungarian-like prefixes (`g_nTimeOffset`, `nIndex`, `bToggle`, `objTrapList`)
- Game state scraped via `getPageVariable()` / `JSON.parse()` from window-level vars
- Persistent state stored in `localStorage` / `sessionStorage` via `setStorage()` / `getStorage()` wrappers
- `fireEvent(element, 'click')` for triggering game UI interactions

## Area bot functions

Each game area has an `initControls*` function. Key ones:

| Area | Function | Line |
|---|---|---|
| Labyrinth/Zokor | `initControlsLaby` / `initControlsZokor` | 13009 / 13670 |
| Fungal/Forbidden Grove | `initControlsFGAR` / `initControlsBCJOD` | 13870 / 13910 |
| Fiery Warpath | `initControlsFW` | 13190 |
| Bristle Woods Rift | `initControlsBWRift` + `bwRift` | 14105 / 1716 |
| Floating Islands | `initControlsFRox` | 14315 |
| Whisker Woods Rift | `initControlsWWRift` | 14450 |
| Geyser (GES) | `initControlsGES` | 14627 |
| Iceberg | `initControlsIceberg` | 13807 |
| Sunken City | `initControlsSC` | 12849 |
| Seasonal Garden | `initControlsSG` | 13504 |
| Balack's Cove | `initControlsBR` | 13414 |

## BWRift specifics

- `bwRift()` (line 1716) uses a 32-element config array indexed by state bitmask — magic numbers, no named constants
- Chamber order defined in `defaultBWRiftConfig.order` (line 1722)
- Settings must be edited in code (`//interface does not work; settings must be done in code`)
- Being refactored per `implementation_plan.md`

## Trap configuration

- `objBestTrap` (line 220) — user's preferred weapons/bases by power type (arcane, draconic, forgotten, hydro, law, physical, rift, shadow, tactical)
- `objTrapCollection` — exhaustive lists of all trap items in the game
- `objTrapList` — populated at runtime from DOM
- `bestRiftWeapon`, `bestRiftBase`, etc. — global overrides (lines 238–243)

## Known pitfalls

- **KR (King's Reward)**: auto-solve orchestrated via OCRAD + canvas. If stuck, solve manually so botting detection doesn't trigger.
- **Page reloads**: the bot reloads the page frequently (`reloadPage()`). Make sure game state persists across reloads.
- **Trap check**: runs once per hour at `trapCheckTimeDiff` offset. Arming sequence uses retry logic (`armTrapRetry = 3`).
- **Delay variables**: many randomized delays (`smallDelay`, `extraDelay`, `checkTimeDelay`) — these are anti-detection. Don't set them all to 0.
