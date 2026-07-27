# VISIONS OS — Update Patch v1.3

You have already built VISIONS OS as a single self-contained HTML file. Do NOT rebuild it from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality that is not mentioned in this patch.

There are seven updates in this patch.

---

## UPDATE 1 — DEBRIEF: Fix Date Accuracy

### Problem
The Debrief currently displays data from the wrong day — tasks logged on Day N appear in the debrief as belonging to Day N+1. This is an off-by-one error in how log entry timestamps are compared against the week boundary.

### Fix

**In the Debrief generation function** (wherever log entries are filtered by week), change all date comparisons to use the **local date** of the timestamp, not the UTC date.

Replace any usage of `new Date(entry.timestamp).toISOString().slice(0, 10)` with a local-date extraction helper:

```js
function localDateStr(isoTimestamp) {
  const d = new Date(isoTimestamp);
  const yyyy = d.getFullYear();
  const mm = String(d.getMonth() + 1).padStart(2, '0');
  const dd = String(d.getDate()).padStart(2, '0');
  return `${yyyy}-${mm}-${dd}`;
}
```

Use `localDateStr(entry.timestamp)` everywhere a log entry's date is compared against a target date string or week boundary — including:
- The weekly completion rate calculation
- The per-day bar chart data
- The carry-over pattern detection
- The arc progress 30-day window
- Any `entry.week` assignment when a log entry is created

**When creating new log entries**, set the `week` field using local date arithmetic, not UTC:
```js
function getLocalISOWeek(date) {
  const d = new Date(date);
  d.setHours(0, 0, 0, 0);
  d.setDate(d.getDate() + 3 - (d.getDay() + 6) % 7);
  const week1 = new Date(d.getFullYear(), 0, 4);
  const weekNum = 1 + Math.round(((d - week1) / 86400000 - 3 + (week1.getDay() + 6) % 7) / 7);
  return `${d.getFullYear()}-W${String(weekNum).padStart(2, '0')}`;
}
```

Use `getLocalISOWeek(new Date())` instead of any UTC-based week string when writing log entries.

**Debrief week range display:** The week date range shown at the top of the Debrief (e.g. "MAY 12 – MAY 18") must also use local dates. Derive the Monday and Sunday of the displayed week from local date arithmetic, not `Date.UTC`.

---

## UPDATE 2 — DEBRIEF: Behavioral Notes — Respect Paused Arc Status

### Problem
The behavioral note rule "Arc hasn't moved in two weeks" incorrectly fires for Arcs that are intentionally paused.

### Fix

In the behavioral notes generation logic, find the rule that checks for Arcs with 0% progress over 2 consecutive weeks. Add a guard condition:

```js
// BEFORE (incorrect):
if (arc.progressDelta === 0 && arcWasZeroLastWeek) {
  notes.push(`${arc.name} hasn't moved in two weeks...`);
}

// AFTER (correct):
if (arc.status !== 'paused' && arc.progressDelta === 0 && arcWasZeroLastWeek) {
  notes.push(`${arc.name} hasn't moved in two weeks...`);
}
```

Additionally, add a new behavioral note rule specifically for paused arcs: if any Arc has `status === 'paused'` and has been paused for more than 14 days (compare `arc.pausedAt` timestamp if available, otherwise skip this check), generate the note:
- `"[Arc Name] has been paused for over two weeks. Consider resuming it or marking it complete."`

If `arc.pausedAt` does not exist as a field in the current data model, add it: when an Arc's status is toggled to `'paused'` in the status-click handler, set `arc.pausedAt = new Date().toISOString()` and save. When toggled away from `'paused'`, clear `arc.pausedAt = null`.

---

## UPDATE 3 — SKIP BEHAVIOR: Carry Count Color Escalation + 4th-Day Block

### Overview
Track how many times each Daily has been carried over. Visually escalate the card color on each carry, and block skip/remove on the 4th carry, triggering a mandatory inactivity reason prompt instead.

---

### 3a. Data Model Change

Add a `carryCount` field to the Daily object (integer, default `0`). This field persists in `visions-dailies` localStorage.

On first load of existing data, if a Daily object does not have `carryCount`, default it to `0`.

---

### 3b. Increment Logic

Each time a Daily is skipped (via either the bank action or the quadrant card action), increment `daily.carryCount` by 1 and save to `visions-dailies`.

When a Daily is marked **done** or **removed** (with a reason — see Update 5), reset `daily.carryCount = 0` and save.

---

### 3c. Visual States

In `renderDailies()` and `renderQCard()`, after rendering the card, apply a class based on `daily.carryCount`:

| carryCount | Class to add | Visual effect |
|---|---|---|
| 1 | `carry-1` | Subtle yellow tint on card background |
| 2 | `carry-2` | Orange tint on card background |
| 3+ | `carry-3` | Pulsing red border |

**Add CSS:**
```css
.carry-1 { background: rgba(138, 98, 0, 0.18) !important; }
.carry-2 { background: rgba(180, 80, 0, 0.25) !important; }
.carry-3 {
  background: rgba(168, 43, 30, 0.15) !important;
  animation: carryPulse 1.8s ease-in-out infinite;
}
@keyframes carryPulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(168, 43, 30, 0); border-color: #2a2a2a; }
  50%       { box-shadow: 0 0 0 3px rgba(168, 43, 30, 0.4); border-color: #a82b1e; }
}
```

These classes are applied **only when the carry color setting is enabled** (see Update 4). If the setting is disabled, do not add any carry class.

---

### 3d. 4th-Day Block

When the user attempts to **skip** or **remove** a Daily where `carryCount >= 3`:
- Do NOT execute the skip/remove action
- Instead, trigger the **Inactivity Reason Prompt** (see Update 5)
- Pass the action type (`'skip'` or `'remove'`) to the prompt so it can be logged correctly after the reason is submitted

---

## UPDATE 4 — SETTINGS: Carry Color Toggle

### Add a toggle to the Settings panel for the carry color escalation feature.

**In `#settings-panel-body`**, below the existing QUADRANT THEME section, add a new section:

```
CARRY INDICATORS
```

Section header style: Space Mono, 10px, uppercase, `#666`, with a `#2a2a2a` divider line above and below — matching the existing QUADRANT THEME section header style.

**Section content:**

A single toggle row:
- Label: `"Color-escalate carried tasks"` — Space Mono, 12px, `#f0ece3`
- Sublabel below: `"Yellow → Orange → Red pulse on 1st, 2nd, 3rd carry"` — Space Mono, 10px, `#555`
- A toggle switch on the right (use a CSS checkbox toggle — no JS library)

**Toggle CSS (add to existing `<style>` block):**
```css
.settings-toggle-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 0;
}
.toggle-switch {
  position: relative;
  width: 36px;
  height: 20px;
  flex-shrink: 0;
}
.toggle-switch input { opacity: 0; width: 0; height: 0; }
.toggle-slider {
  position: absolute;
  inset: 0;
  background: #2a2a2a;
  border-radius: 20px;
  cursor: pointer;
  transition: background 0.2s;
}
.toggle-slider::before {
  content: '';
  position: absolute;
  width: 14px; height: 14px;
  left: 3px; top: 3px;
  background: #666;
  border-radius: 50%;
  transition: transform 0.2s, background 0.2s;
}
.toggle-switch input:checked + .toggle-slider { background: #1d7044; }
.toggle-switch input:checked + .toggle-slider::before {
  transform: translateX(16px);
  background: #f0ece3;
}
```

**HTML for the toggle row:**
```html
<div class="settings-toggle-row">
  <div>
    <div style="font-family:'Space Mono',monospace;font-size:12px;color:#f0ece3;">Color-escalate carried tasks</div>
    <div style="font-family:'Space Mono',monospace;font-size:10px;color:#555;margin-top:3px;">Yellow → Orange → Red pulse on 1st, 2nd, 3rd carry</div>
  </div>
  <label class="toggle-switch">
    <input type="checkbox" id="toggle-carry-colors">
    <span class="toggle-slider"></span>
  </label>
</div>
```

**Behavior (JS):**

- On `renderSettings()` (or on settings panel open): read `settings.carryColorsEnabled` from `visions-settings`. Set `#toggle-carry-colors` checked state accordingly. Default: `true` (on).
- On `change` of `#toggle-carry-colors`: update `settings.carryColorsEnabled` and save to `visions-settings`. Immediately re-render the Dailies bank and quadrant so cards update their carry classes.

---

## UPDATE 5 — INACTIVITY REASON PROMPT + DEBRIEF SECTION

### 5a. Inactivity Reason Prompt (Modal)

When triggered (from Update 3d — 4th-day block), show an inline modal overlay with the following content:

**HTML structure** (inject dynamically, do not hardcode in the HTML file — create and append via JS, remove on dismiss):

```html
<div id="inactivity-overlay">
  <div id="inactivity-modal">
    <div id="inactivity-modal-header">WHY IS THIS STALLED?</div>
    <div id="inactivity-modal-task-name">[Task Name]</div>
    <div id="inactivity-modal-subtext">This task has been carried 3 times. Log the reason before skipping or removing it.</div>
    <div class="inactivity-options">
      <button class="inactivity-reason-btn" data-reason="Too vague">Too vague — needs breaking down</button>
      <button class="inactivity-reason-btn" data-reason="Not a priority">Not a priority right now</button>
      <button class="inactivity-reason-btn" data-reason="Blocked">Blocked — waiting on something</button>
      <button class="inactivity-reason-btn" data-reason="Avoidance">Avoidance — I know I need to do it</button>
      <button class="inactivity-reason-btn" data-reason="Wrong day">Wrong day for this task</button>
    </div>
    <div style="margin-top:10px;">
      <input type="text" id="inactivity-custom-input" placeholder="Or type your own reason..." />
    </div>
    <div id="inactivity-modal-actions">
      <button id="inactivity-confirm-btn">LOG & SKIP</button>
      <button id="inactivity-remove-btn">LOG & REMOVE</button>
      <button id="inactivity-cancel-btn">CANCEL</button>
    </div>
  </div>
</div>
```

**CSS:**
```css
#inactivity-overlay {
  display: flex;
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.65);
  z-index: 9000;
  align-items: center;
  justify-content: center;
}
#inactivity-modal {
  background: #1a1a1a;
  border: 1px solid #a82b1e;
  border-radius: 6px;
  padding: 24px;
  width: 400px;
  max-width: 90vw;
  font-family: 'Space Mono', monospace;
}
#inactivity-modal-header {
  font-size: 13px;
  color: #a82b1e;
  text-transform: uppercase;
  letter-spacing: 2px;
  margin-bottom: 8px;
}
#inactivity-modal-task-name {
  font-size: 15px;
  color: #f0ece3;
  margin-bottom: 6px;
}
#inactivity-modal-subtext {
  font-size: 10px;
  color: #555;
  margin-bottom: 16px;
  line-height: 1.6;
}
.inactivity-options {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.inactivity-reason-btn {
  background: #111;
  border: 1px solid #2a2a2a;
  color: #f0ece3;
  font-family: 'Space Mono', monospace;
  font-size: 11px;
  padding: 8px 12px;
  text-align: left;
  cursor: pointer;
  border-radius: 3px;
  transition: border-color 0.15s, background 0.15s;
}
.inactivity-reason-btn:hover { border-color: #555; background: #1e1e1e; }
.inactivity-reason-btn.selected { border-color: #a82b1e; background: rgba(168,43,30,0.12); color: #f0ece3; }
#inactivity-custom-input {
  width: 100%;
  background: #111;
  border: 1px solid #2a2a2a;
  color: #f0ece3;
  font-family: 'Space Mono', monospace;
  font-size: 11px;
  padding: 8px 10px;
  border-radius: 3px;
  margin-top: 6px;
}
#inactivity-custom-input:focus { outline: none; border-color: #555; }
#inactivity-modal-actions {
  display: flex;
  gap: 8px;
  margin-top: 16px;
}
#inactivity-confirm-btn {
  background: #8a6200;
  color: #f0ece3;
  border: none;
  font-family: 'Space Mono', monospace;
  font-size: 10px;
  padding: 8px 14px;
  cursor: pointer;
  border-radius: 3px;
  text-transform: uppercase;
}
#inactivity-remove-btn {
  background: #a82b1e;
  color: #f0ece3;
  border: none;
  font-family: 'Space Mono', monospace;
  font-size: 10px;
  padding: 8px 14px;
  cursor: pointer;
  border-radius: 3px;
  text-transform: uppercase;
}
#inactivity-cancel-btn {
  background: none;
  border: 1px solid #2a2a2a;
  color: #666;
  font-family: 'Space Mono', monospace;
  font-size: 10px;
  padding: 8px 14px;
  cursor: pointer;
  border-radius: 3px;
  text-transform: uppercase;
}
```

**Behavior (JS):**

- A reason must be selected (either a preset button or a non-empty custom input) before `LOG & SKIP` or `LOG & REMOVE` are enabled. While no reason is selected, both action buttons are visually dimmed (`opacity: 0.4`, `cursor: default`).
- Clicking a preset button marks it as `.selected` and deselects others.
- Typing in the custom input deselects any preset button.
- On `LOG & SKIP`:
  - Write an inactivity log entry (see 5b below)
  - Execute the skip action on the task (as if skip was pressed normally), which also increments `carryCount`
  - Dismiss the modal
- On `LOG & REMOVE`:
  - Write an inactivity log entry (see 5b below)
  - Remove the task from the bank entirely
  - Reset `daily.carryCount = 0`
  - Dismiss the modal
- On `CANCEL`: dismiss the modal, no changes
- `Escape` key: dismiss the modal

---

### 5b. Inactivity Log Data

Add a new localStorage key: **`visions-inactivity`** — an array of inactivity log entries.

**Entry structure:**
```json
{
  "id": "uid",
  "dailyId": "uid",
  "dailyName": "Task name",
  "action": "skipped | removed",
  "reason": "Selected or typed reason string",
  "carryCount": 3,
  "timestamp": "ISO datetime",
  "week": "2026-W21"
}
```

Append a new entry on every LOG & SKIP or LOG & REMOVE action. Do not overwrite previous entries for the same task — each event gets its own entry.

---

### 5c. Debrief Section — STALLED TASKS

Add a new section to the Debrief tab, between **CARRY-OVER PATTERN** and **CATEGORY BALANCE**. Title it `STALLED TASKS`.

**Section content:**

- If there are no inactivity entries for the current week: show faint italic text `"No stalled tasks logged this week."` in `#666`.
- If there are entries: render a list, one row per entry, showing:

```
[Task Name]  →  [Action: SKIPPED / REMOVED]  ·  "[Reason]"  ·  [Carry count]× carried
```

Format: Space Mono, 12px. Task name in `#f0ece3`. Action badge styled like existing category badges (`SKIPPED` in amber, `REMOVED` in red). Reason in italic `#888`. Carry count in `#555`.

**In Debrief history snapshots** (stored in `visions-debrief`), add a `stalledTasks` array field to the snapshot object containing the inactivity entries for that week.

---

## UPDATE 6 — THE RUNWAY: Arc Progress Bar Gradient

### Change the Arc progress bar fill from a solid color to a progress-responsive gradient.

**Current behavior:** The filled portion of each Arc's progress bar is a solid color (green for SKILL, blue for ACTIVE, grey for ADMIN).

**New behavior:** The filled bar transitions from a neutral grey at 0% progress to full green at 100%, regardless of Arc category. The gradient is applied to the fill element itself, not the container.

**Implementation:**

In `renderArcs()`, instead of setting the bar fill's background to a solid category color, use a dynamic inline style that applies a `linear-gradient`:

```js
function getArcBarStyle(progress) {
  // progress: 0–100
  const pct = Math.max(0, Math.min(100, progress));
  // Interpolate from grey (#3a3a3a) at 0% to green (#1d7044) at 100%
  // Use a gradient that covers the full bar width, clipped by the fill width
  return `linear-gradient(to right, #3a3a3a 0%, #2a5c3a ${Math.max(0, pct - 30)}%, #1d7044 100%)`;
}
```

Apply this to the bar fill element:
```js
barFillEl.style.background = getArcBarStyle(arc.progress);
barFillEl.style.width = arc.progress + '%';
```

This means:
- At **0%**: bar is invisible (zero width) — no color shown
- At **10–30%**: bar shows dark grey-green
- At **50%**: bar shows mid-tone green
- At **100%**: bar shows full `#1d7044` green

The unfilled portion of the bar (the container) stays `#1e1e1e` (its existing background) — no change needed there.

**Remove** the existing category-based bar color logic (the `if SKILL → green, if ACTIVE → blue, if ADMIN → grey` branches) for the bar fill element only. Category badges on the Arc card are unchanged.

**Apply this gradient only when the arc gradient setting is enabled** (see Update 7). When disabled, fall back to the existing solid category-color logic.

---

## UPDATE 7 — SETTINGS: Arc Bar Gradient Toggle

### Add a second toggle to the CARRY INDICATORS section in the Settings panel.

Below the existing carry color toggle row (from Update 4), add a second toggle row:

```html
<div class="settings-toggle-row">
  <div>
    <div style="font-family:'Space Mono',monospace;font-size:12px;color:#f0ece3;">Arc progress gradient</div>
    <div style="font-family:'Space Mono',monospace;font-size:10px;color:#555;margin-top:3px;">Grey → Green gradient fill on Arc progress bars</div>
  </div>
  <label class="toggle-switch">
    <input type="checkbox" id="toggle-arc-gradient">
    <span class="toggle-slider"></span>
  </label>
</div>
```

**Behavior (JS):**

- On settings panel open: read `settings.arcGradientEnabled` from `visions-settings`. Set `#toggle-arc-gradient` checked state accordingly. Default: `true` (on).
- On `change` of `#toggle-arc-gradient`: update `settings.arcGradientEnabled` and save to `visions-settings`. If THE RUNWAY tab is currently active, immediately re-render arcs so bars update. If not active, they will re-render next time the tab is opened.

**New `visions-settings` shape after all patches:**
```json
{
  "quadrantColors": { "lockedIn": "#...", "processing": "#...", "cruise": "#...", "maintenance": "#..." },
  "dailiesCollapsed": false,
  "carryColorsEnabled": true,
  "arcGradientEnabled": true
}
```

---

## VISUAL / STYLE NOTES FOR NEW ELEMENTS

All new UI elements must match the existing app's aesthetic:
- Font: Space Mono throughout
- Background: `#111` or `#1a1a1a` for inputs and surfaces
- Borders: `1px solid #2a2a2a`
- Text: `#f0ece3` primary, `#666` secondary
- Buttons follow existing patterns — no new design patterns introduced
- Animations: 200–250ms ease for all transitions
- No new external libraries, no new CDN links

---

## WHAT NOT TO CHANGE

- Do not modify the tab bar layout or any existing tab navigation
- Do not alter the Drift timer logic
- Do not change the drag-and-drop behavior for Dailies or Arc reordering
- Do not change the Morning Sync format
- Do not modify the existing QUADRANT THEME section in Settings
- Do not change typography, spacing, or color of any existing elements not mentioned in this patch
- Do not simplify or omit any existing features
- Do not change any existing localStorage keys beyond: writing `pausedAt` on Arc objects (within `visions-arcs`), adding `carryCount` to Daily objects (within `visions-dailies`), adding `stalledTasks` to Debrief snapshots (within `visions-debrief`), and adding the new `visions-inactivity` key

---

**Output:** The complete updated single HTML file with all seven updates applied. Same file format as original — all CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts in `<head>`. Immediately usable by double-clicking on a Mac.**
