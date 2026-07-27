# VISIONS OS — Update Patch v1.2

You have already built VISIONS OS as a single self-contained HTML file. Do NOT rebuild it from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality that is not mentioned in this patch.

There are five updates in this patch.

---

## UPDATE 1 — TOPBAR: Move Settings to Top Bar as Gear Icon

### Remove the SETTINGS tab from the tab bar entirely.

In the HTML, remove the `<button class="tab-btn" data-tab="settings">SETTINGS</button>` element from `#tabbar`.

Remove the `<div id="tab-settings" class="tab-panel">` block from `#main` (keep all its internal content — it will be reused, see below).

### Add a gear icon button to the top bar.

**Placement:** In `#topbar`, insert a `⚙` gear button directly to the right of `#morning-sync-btn`, before the end of the topbar div.

**Style the gear button as follows:**
```css
#settings-gear-btn {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-primary);
  font-size: 16px;
  width: 34px;
  height: 34px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border-radius: 3px;
  transition: all 0.2s;
  margin-left: 8px;
  line-height: 1;
}
#settings-gear-btn:hover { border-color: #555; background: #1a1a1a; }
```

**Topbar right side:** Wrap `#morning-sync-btn` and the new `#settings-gear-btn` in a small flex container div so they sit side by side:
```html
<div style="display:flex;align-items:center;gap:0">
  <button id="morning-sync-btn">☀ MORNING SYNC</button>
  <button id="settings-gear-btn">⚙</button>
</div>
```

### Settings now open as a slide-in panel (not a tab).

**Add a new overlay panel** to the HTML, just before the closing `</body>` tag (after `#toast-container`):

```html
<div id="settings-panel-overlay">
  <div id="settings-panel">
    <div id="settings-panel-header">
      <span id="settings-panel-title">⚙ SETTINGS</span>
      <button id="settings-panel-close">✕</button>
    </div>
    <div id="settings-panel-body">
      <!-- paste the full inner contents of the old tab-settings div here -->
    </div>
  </div>
</div>
```

The inner contents of `#settings-panel-body` are identical to what was inside `<div id="tab-settings" class="tab-panel">` — the `.settings-section` div with all color editor rows and buttons.

**Style the panel:**
```css
#settings-panel-overlay {
  display: none;
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  z-index: 8500;
}
#settings-panel-overlay.open { display: block; }
#settings-panel {
  position: fixed;
  top: 0;
  right: 0;
  width: 420px;
  max-width: 95vw;
  height: 100vh;
  background: #141414;
  border-left: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  z-index: 8501;
  animation: slideInRight 0.25s ease;
}
@keyframes slideInRight {
  from { transform: translateX(100%); }
  to   { transform: translateX(0); }
}
#settings-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border);
  flex-shrink: 0;
}
#settings-panel-title {
  font-family: 'Space Mono', monospace;
  font-size: 12px;
  letter-spacing: 2px;
  color: var(--text-primary);
  text-transform: uppercase;
}
#settings-panel-close {
  background: none;
  border: none;
  color: var(--text-secondary);
  font-size: 14px;
  cursor: pointer;
  padding: 4px 6px;
  border-radius: 2px;
  transition: color 0.2s;
}
#settings-panel-close:hover { color: var(--text-primary); }
#settings-panel-body {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
  scrollbar-width: thin;
  scrollbar-color: #2a2a2a transparent;
}
#settings-panel-body::-webkit-scrollbar { width: 4px; }
#settings-panel-body::-webkit-scrollbar-thumb { background: #2a2a2a; }
```

**Behavior (JS):**

- Remove the `if (tab === 'settings') { renderSettings(); }` line from the `switchTab()` function.
- Remove `'settings'` from all tab-switching logic. The SETTINGS tab no longer exists as a tab.
- Add a click handler on `#settings-gear-btn` that adds the class `open` to `#settings-panel-overlay` and calls `renderSettings()`.
- Add a click handler on `#settings-panel-close` that removes the class `open` from `#settings-panel-overlay`.
- Add a click handler on `#settings-panel-overlay` itself: if `e.target === the overlay element` (i.e. clicking the dark backdrop outside the panel), remove `open` to dismiss.
- Add `Escape` key support: in the existing `keydown` handler, also remove `open` from `#settings-panel-overlay` when Escape is pressed.
- The `renderSettings()` function already exists and wires up the color inputs — no changes needed to its internal logic. It will now be called when the panel opens instead of when the settings tab is switched to. The `settingsInitialized` guard already prevents double-wiring, so this is safe.

---

## UPDATE 2 — TODAY: Collapsible Dailies Sidebar

### Make the Dailies Bank panel collapsible.

When collapsed, the Dailies panel shrinks to a narrow vertical strip and the Quadrant panel expands to fill the full remaining width.

**Add a collapse toggle button** to the Dailies panel header row. Place it to the right of the "DAILIES" heading, inside a flex row wrapper.

Replace the current plain `<div class="panel-header">DAILIES</div>` with:
```html
<div class="dailies-panel-header-row">
  <div class="panel-header" style="margin:0">DAILIES</div>
  <button id="dailies-collapse-btn" title="Collapse panel">◀</button>
</div>
```

**Add CSS:**
```css
.dailies-panel-header-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
  flex-shrink: 0;
}
#dailies-collapse-btn {
  background: none;
  border: 1px solid var(--border);
  color: var(--text-secondary);
  font-size: 10px;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border-radius: 3px;
  transition: all 0.2s;
  flex-shrink: 0;
  font-family: 'Space Mono', monospace;
}
#dailies-collapse-btn:hover { border-color: #555; color: var(--text-primary); }

#dailies-panel.collapsed {
  width: 32px !important;
  min-width: 32px !important;
  overflow: hidden;
}
#dailies-panel.collapsed .dailies-panel-header-row {
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 8px;
  margin-bottom: 0;
}
#dailies-panel.collapsed .panel-header {
  writing-mode: vertical-rl;
  text-orientation: mixed;
  transform: rotate(180deg);
  font-size: 14px;
  color: var(--text-secondary);
  white-space: nowrap;
  margin: 0;
}
#dailies-panel.collapsed > *:not(.dailies-panel-header-row) {
  display: none;
}
#dailies-panel.collapsed #dailies-collapse-btn {
  transform: rotate(180deg);
}
```

**Behavior (JS):**

Add a click handler on `#dailies-collapse-btn`:
- Toggle the class `collapsed` on `#dailies-panel`.
- Save the collapsed state to `visions-settings` in localStorage as `settings.dailiesCollapsed = true/false`. Save immediately on toggle.
- On app load (inside `init()`), after loading settings, check `settings.dailiesCollapsed` — if `true`, add the `collapsed` class to `#dailies-panel` on initial render.

The layout already uses `flex: 1` on `#quadrant-panel`, so it will automatically expand to fill the freed space when the Dailies panel collapses — no additional layout changes needed.

---

## UPDATE 3 — SKIP BEHAVIOR: Skipped Tasks Reappear on the Quadrant Next Day

### Current behavior (do not keep):
Skipped tasks are returned to the Dailies Bank with a `↩ carried` tag and reappear only in the bank.

### New behavior:
When a task is skipped (either from the bank or from a quadrant card), it should be **automatically re-placed into the same quadrant zone it was in** when the app is next opened the following day, in addition to carrying over in the bank.

**Changes to the data model:**

On each quadrant card object stored in `visions-quadrant`, add a field `skippedZone` (string or null). This is already part of the quadrant state object — no new localStorage key needed.

**Changes to `handleQCardAction` (skip branch):**

When a card is skipped from the quadrant, before removing it from the zone array, record its zone. Instead of fully removing the card from `quadrantState.zones[zone]`, move it to a new array `quadrantState.skippedCards` (an array at the top level of `quadrantState`):
```js
if (!quadrantState.skippedCards) quadrantState.skippedCards = [];
card.skippedZone = zone;
card.status = 'skipped';
quadrantState.skippedCards.push(card);
// Then splice from the active zone as before
```

**Changes to `handleDailyAction` (skip branch from bank):**

When a task is skipped from the Dailies Bank (not from the quadrant), check if it's currently in any zone. If it is, apply the same `skippedCards` logic as above (move from zone to `skippedCards`). If it's not in any zone, store a skipped card entry with `skippedZone: 'locked'` as the default zone to return it to (Locked In), so it re-appears somewhere useful.

**Changes to `resetQuadrant()`:**

After the existing reset logic clears the zones, add a step that processes `quadrantState.skippedCards`:
```js
(quadrantState.skippedCards || []).forEach(card => {
  const targetZone = card.skippedZone || 'locked';
  if (!quadrantState.zones[targetZone]) quadrantState.zones[targetZone] = [];
  // Reset the card to active status for the new day
  card.status = 'active';
  card.isDrift = false;
  card.driftStartTime = null;
  card.driftExpired = false;
  quadrantState.zones[targetZone].push(card);
});
quadrantState.skippedCards = [];
```

This means on the next day's first open, skipped cards automatically reappear in the zone they were skipped from (or Locked In if skipped from the bank). The `↩ carried` tag will still show on the Daily card in the bank as before.

---

## UPDATE 4 — THE RUNWAY: Editable Arc Names

### Make Arc names inline-editable.

**Current behavior:** Arc names are rendered as a static `<span class="arc-name">` inside the Arc card header.

**New behavior:** Replace the static span with a contenteditable element.

In `renderArcs()`, replace:
```html
<span class="arc-name">${arc.name}</span>
```
with:
```html
<span class="arc-name arc-name-editable" contenteditable="true" data-arcid="${arc.id}" spellcheck="false">${arc.name}</span>
```

**Add CSS:**
```css
.arc-name-editable {
  outline: none;
  border-bottom: 1px solid transparent;
  transition: border-color 0.2s;
  cursor: text;
  min-width: 40px;
  border-radius: 2px;
  padding: 0 2px;
}
.arc-name-editable:hover { border-bottom-color: #444; }
.arc-name-editable:focus { border-bottom-color: #888; background: rgba(255,255,255,0.04); }
```

**Behavior (JS):**

Add event listeners on each `.arc-name-editable` element after rendering:
- On `blur`: read `el.textContent.trim()`. If non-empty and different from `arc.name`, update `arc.name` in the `arcs` array and call `save(KEYS.arcs, arcs)`. Do NOT call `renderArcs()`. Show a toast: `"Arc renamed."` (green, 2s). **Do not modify `arc.linkedDailyIds` under any circumstances during this operation.**
- On `keydown`: if `Enter` is pressed, call `el.blur()` to commit and `preventDefault()` to avoid inserting a line break.
- Drag events on the arc card must not fire when the user is clicking into the name to edit. Add `e.stopPropagation()` on `mousedown` inside the editable span to prevent drag from interfering with text selection.

**Ensuring linked dailies are not affected:** The save operation on blur only writes `arc.name`. It does not call `renderArcs()` and does not touch `linkedDailyIds`. The arc description textarea already saves via its own `blur` handler in the existing code — confirm it only saves `arc.description` and does not touch `linkedDailyIds`. No change to description behavior is needed unless found to be incorrect.

---

## UPDATE 5 — NEW DAILY FORM: Add "None" Category Option

### Add a "None" option to the category dropdown in the New Daily form.

In the HTML, find the `<select class="form-select" id="nd-category">` element inside `#new-daily-form`. Add a `None` option as the first item:

```html
<select class="form-select" id="nd-category">
  <option value="NONE">None</option>
  <option value="ACTIVE">ACTIVE</option>
  <option value="SKILL">SKILL</option>
  <option value="ADMIN">ADMIN</option>
</select>
```

**Category badge rendering:** In `renderDailies()` and `renderQCard()`, the category badge uses `cat-${d.category}` for styling. Add a CSS rule for the NONE category:
```css
.cat-NONE { background: #1e1e1e; color: #555; }
```

**Filter behavior:** Tasks with category `NONE` appear when the `ALL` filter is active but not when SKILL, ACTIVE, or ADMIN filters are selected. This is already the default behavior of the existing filter logic (exact category match), so no change to filter logic is needed.

**Data:** The Daily object's `category` field stores the string `"NONE"` for these tasks. No other data model changes needed.

---

## VISUAL / STYLE NOTES FOR NEW ELEMENTS

All new UI elements must match the existing app's aesthetic:
- Font: Space Mono throughout
- Background: `#111` or `#1a1a1a` for inputs and surfaces
- Borders: `1px solid #2a2a2a`
- Text: `#f0ece3` primary, `#666` secondary
- Buttons follow existing icon-btn and text-button patterns — no new design patterns
- Animations: 200–250ms ease for transitions
- No new external libraries, no new CDN links

---

## WHAT NOT TO CHANGE

- Do not modify the Debrief tab in any way
- Do not alter the Morning Sync button behavior or format
- Do not change the drag-and-drop behavior for Dailies or Arc reordering
- Do not change the Drift timer logic
- Do not change any existing localStorage keys — only read/write `skippedCards` within the existing `visions-quadrant` key, and `dailiesCollapsed` within the existing `visions-settings` key
- Do not change typography, spacing, or color of any existing elements not mentioned in this patch
- Do not simplify or omit any existing features

---

**Output:** The complete updated single HTML file with all five updates applied. Same file format as original — all CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts in `<head>`. Immediately usable by double-clicking on a Mac.**
