# VISIONS OS — Patch v1.5b

You have already built VISIONS OS as a single self-contained HTML file (v1.4). Do NOT rebuild from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality that is not mentioned in this patch.

There are four updates in this patch.

---

## UPDATE 1 — THE RUNWAY: Completed Tasks Stay Linked to Their Arc

### Current behavior (do not keep):

When a qualifying task (`category === 'ACTIVE'` and `arcId !== null`) is marked done — from either the quadrant card or the Dailies Bank — the task's id is removed from the linked Arc's `linkedDailyIds` array in `visions-arcs`.

### New behavior:

When a qualifying task is marked done under any circumstance, **do not remove its id from the Arc's `linkedDailyIds` array**. The task id must remain in the Arc's linked list permanently. The Arc object's `linkedDailyIds` is never modified as a result of a done action.

**Find every done handler that contains this line (or equivalent):**
```js
arc.linkedDailyIds = arc.linkedDailyIds.filter(id => id !== daily.id);
```
or:
```js
arc.linkedDailyIds.splice(arc.linkedDailyIds.indexOf(daily.id), 1);
```

**Remove that line entirely** from the done handler. Leave all other logic in the handler untouched.

---

### Visual update — Arc detail expanded view (LINKED DAILIES list):

In the expanded Arc detail view, the LINKED DAILIES list iterates `arc.linkedDailyIds` and looks up each id in `visions-dailies`. Update this rendering so that tasks which no longer exist in `visions-dailies` (because they were consumed on completion) are displayed with a visual completion indicator rather than being silently skipped or erroring.

For each id in `arc.linkedDailyIds`, apply this lookup logic:

```js
// 1. Look up the Daily in the live dailies array
const daily = dailies.find(d => d.id === id);

if (daily) {
  // Active task — render normally with unlink button (existing behavior)
} else {
  // Task was completed and consumed — look up its name from the log
  const logEntry = log
    .filter(e => e.dailyId === id && e.action === 'done')
    .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp))[0];

  const displayName = logEntry?.dailyName || '[completed task]';

  // Render as struck-through completion record — NO unlink button
  // HTML:
  // <span style="text-decoration:line-through;opacity:0.4;font-style:italic;">[displayName]</span>
  // <span style="font-size:9px;color:#1d7044;margin-left:6px;">✓ done</span>
}
```

If a task id is found in neither `dailies` nor `log` (hard-deleted), render as:
```html
<span style="opacity:0.25;font-style:italic;">[removed]</span>
```
No unlink button for this row either.

**Ensure `dailyName` is written to log entries at completion time.** In the done handler (both quadrant and bank paths), when writing the `visions-log` entry for a qualifying task, include the field:
```json
"dailyName": "task name string at moment of completion"
```
Read this from `daily.text` before any state change is applied.

**Arc progress calculation:** No change needed. The progress calculation scans `visions-log` for completions of linked Daily ids in the past 30 days. Since the id remains in `arc.linkedDailyIds` and the log entry is still written, Arc progress continues to increment correctly.

---

## UPDATE 2 — DAILIES BANK: Paused Arc Dailies Sort to Bottom

### Overview

When an Arc's status is `'paused'`, all Daily tasks linked to that Arc sort to the bottom of the Dailies Bank list under all active filters. This sort is render-time only — it does not modify the stored order in `visions-dailies`.

### Implementation

**In `renderDailies()`**, after the existing filter logic produces the filtered array and before cards are rendered to the DOM, add a sort step:

```js
// Build a Set of arc ids that are currently paused
const pausedArcIds = new Set(
  arcs.filter(a => a.status === 'paused').map(a => a.id)
);

// Partition into non-paused and paused groups — preserve order within each group
const activeDailies = filteredDailies.filter(d => !pausedArcIds.has(d.arcId));
const pausedDailies  = filteredDailies.filter(d =>  pausedArcIds.has(d.arcId));

// Render order: active first, paused last
const orderedDailies = [...activeDailies, ...pausedDailies];
```

Use `orderedDailies` in place of `filteredDailies` when building the card DOM elements.

---

### Visual treatment for paused-linked cards

When building a Daily card where `pausedArcIds.has(daily.arcId)` is true, add the CSS class `paused-arc-daily` to the card root element.

**Add CSS:**
```css
.paused-arc-daily {
  opacity: 0.55;
}
.paused-arc-daily .arc-tag {
  color: #8a6200;
  border-color: #8a6200;
}
.paused-suffix {
  font-size: 8px;
  color: #8a6200;
  letter-spacing: 0.08em;
  margin-left: 3px;
}
```

In the Arc tag element rendered on the card, append a `· PAUSED` suffix:

```html
<span class="arc-tag">
  ↗ [Arc Name]<span class="paused-suffix"> · PAUSED</span>
</span>
```

This sort and visual treatment applies under all filter states (ALL, SKILL, ACTIVE, ADMIN). When an Arc is un-paused, the next `renderDailies()` call automatically returns those tasks to their normal position.

---

## UPDATE 3 — DEBRIEF: Dynamic Arc Progress List

### Problem

The Arc progress section in the Debrief is built from a static or initialization-time Arc list. It does not include Arcs added after first launch, and does not remove Arcs that have been deleted.

### Fix

**In the Debrief generation function**, replace any static or cached Arc list with a live read of `visions-arcs` at the moment the Debrief is generated or rendered.

**When generating a new Debrief snapshot** (Sunday auto-generate or manual "Run Debrief"):

- Read the current `arcs` array from `visions-arcs` at generation time.
- Include all Arcs present in `visions-arcs` at that moment — both pre-loaded and user-added.
- Store Arc progress data in the snapshot's `arcProgress` array using each Arc's `id` as the key, alongside the Arc name at snapshot time:

```json
{
  "arcId": "uid",
  "arcName": "name at snapshot time",
  "progressPct": 34,
  "progressDelta": 8
}
```

**When rendering a saved Debrief snapshot:**

- Cross-reference the snapshot's `arcProgress` entries against the current `visions-arcs`.
- If an Arc id from the snapshot no longer exists in `visions-arcs` (deleted after the snapshot): do not render that row. Remove it silently from the displayed list.
- If an Arc exists in `visions-arcs` but was not in the snapshot (added after the snapshot): it will appear in the next generated Debrief. Do not back-fill it into old snapshots.

**When rendering the current week's live Debrief** (not a historical snapshot):

- Read directly from the live `visions-arcs` array.
- Calculate progress delta vs. last week's snapshot for Arcs that appear in both.
- For Arcs with no prior snapshot entry (newly added), show `★ NEW` instead of a delta arrow.

**Arc progress delta display — add a fourth state:**

The existing format is:
```
↑ +8%  Sound Design
—  0%  CV/Resume
↓ -2%  Visual Portfolio
```

Add this new row type for new Arcs:
```
★ NEW  [Arc Name]
```

Style: `★ NEW` in `#1a5c8a` (accent blue), Space Mono 10px, uppercase.

---

## UPDATE 4 — DEBRIEF: Extended Data Capture (Export-Ready)

### Overview

Add four new fields to the Debrief snapshot object stored in `visions-debrief`. These fields are captured silently at generation time. There is no new UI — this is purely structural data enrichment to enable a future export function to serialize the full snapshot without additional data gathering.

Existing Debrief snapshots generated before this patch simply lack these fields — the rendering code must handle their absence gracefully with null/empty fallbacks. No existing UI is modified.

---

### New snapshot fields

Extend the Debrief snapshot object with the following four fields:

```json
{
  "taskLog": [...],
  "arcSnapshots": [...],
  "zoneLog": [...],
  "sessionActivity": {...}
}
```

---

### `taskLog` — Full task event log for the week

An array of every `visions-log` entry for the current ISO week, enriched with human-readable names at snapshot time.

**Structure per entry:**
```json
{
  "dailyId": "uid",
  "dailyName": "Task name",
  "arcId": "uid or null",
  "arcName": "Arc name or null",
  "action": "done | skipped | removed | drift-flagged",
  "zone": "locked-in | processing | cruise | maintenance | null",
  "passiveCompletion": false,
  "carryCount": 0,
  "timestamp": "ISO datetime",
  "dayLabel": "MON | TUE | WED | THU | FRI | SAT | SUN"
}
```

To populate `dailyName`: look up the Daily by id in the live `dailies` array. If not found (task was consumed), fall back to the `dailyName` field already stored on the log entry. If an Arc no longer exists, use `"[deleted arc]"` as `arcName`.

`dayLabel` is derived from the local date of `timestamp`:
```js
const days = ['SUN','MON','TUE','WED','THU','FRI','SAT'];
const dayLabel = days[new Date(entry.timestamp).getDay()];
```

---

### `arcSnapshots` — Arc state frozen at generation time

An array capturing the state of every Arc at the exact moment the Debrief is generated. Enables future export to reconstruct Arc history even after Arcs are renamed or deleted.

**Structure per entry:**
```json
{
  "arcId": "uid",
  "arcName": "name at snapshot time",
  "arcCategory": "SKILL | ACTIVE | ADMIN",
  "arcStatus": "active | paused | complete",
  "progressPct": 34,
  "completionsThisWeek": 4,
  "completionsThisMonth": 11,
  "targetCompletions": 20,
  "linkedDailyCount": 3,
  "priority": 2
}
```

Capture all Arcs from the live `visions-arcs` array at generation time, regardless of status.

`completionsThisWeek`: count of `visions-log` entries for the current week where `arcId === arc.id` and `action === 'done'`.

`completionsThisMonth`: count of `visions-log` entries in the past 30 days where `arcId === arc.id` and `action === 'done'`.

---

### `zoneLog` — Per-zone task distribution detail

An array of every log entry for the current week where `zone !== null`, enriched with task names and hour-of-day context.

**Structure per entry:**
```json
{
  "dailyId": "uid",
  "dailyName": "Task name",
  "zone": "locked-in | processing | cruise | maintenance",
  "action": "done | skipped | removed",
  "timestamp": "ISO datetime",
  "hourOfDay": 14
}
```

`hourOfDay` is the local hour (0–23):
```js
const hourOfDay = new Date(entry.timestamp).getHours();
```

Derive from `visions-log` entries for the current week where `entry.zone !== null`.

---

### `sessionActivity` — Daily activity summary

An object keyed by day label, summarising activity for each day of the week.

**Structure:**
```json
{
  "MON": {
    "tasksDone": 3,
    "tasksSkipped": 1,
    "tasksRemoved": 0,
    "driftCount": 0,
    "firstActivityAt": "ISO datetime or null",
    "lastActivityAt": "ISO datetime or null"
  },
  "TUE": { ... },
  "WED": { ... },
  "THU": { ... },
  "FRI": { ... },
  "SAT": { ... },
  "SUN": { ... }
}
```

Derive from `visions-log` entries for the current week, grouped by local day label. Days with zero activity appear in the object with all numeric values at 0 and both timestamps as `null`.

`firstActivityAt` / `lastActivityAt`: the earliest and latest `timestamp` values among all log entries for that day.

---

## WHAT NOT TO CHANGE

- Do not modify the TODAY tab layout, drag-and-drop behavior, or filter logic beyond the sort addition in Update 2
- Do not change the Morning Sync format or top bar
- Do not change the Settings panel
- Do not change the Drift timer logic
- Do not change any existing `save()` / `renderDailies()` sequences
- Do not change typography, spacing, or color of any existing UI elements not mentioned in this patch
- Do not add any external libraries or CDN links
- Do not simplify or omit any existing features from v1.4

---

**Output:** The complete updated single HTML file with all four updates applied on top of v1.4. All CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts in `<head>`. Immediately usable by double-clicking on a Mac.**
