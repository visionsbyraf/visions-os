# VISIONS OS — Update Patch v1.4

You have already built VISIONS OS as a single self-contained HTML file. Do NOT rebuild it from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality that is not mentioned in this patch.

There are two updates in this patch. Both apply exclusively to Daily tasks where **`category === 'ACTIVE'`** and **`arcId !== null`** (i.e. linked to an Arc). Tasks in SKILL, ADMIN, or NONE categories, or unlinked ACTIVE tasks, retain their existing behavior unchanged.

---

## CONDITION HELPER

Throughout this patch, the phrase **"qualifying task"** means a Daily object where both of the following are true:
- `daily.category === 'ACTIVE'`
- `daily.arcId !== null`

All behavior changes below apply only to qualifying tasks. Add this check as a reusable inline condition wherever needed:

```js
function isQualifyingTask(daily) {
  return daily.category === 'ACTIVE' && daily.arcId !== null;
}
```

---

## UPDATE 1 — QUADRANT: Done Qualifying Tasks Do Not Return to Dailies

### 1a. Current behavior (do not keep for qualifying tasks)
When any task is marked done from a quadrant card, it is marked `status: 'done'` in `visions-dailies` and remains in the Dailies Bank (visually struck through at 40% opacity).

### 1b. New behavior for qualifying tasks marked done FROM THE QUADRANT

When a qualifying task's quadrant card is marked **done** (`✓` button on the quadrant card):

- Log the completion entry in `visions-log` as normal (action: `'done'`, zone, timestamp, week).
- Update the Arc's progress calculation as normal.
- **Remove the Daily object from `visions-dailies` entirely.** Do not set `status: 'done'`. Delete the object from the array and save.
- Remove the task's id from the linked Arc's `linkedDailyIds` array in `visions-arcs` and save. (The arc itself is not deleted — only the link is cleaned up.)
- Remove the quadrant card from the zone as normal.
- Re-render the Dailies Bank so the task no longer appears.
- Show a toast: `"Task complete — removed from bank."` (green, 3s).

**Non-qualifying tasks marked done from the quadrant** retain existing behavior (strikethrough, 40% opacity, stays in bank).

---

### 1c. New behavior for qualifying tasks SKIPPED or REMOVED from the quadrant

No change. Skipping or removing a qualifying task from the quadrant returns it to the Dailies Bank as before (with carry count logic from patch v1.3 applied). This behavior is unchanged.

---

### 1d. New behavior: REMOVE from the Dailies Bank (task never on quadrant)

**Current behavior (do not keep for qualifying tasks):** Clicking `✕` on a Daily card in the bank removes it immediately and silently.

**New behavior for qualifying tasks removed directly from the Dailies Bank:**

When `✕` is clicked on a qualifying task in the Dailies Bank:

- **Do not delete immediately.**
- Show an inline confirmation prompt directly below the task card (same pattern as the Arc delete confirmation in patch v1.1). Do not use a modal or browser alert.
- The prompt reads:

  ```
  Remove "[Task Name]" permanently? This cannot be undone.
  ```

  With two buttons: `CONFIRM DELETE` (red, `#a82b1e` background) and `CANCEL` (muted, `#2a2a2a` border).

- If **CANCEL**: dismiss the prompt, no changes.
- If **CONFIRM DELETE**:
  - Remove the Daily object from `visions-dailies` permanently.
  - Remove the task's id from the linked Arc's `linkedDailyIds` in `visions-arcs`.
  - Log a removal entry in `visions-log`: `{ action: 'removed', zone: null, ... }`.
  - Re-render the Dailies Bank.
  - Show a toast: `"Daily permanently removed."` (amber, 3s).

**Non-qualifying tasks removed from the bank** retain the existing silent-remove behavior (no confirmation prompt).

---

## UPDATE 2 — DONE FROM DAILIES BANK (Never on Quadrant): Passive Completion Logging

### 2a. Current behavior (do not keep for qualifying tasks)
Clicking `✓` on a Daily card in the bank marks it `status: 'done'` (strikethrough, 40% opacity). It stays in the bank.

### 2b. New behavior for qualifying tasks marked done directly from the Dailies Bank

When `✓` is clicked on a qualifying task **in the Dailies Bank** (i.e. the task is not currently placed in any quadrant zone — check `visions-quadrant` to confirm the task id is absent from all zones):

- **Remove the Daily object from `visions-dailies` entirely** (same as the quadrant-done path).
- Remove the task's id from the linked Arc's `linkedDailyIds` in `visions-arcs`.
- Log a completion entry in `visions-log`:
  ```json
  {
    "action": "done",
    "zone": null,
    "passiveCompletion": true,
    "timestamp": "ISO datetime",
    "week": "2026-Wxx"
  }
  ```
  The `passiveCompletion: true` flag distinguishes this from a quadrant-driven done action.
- Re-render the Dailies Bank.
- Show a toast: `"Logged as done — removed from bank."` (green, 3s).

**If the task IS currently placed in a quadrant zone** when `✓` is clicked from the bank: do not apply this behavior. In that case, the `✓` action from the bank should instead behave as if `✓` was clicked on the quadrant card directly (delegate to the quadrant done handler for that task). This prevents double-processing.

**Non-qualifying tasks marked done from the bank** retain existing behavior (strikethrough, 40% opacity, stays in bank).

---

### 2c. Debrief Section — PASSIVE COMPLETIONS

Add a new subsection inside the existing **COMPLETION RATE** section of the Debrief (not a new top-level section — place it directly below the Done/Skipped/Removed stat row).

**Title:** `PASSIVE COMPLETIONS` — Space Mono, 9px, uppercase, `#555`, letter-spacing 1.5px.

**Content:** A list of tasks logged with `passiveCompletion: true` during the current debrief week. For each entry show:

```
[Task Name]  ·  linked to [Arc Name]  ·  [date, e.g. MON 19]
```

Format: Space Mono, 11px. Task name in `#f0ece3`. "linked to [Arc Name]" in `#555`. Date in `#444`.

Below the list (or in place of it if no passive completions exist), show one of the following editorial notes in italic `#555`, 10px:

- If 1–2 passive completions: `"Some tasks got done off-quadrant — good instinct, but consider placing them in a zone next time for better tracking."`
- If 3+ passive completions: `"Several tasks were completed outside the quadrant this week. This may indicate cruise-mode work habits — productive, but harder to track and attribute."`
- If 0 passive completions: `"All completions went through the quadrant this week."` (no italic, just muted)

**In Debrief history snapshots** stored in `visions-debrief`, add a `passiveCompletions` array field to the snapshot object, containing the passive completion log entries for that week (dailyId, dailyName, arcId, arcName, timestamp).

---

## DATA MODEL NOTES

No new localStorage keys are introduced in this patch.

Fields added to existing structures:
- `visions-log` entries: optional `passiveCompletion: true` boolean on applicable entries (absent on all other entries — treat absence as `false`).
- `visions-debrief` snapshot objects: optional `passiveCompletions: []` array field.

The `visions-dailies` array shrinks as qualifying tasks are completed or permanently removed — this is intentional. Unlike non-qualifying tasks which accumulate in the bank with done/skipped states, qualifying tasks are consumed on completion.

---

## WHAT NOT TO CHANGE

- Do not modify behavior for SKILL, ADMIN, or NONE category tasks, or for ACTIVE tasks with `arcId === null`
- Do not change the Drift timer logic
- Do not change drag-and-drop behavior
- Do not change the carry count or inactivity prompt logic from patch v1.3
- Do not change Arc delete or Daily unlink behavior from patch v1.1
- Do not change the Morning Sync format
- Do not modify the Settings panel beyond what is described
- Do not change typography, spacing, or color of any existing elements not mentioned in this patch
- Do not simplify or omit any existing features

---

**Output:** The complete updated single HTML file with both updates applied. Same file format as original — all CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts in `<head>`. Immediately usable by double-clicking on a Mac.**
