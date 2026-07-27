# VISIONS OS — Patch v1.5b.1

You have already built VISIONS OS as a single self-contained HTML file (v1.5b). Do NOT rebuild from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality not mentioned in this patch.

There are two updates in this patch.

---

## CONDITION CHANGE — Expand "Qualifying Task" to All Arc-Linked Tasks

Throughout the existing codebase, a "qualifying task" is defined as:
```js
function isQualifyingTask(daily) {
  return daily.category === 'ACTIVE' && daily.arcId !== null;
}
```

**Replace this definition with:**
```js
function isQualifyingTask(daily) {
  return daily.arcId !== null;
}
```

This single change extends all qualifying task behavior — permanent removal on done, passive completion logging, arc detail display — to SKILL, ADMIN, and NONE category tasks as long as they are linked to an Arc. ACTIVE arc-linked tasks are unaffected; they already qualified. Tasks with no Arc link (`arcId === null`) are completely unaffected regardless of category.

Apply this updated definition everywhere `isQualifyingTask` is called or its condition is inlined. Do not change any other logic — only the condition itself.

---

## UPDATE 1 — DONE BEHAVIOR: Extend Passive Completion to All Arc-Linked Tasks

No additional code changes are needed for the done path beyond the condition change above. The done behavior — permanent removal from `visions-dailies`, `passiveCompletion: true` log entry, arc detail struck-through display — was already fully implemented in v1.4 and v1.5b for qualifying tasks. Expanding the condition automatically extends it to SKILL and ADMIN arc-linked tasks.

Confirm the following remain intact after applying the condition change:

- Qualifying task marked **done from the bank** (not on quadrant): Daily object removed from `visions-dailies`. Log entry written with `action: 'done'`, `passiveCompletion: true`, `dailyName: daily.text`. Arc's `linkedDailyIds` unchanged. Arc detail shows struck-through entry at 40% opacity with `✓ done` label. Debrief PASSIVE COMPLETIONS section includes the entry.

- Qualifying task marked **done from the quadrant**: Same permanent removal and log entry. `passiveCompletion: false` (it was on the quadrant). Arc's `linkedDailyIds` unchanged. Arc detail shows same struck-through display.

If either of these paths has a category check that would block SKILL or ADMIN tasks from entering the flow, remove that check now.

---

## UPDATE 2 — REMOVAL BEHAVIOR: Split Intent Prompt for Arc-Linked Tasks

### Overview

When any arc-linked task (`arcId !== null`) is removed — from either the Dailies Bank or the Quadrant — the existing simple confirmation or silent removal is replaced by a two-option intent modal. The user chooses how the removal should be treated relative to the Arc.

This replaces:
- The existing inline confirmation prompt for qualifying task bank removal (v1.4 Update 1d)
- The existing silent return-to-bank behavior for qualifying task quadrant removal (v1.4 Update 1c)

Non-arc-linked tasks (`arcId === null`) retain their existing removal behavior in both bank and quadrant — no changes for those.

---

### 2a. Split Intent Modal — HTML & CSS

Inject the modal element once on init, reuse it across invocations. Do not create a new element per removal action.

**HTML** (inject into `<body>` on init, hidden by default):

```html
<div id="arc-remove-overlay" style="display:none;">
  <div id="arc-remove-modal">
    <div id="arc-remove-task-name"></div>
    <div id="arc-remove-arc-name"></div>
    <div id="arc-remove-body">
      This task is linked to an Arc. Choose how to remove it.
    </div>
    <div id="arc-remove-actions">
      <div class="arc-remove-option" id="arc-remove-unlink">
        <div class="arc-remove-option-title">UNLINK & DELETE</div>
        <div class="arc-remove-option-desc">Removes the task entirely. No record kept in the Arc.</div>
      </div>
      <div class="arc-remove-option" id="arc-remove-archive">
        <div class="arc-remove-option-title">ARCHIVE IN ARC</div>
        <div class="arc-remove-option-desc">Removes from your bank and quadrant. Kept as an archived entry in the Arc's history.</div>
      </div>
    </div>
    <button id="arc-remove-cancel">CANCEL</button>
  </div>
</div>
```

**CSS:**

```css
#arc-remove-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.6);
  z-index: 9100;
  display: flex;
  align-items: center;
  justify-content: center;
}
#arc-remove-modal {
  background: #1a1a1a;
  border: 1px solid #2a2a2a;
  border-radius: 6px;
  padding: 22px 24px;
  width: 380px;
  max-width: 92vw;
  font-family: 'Space Mono', monospace;
}
#arc-remove-task-name {
  font-size: 13px;
  color: #f0ece3;
  font-weight: 700;
  margin-bottom: 3px;
}
#arc-remove-arc-name {
  font-size: 9px;
  color: #1a5c8a;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  margin-bottom: 12px;
}
#arc-remove-body {
  font-size: 10px;
  color: #555;
  line-height: 1.6;
  margin-bottom: 16px;
}
#arc-remove-actions {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 14px;
}
.arc-remove-option {
  border: 1px solid #2a2a2a;
  border-radius: 4px;
  padding: 11px 14px;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}
.arc-remove-option:hover {
  background: rgba(255,255,255,0.03);
  border-color: #444;
}
.arc-remove-option.selected {
  border-color: #f0ece3;
  background: rgba(240,236,227,0.05);
}
.arc-remove-option-title {
  font-size: 10px;
  color: #f0ece3;
  letter-spacing: 1px;
  text-transform: uppercase;
  margin-bottom: 4px;
}
#arc-remove-unlink .arc-remove-option-title { color: #a82b1e; }
#arc-remove-archive .arc-remove-option-title { color: #1d7044; }
.arc-remove-option-desc {
  font-size: 9px;
  color: #555;
  line-height: 1.5;
}
#arc-remove-cancel {
  background: none;
  border: none;
  color: #444;
  font-family: 'Space Mono', monospace;
  font-size: 9px;
  letter-spacing: 1px;
  text-transform: uppercase;
  cursor: pointer;
  padding: 0;
  transition: color 0.15s;
}
#arc-remove-cancel:hover { color: #888; }
```

---

### 2b. Modal Logic

Write a single function `showArcRemoveModal(daily, sourceContext)` where `sourceContext` is either `'bank'` or `'quadrant'`. This function:

1. Populates `#arc-remove-task-name` with `daily.text`.
2. Looks up the linked Arc by `daily.arcId` in `arcs`. Populates `#arc-remove-arc-name` with `"↗ " + arc.name` (or `"↗ Arc"` if not found).
3. Shows `#arc-remove-overlay` (set `display: 'flex'`).
4. Attaches one-time click handlers to `#arc-remove-unlink`, `#arc-remove-archive`, and `#arc-remove-cancel`.

**On `#arc-remove-unlink` click (UNLINK & DELETE):**
- Remove the Daily object from `visions-dailies`.
- Remove `daily.id` from the linked Arc's `linkedDailyIds` in `visions-arcs`.
- Write a log entry: `{ action: 'removed', dailyId, dailyName: daily.text, arcId: daily.arcId, zone: sourceContext === 'quadrant' ? currentZone : null, timestamp, week }`.
- Save `visions-dailies`, `visions-arcs`, `visions-log`.
- If `sourceContext === 'quadrant'`: remove the card from the zone and re-render the quadrant.
- Re-render the Dailies Bank.
- Show toast: `"Task removed and unlinked from Arc."` (amber, 3s).
- Hide the modal.

**On `#arc-remove-archive` click (ARCHIVE IN ARC):**
- Remove the Daily object from `visions-dailies`.
- **Do NOT modify** the linked Arc's `linkedDailyIds`. The id remains.
- Write a log entry: `{ action: 'archived', dailyId, dailyName: daily.text, arcId: daily.arcId, zone: sourceContext === 'quadrant' ? currentZone : null, timestamp, week }`.
- Save `visions-dailies`, `visions-log`.
- If `sourceContext === 'quadrant'`: remove the card from the zone and re-render the quadrant.
- Re-render the Dailies Bank.
- Show toast: `"Task archived in Arc."` (green, 3s).
- Hide the modal.

**On `#arc-remove-cancel` click:**
- Hide the modal. No changes.

**On `Escape` key while modal is open:**
- Hide the modal. No changes.

Only one handler set should be active at a time. Use one-time event listeners (`{ once: true }`) or explicitly remove listeners before attaching new ones, so that rapid successive removal actions do not stack handlers.

---

### 2c. Wire the Modal into Existing Remove Handlers

**Bank removal (`✕` button on Daily card):**

Find the existing remove handler for Daily cards in the bank. For arc-linked tasks (`daily.arcId !== null`):
- Remove any existing inline confirmation prompt logic (the v1.4 "Remove permanently?" prompt).
- Replace with: `showArcRemoveModal(daily, 'bank')`.

For non-arc-linked tasks (`daily.arcId === null`): existing behavior unchanged (silent immediate removal).

**Quadrant card removal (`✕` button on quadrant card):**

Find the existing remove handler for quadrant cards. For arc-linked tasks (`daily.arcId !== null`):
- Remove the existing behavior of returning the task to the Dailies Bank.
- Replace with: `showArcRemoveModal(daily, 'quadrant')`.
  - Pass the current zone name as a variable so the log entry can capture it (store it in a closure or a module-level variable before calling the function).

For non-arc-linked tasks (`daily.arcId === null`): existing behavior unchanged (returns to Dailies Bank as active).

**Right-click context menu DELETE (v1.7):**

If the right-click context menu DELETE action exists on Daily cards, apply the same arc-linked check:
- Arc-linked task: `showArcRemoveModal(daily, 'bank')`.
- Non-arc-linked: existing behavior.

**Batch delete (v1.7):**

The batch delete confirmation banner (v1.7 Update 2e) processes multiple tasks at once. For batch operations, the split intent modal is NOT shown per-task. Instead, update the banner text to reflect the new distinction:

- If any selected tasks are arc-linked: add a line to the banner:
  ```
  Arc-linked tasks will be archived in their Arc (not deleted from Arc history).
  ```
  Batch removal of arc-linked tasks defaults to the **ARCHIVE IN ARC** path — no individual prompt per task. This keeps batch operations fast without per-item interruption.
- Log each arc-linked task with `action: 'archived'` and each non-arc-linked task with `action: 'removed'`.

---

### 2d. Arc Detail View — Archived Entry Display

In the Arc detail expanded view, the LINKED DAILIES list already handles three states (from v1.5b Update 1):
1. Found in `visions-dailies` → render normally with unlink button
2. Not found, log entry `action: 'done'` → struck-through 40% opacity + `✓ done`
3. Not found anywhere → `[removed]` at 25% opacity

**Add a fourth state:**

When looking up a `dailyId` in the log and finding the most recent entry has `action: 'archived'`:

```html
<span style="opacity:0.3; font-style:italic; text-decoration:line-through;">[dailyName]</span>
<span style="font-size:9px; color:#8a6200; margin-left:6px;">· archived · [date]</span>
```

Where `[date]` is formatted from the log entry's timestamp as `DD MMM` (e.g. `07 JUN`).

No unlink button for archived entries. The visual distinction from done entries:
- Done: 40% opacity, standard strikethrough, green `✓ done` label
- Archived: 30% opacity, italic strikethrough, amber `· archived · [date]` label
- Removed (unlinked): 25% opacity, no label, plain italic `[removed]`

---

## DATA NOTES

No new localStorage keys. The `action: 'archived'` value is new in `visions-log` entries — all existing log reading code that checks for `action === 'done'` or `action === 'removed'` is unaffected. Add handling for `'archived'` wherever log entries are grouped or counted in the Debrief generation function — archived tasks should appear as their own category in the Stalled Tasks section (if it exists) rather than being silently dropped.

---

## WHAT NOT TO CHANGE

- Non-arc-linked task removal behavior in bank and quadrant
- Skip behavior in bank and quadrant
- The v1.3 carry count and inactivity prompt (fires independently on carryCount >= 3, unaffected by this patch)
- Done behavior from bank or quadrant (covered by Update 1 condition change only)
- Arc progress calculation logic
- Morning Sync, Settings panel, Debrief tab structure
- Drag-and-drop behavior
- Typography, spacing, or colors of any existing elements not described above
- No external libraries

---

**Output:** The complete updated single HTML file with both updates applied on top of v1.5b. All CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts and favicon `<link>` tags in `<head>`. Immediately usable by double-clicking on a Mac.**
