# VISIONS OS — Correction Patch v1.4.1

This patch corrects a single behavior introduced in patch v1.4. Do NOT rebuild from scratch. Apply only the changes described below.

---

## CORRECTION — Completed Tasks Remain Linked to Their Arc

### What patch v1.4 specified (incorrect — revert this):

Both Update 1b (done from quadrant) and Update 2b (done from Dailies Bank) instructed:

> "Remove the task's id from the linked Arc's `linkedDailyIds` array in `visions-arcs` and save."

**Remove that step entirely from both handlers.** Do not unlink the Daily from its Arc on completion under any circumstance.

---

### Correct behavior:

When a qualifying task is marked done — whether from the quadrant card or from the Dailies Bank — the Arc's `linkedDailyIds` array is **not modified**. The task id remains in the Arc's linked list permanently.

---

### Visual effect in THE RUNWAY (Arc detail view):

In the expanded Arc detail view, the LINKED DAILIES list already renders each linked Daily by id. Update this rendering so that completed tasks (Daily objects with `status: 'done'`, or Daily objects that no longer exist in `visions-dailies` because they were consumed on completion) are displayed differently from active ones:

- If the Daily object **exists in `visions-dailies`** and `status` is active/skipped: render normally.
- If the Daily object **no longer exists in `visions-dailies`** (consumed on completion — removed from the array per v1.4): render the task name with **strikethrough text and 40% opacity**, using the last known name. To preserve the name after the object is deleted, store a snapshot in the log entry at the time of completion:

  When writing the `visions-log` entry for a qualifying task completion, add the field:
  ```json
  "dailyName": "Task name at time of completion"
  ```
  This field should already be added in v1.4 — confirm it is present. If not, add it now.

  In the Arc detail view, when iterating `arc.linkedDailyIds`, for each id:
  1. Look up the Daily in `visions-dailies`.
  2. If found: render normally (with unlink button as per patch v1.1).
  3. If not found: look up the most recent `visions-log` entry where `dailyId === id` and `action === 'done'`. Use `entry.dailyName` as the display name. Render as:
     ```html
     <span style="text-decoration:line-through;opacity:0.4;font-style:italic;">[dailyName]</span>
     <span style="font-size:9px;color:#1d7044;margin-left:6px;">✓ done</span>
     ```
  4. If not found in either place: render as `<span style="opacity:0.25;font-style:italic;">[removed]</span>`.

  Do not show the unlink button (`✕`) for completed (struck-through) or removed tasks — only for active ones.

---

### Debrief record:

The `passiveCompletions` array in the Debrief snapshot (introduced in v1.4, Update 2c) should continue to log passive completions by `dailyId` and `dailyName` as specified. No change needed here — the fix above ensures the Arc linkage is preserved, which means Arc-level attribution in the Debrief remains accurate without any additional changes.

---

### Arc progress calculation:

The progress calculation scans `visions-log` for completions of linked Daily ids in the past 30 days. Since the Daily id remains in `arc.linkedDailyIds` and the log entry is written on completion, Arc progress continues to increment correctly. No change to progress logic is needed.

---

## WHAT NOT TO CHANGE

- Do not alter any other behavior from patch v1.4
- Do not change the permanent deletion flow (CONFIRM DELETE from the bank) — that path does not involve a done action and is unaffected
- Do not change carry count, drift, or any other existing logic
- Do not change the unlink button behavior for active tasks (patch v1.1) — only suppress the unlink button for completed/removed tasks as described above

---

**Output:** The complete updated single HTML file with this correction applied on top of all previous patches. Same file format — all CSS in `<style>`, all JS in `<script>` at bottom of body. Immediately usable by double-clicking on a Mac.**
