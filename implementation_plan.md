# Implementation Plan - Redesign `bwRift` (Bristle Woods Rift Bot)

This plan outlines the proposed refactoring and redesign of the `bwRift` function in [MouseHunt AutoBot.js](file:///c:/Code/Mousehunt-Autobot/MouseHunt%20AutoBot.js) to modernize timing execution, improve readability, and prevent background tab freezing.

## Current Issues & Areas for Improvement

1.  **Legacy Timers and Race Conditions**:
    *   The portal selection, confirmation, and page reloads are executed via nested `setTimeout` callbacks (e.g., lines 2050–2072).
    *   The pocketwatch activation toggler uses `setInterval` (lines 2171–2177).
    *   Both are subject to browser background throttling and can fail to complete the sequence in inactive tabs.

2.  **Highly Obfuscated Indexing (Magic Numbers)**:
    *   Uses hardcoded config arrays of size 32 (like `master.weapon`) indexed by complex state bitmasks (`nIndexBuffCurse`, e.g., mapping `8` for cursed, `9` for second acolyte run).
    *   Chamber states are represented by mapping strings to numerical keys (`chambers.RUNIC = 3`, etc.).

3.  **No Interface Config Options**:
    *   The settings are completely hardcoded (`//interface does not work; settings must be done in code`).

4.  **Legacy Window Variables Scraping**:
    *   Uses `JSON.parse(getPageVariable('JSON.stringify(user.quests.QuestRiftBristleWoods)'))` which is slow and can be replaced with direct accessing of page variables if they are same-scope, or wrapped in a cleaner retrieval helper.

---

## Proposed Changes

### MouseHunt AutoBot.js

#### [MODIFY] [MouseHunt AutoBot.js](file:///c:/Code/Mousehunt-Autobot/MouseHunt%20AutoBot.js)

We will restructure `bwRift` into a modern, async-await-based workflow:

1.  **Change Function Signature to `async`**:
    *   Convert `function bwRift()` to `async function bwRift()`.

2.  **Replace nested timeouts/intervals with `bgSleep`**:
    *   Refactor the portal click sequence to utilize sequential awaits.
    *   Refactor the pocketwatch `setInterval` logic to use a background-resilient loop with `await bgSleep(1000)`.

3.  **Modernize the Configuration Object**:
    *   Translate the Chamber & Curse/Buff indexing from magic numbers into descriptive object properties (e.g. `config.chambers.ACOLYTE.weapon` rather than `config.master.weapon[11]`).

---

## Redesigned Snippet Preview

### Refactored Async Portal Click Sequence:
```javascript
// Old Nested Timout Code:
// fireEvent(portal, 'click');
// window.setTimeout(function () {
//     fireEvent(confirmButton, 'click');
//     window.setTimeout(function () { reloadPage(); }, 1000);
// }, 1000);

// New Await Code:
fireEvent(classPortalContainer[0].children[nMinIndex], 'click');
await bgSleep(1000);
const confirmButton = document.getElementsByClassName('mousehuntActionButton small')[1];
if (confirmButton) {
    fireEvent(confirmButton, 'click');
}
await bgSleep(1000);
reloadPage();
```

### Refactored Pocketwatch Activator:
```javascript
if (bToggle) {
    let nRetry = 5;
    while (nRetry > 0) {
        if (classLootBooster.getAttribute('class').indexOf('chamberEmpty') < 0) {
            fireEvent(classButton, 'click');
            break;
        }
        await bgSleep(1000);
        nRetry--;
    }
}
```

---

## Verification Plan

### Automated/Manual Testing
1.  Verify the script loads without syntax errors.
2.  Enable debug logging and monitor the portal-clicking behavior in Bountiful Beanstalk/Bristle Woods Rift.
3.  Simulate background tab behavior while inside the Bristle Woods Rift and confirm that portals are entered and pocketwatches are toggled successfully without stalling.
