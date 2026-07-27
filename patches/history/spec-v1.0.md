# VISIONS OS — Full Build Specification

Build me a single-file HTML app called VISIONS OS — a personal productivity system for a solo visual content creator. Below is the complete specification. Follow it precisely.

---

## TECH STACK

Single self-contained HTML file. Vanilla JS, CSS, no external frameworks. Google Fonts allowed via CDN link. All data stored in `localStorage`. Must work offline by double-clicking the file on a Mac. Do not use React, Vue, or any bundler.

---

## VISUAL DESIGN

**Aesthetic:** Dark, editorial, cinematic. Inspired by a film production OS — not a generic productivity app.

**Color palette:**
- Background: `#111111`
- Surface cards: `#1a1a1a`
- Borders: `#2a2a2a`
- Text primary: `#f0ece3`
- Text secondary: `#666`
- Accent green (done/SKILL): `#1d7044`
- Accent blue (ACTIVE): `#1a5c8a`
- Accent red (Drift/timer): `#a82b1e`
- Accent amber (skipped/warning): `#8a6200`
- Accent white (highlight): `#ffffff`

**Typography:**
- Display / headings: `Permanent Marker` (Google Fonts)
- Body / UI: `Space Mono` (Google Fonts)

**Quadrant colors:**
- LOCKED IN: `#1d3d2a` (dark green)
- PROCESSING: `#1a2d45` (dark blue)
- CRUISE: `#2a2a1a` (dark amber/olive)
- MAINTENANCE: `#1e1e1e` (near-black)
- DRIFT (timer zone, subtle overlay): `#3a1a1a` (dark red tint)

**Tab bar:** Minimal, top of screen. Active tab underlined with a 2px white line. Inactive tabs in `#444`.

**Cards:** Rounded corners (`border-radius: 5px`), subtle border `1px solid #2a2a2a`, slight shadow on hover.

---

## APP STRUCTURE

Three tabs at the top:

```
[ TODAY ]     [ THE RUNWAY ]     [ DEBRIEF ]
```

Below the tab bar, a persistent top bar always visible across all tabs:

- Left: App name "VISIONS OS" in Permanent Marker font
- Center: Current date (e.g. MON, MAY 18)
- Right: "☀ MORNING SYNC" button — when clicked, generates a plain-text summary of today's active Dailies + Runway Arc statuses, copies it to clipboard, and shows a brief "Copied!" confirmation. The summary format should be: `VISIONS OS — [DATE] | Active Dailies: [task1, task2...] | Arcs in Progress: [Arc name X% · Arc name X%...]`

---

## TAB 1 — TODAY

**Layout:** Two panels side by side. Left panel = DAILIES BANK (40% width). Right panel = THE QUADRANT (60% width). On narrow screens, stack vertically.

---

### LEFT PANEL: DAILIES BANK

**Header:** "DAILIES" in Permanent Marker, small. Below it a search/filter bar (text input) and three filter buttons: `ALL` `SKILL` `ACTIVE` `ADMIN`.

**Adding a new Daily:**
- A collapsed "+ NEW DAILY" button at the top of the bank
- When clicked, expands a form inline with: Task name (text input), Category selector (SKILL / ACTIVE / ADMIN), Arc link selector (dropdown of all Arcs from The Runway, plus "None"), optional Description textarea, Confirm button
- New tasks go to top of the bank

**Each Daily card in the bank shows:**
- Left: A drag handle icon (`⠿`)
- Task name in Space Mono, 13px
- Category badge: colored pill — green for SKILL, blue for ACTIVE, grey for ADMIN
- Arc tag below the name if linked (e.g. "↗ Sound Design")
- Right side: Three icon buttons — `✓` (done), `→` (skip), `✕` (remove)
- Below task name: A collapsed arrow `›` — clicking it expands an inline editable description field (textarea, saves on blur)

**Task states:**
- **Active:** Normal appearance
- **Done:** Strikethrough, 40% opacity, green checkmark
- **Skipped:** Italic, 40% opacity, amber arrow icon — task reappears in bank next day with a small "↩ carried" label
- **Removed:** Disappears from bank entirely, logged in Debrief data

**Drag behavior:** Daily cards can be dragged from the bank into any quadrant on the right panel. When dragged into a quadrant, a ghost/placeholder appears in the bank with a dashed border to show the task is "placed." The task remains in the bank list but dimmed — it still shows its action buttons. Dragging it back to the bank or pressing a reset icon on the quadrant card returns it to normal.

**Pre-loaded Dailies (load these by default on first launch, do NOT reload on subsequent opens):**

ACTIVE category:
1. Brief / Synopsis
2. Script
3. Storyboard
4. Shot List
5. Assemble Load-out
6. Charge Equipment
7. Execute Content Shoot
8. Record B-Rolls
9. Cut Footage
10. Assemble Rough Cut
11. Quick Edit
12. Color Correction
13. Effects / GFX (Fusion)
14. Sound Design / SFX
15. Assemble Music
16. Export / Delivery

SKILL category:
17. Collect Visual References
18. Break Down a Reference Video
19. Learn Arabic Word
20. Watch Fusion Tutorial
21. Explore New AI Tool
22. Read / Quran

ADMIN category:
23. Organize Project Files
24. Consolidate Old Projects
25. Reply Messages / Catch Up
26. Check Emails

---

### RIGHT PANEL: THE QUADRANT

**Header:** "THE QUADRANT" label top right. Small legend below it showing the 4 zone names and their energy/output meaning.

**Layout:** 2x2 grid of zones. Each zone fills its cell fully. A subtle cross-hair divides the grid (2px dark lines). Axis labels outside the grid:
- Top: "HIGH ENERGY" (small, muted)
- Bottom: "LOW ENERGY"
- Left: "CREATIVE OUTPUT"
- Right: "ADMINISTRATIVE OUTPUT"

**Zone definitions:**

| Position | Name | Description | Color |
|---|---|---|---|
| Top-Left | LOCKED IN | High Energy + Creative | `#1d3d2a` |
| Top-Right | PROCESSING | High Energy + Administrative | `#1a2d45` |
| Bottom-Left | CRUISE | Low Energy + Creative | `#2a2a1a` |
| Bottom-Right | MAINTENANCE | Low Energy + Administrative | `#1e1e1e` |

**DRIFT indicator:** Not a 5th quadrant. Instead, if a task is flagged as "distraction" (user can right-click a quadrant card and select "flag as drift"), the card gets a red border and a countdown timer starts. Default timer: 20 minutes. Timer shows as a small pill on the card counting down. When it hits 0, the card pulses red and a toast notification appears: "Drift time's up — back to work?" with a dismiss button. Timer is paused if the app tab is not active (use `visibilitychange` event).

**Quadrant card behavior (once a Daily is dropped in):**
- Shows task name, category badge, Arc link tag
- Three buttons: `✓` Done, `→` Skip, `✕` Remove from quadrant (returns to bank as active)
- Right-click context menu: "Flag as Drift" / "Move to..." (shows other 3 zones) / "Add note"
- Cards are draggable between zones within the quadrant

**Quadrant state:** Resets each day at midnight (or on first open after midnight). Completed/skipped tasks are logged before reset.

---

## TAB 2 — THE RUNWAY

**Purpose:** Long-term Arc view. All Arcs listed by priority order (top = highest priority). User can manually reorder by drag.

**Header:** "THE RUNWAY" in Permanent Marker. Right side: "+ NEW ARC" button.

**Adding a new Arc:**
- Modal overlay: Arc name, Category (SKILL / ACTIVE / ADMIN), Description, Target milestone (optional text), Linked Dailies (multi-select from existing Dailies list)
- Confirm adds it to bottom of list

**Each Arc card shows:**
- Priority number (auto-numbered 01, 02, 03...)
- Arc name in Permanent Marker, 15px
- Category badge
- **Progress bar:** Full width, 0–100%, calculated automatically (see logic below). Percentage shown numerically to the right of the bar. Bar color: green for SKILL, blue for ACTIVE, grey for ADMIN. Bar fills with a smooth `transition: width 0.4s ease`.
- Status tag: `ACTIVE` / `PAUSED` / `COMPLETE` — clickable to toggle
- Small text below: "X tasks linked · Last activity: [date]"
- Expand button `›` — opens Arc detail view inline below the card:
  - Full description (editable)
  - Milestone notes (editable textarea)
  - List of all linked Dailies (checkboxes showing which are done)
  - History log: last 10 completions with timestamps
  - "Add linked Daily" button (opens Daily selector)

**Progress calculation:**
- Each Arc has linked Dailies
- Progress = (number of times linked Dailies have been marked Done in the past 30 days) / (target completions) x 100
- Target completions default to 20 (configurable per Arc in the detail view)
- Example: If "Sound Design" Arc has 3 linked Dailies and they've been completed 6 times total in 30 days, and target is 20, progress = 30%
- When progress hits 100%, Arc status auto-changes to `COMPLETE` and a subtle celebration animation plays (confetti or pulse glow)

**Pre-loaded Arcs (first launch only), in this priority order:**

```
01  First Pitch Deck            ACTIVE
02  Visual Portfolio             ACTIVE
03  Niche Definition             ACTIVE
04  Media Service Package        ACTIVE
05  Master Deck                  ACTIVE
06  Pre-Production Discipline    SKILL
07  Sound Design                 SKILL
08  Color & Fusion               SKILL
09  MACRO Library                SKILL
10  SFX Library                  SKILL
11  AI Update                    SKILL
12  Tech Update                  SKILL
13  Project Files Consolidation  ADMIN
14  CV / Resume                  ACTIVE
15  Arabic Vocabulary            SKILL
```

**Pre-link these Dailies to Arcs on first launch:**
- "First Pitch Deck" → Brief/Synopsis, Script, Storyboard
- "Pre-Production Discipline" → Brief/Synopsis, Script, Storyboard, Shot List
- "Sound Design" → Sound Design/SFX, Assemble Music, Explore New AI Tool
- "Color & Fusion" → Color Correction, Effects/GFX (Fusion), Watch Fusion Tutorial
- "SFX Library" → Sound Design/SFX, Assemble Music
- "MACRO Library" → Effects/GFX (Fusion), Watch Fusion Tutorial
- "AI Update" → Explore New AI Tool
- "Project Files Consolidation" → Organize Project Files, Consolidate Old Projects
- "Arabic Vocabulary" → Learn Arabic Word
- "Visual Portfolio" → Execute Content Shoot, Quick Edit, Export/Delivery
- "Tech Update" → Explore New AI Tool, Break Down a Reference Video

---

## TAB 3 — DEBRIEF

**Purpose:** Weekly performance review. Auto-generates on first open each Sunday, or triggered manually via a "Run Debrief" button.

**Header:** "DEBRIEF" in Permanent Marker. Below: the week date range (e.g. "MAY 12 – MAY 18").

### Sections:

**1. COMPLETION RATE**
- Large number: e.g. "68%" — percentage of Dailies marked Done vs. total placed on quadrant that week
- Below: three smaller stats — Done: X · Skipped: X · Removed: X
- Bar chart (simple CSS bars, no library): one bar per day of the week, showing daily completion %

**2. QUADRANT BEHAVIOR**
- Four small stat blocks, one per zone — showing how many tasks landed in each zone that week
- Identifies dominant zone: e.g. "You spent most time in MAINTENANCE this week"

**3. ARC PROGRESS**
- List of all active Arcs with their % change from last week
- Format: "↑ +8% Sound Design" / "— 0% CV/Resume" / "↑ +3% Visual Portfolio"
- Arcs that made zero progress are highlighted in amber

**4. CARRY-OVER PATTERN**
- Tasks skipped 3+ times in a week are flagged: e.g. "Script — skipped 4 times this week"
- Label: "AVOIDANCE SIGNALS"

**5. CATEGORY BALANCE**
- Donut chart (CSS only, no JS library): shows split of completed tasks between SKILL / ACTIVE / ADMIN
- Target range shown as a reference marker: recommended balance is ~40% ACTIVE, 40% SKILL, 20% ADMIN

**6. DRIFT INCIDENTS**
- Count of how many drift flags were triggered
- Total drift time in minutes

**7. BEHAVIORAL NOTES**
Three plain-text auto-generated observations based on the data, written in direct editorial tone. Use rule-based logic:
- If completion < 50%: "Tough week. Less than half your quadrant tasks got done. Check if you're overloading the quadrant or underestimating task size."
- If MAINTENANCE > 40% of tasks: "You leaned heavy on admin this week. Make sure maintenance isn't a proxy for avoiding harder creative work."
- If any Arc has 0% progress for 2 weeks: "[Arc Name] hasn't moved in two weeks. It either needs to be broken into smaller Dailies or deprioritized."
- If skipped task appears 3+ times: "[Task] keeps getting skipped. Consider removing it from the bank entirely or breaking it into a smaller starting action."
- If completion > 80%: "Strong week. This is what forward momentum looks like."

**8. FOCUS RECOMMENDATION**
One sentence at the bottom, bold: the highest-priority Arc with zero or lowest progress this week becomes next week's recommended focus. Format: "Next week — focus on [Arc Name]."

**History:** Store the last 8 weeks of Debrief data in localStorage. A small "< Previous" / "Next >" nav at the top of the Debrief tab lets you scroll through past weeks.

---

## DATA MODEL (localStorage keys)

```
visions-dailies       → Array of Daily objects
visions-arcs          → Array of Arc objects
visions-log           → Array of completion log entries (date, taskId, action)
visions-debrief       → Array of weekly Debrief snapshots
visions-quadrant      → Today's quadrant state (resets daily)
visions-settings      → User preferences
visions-initialized   → Boolean (true after first launch data load)
```

**Daily object:**
```json
{
  "id": "uid",
  "text": "Task name",
  "category": "SKILL | ACTIVE | ADMIN",
  "arcId": "arc-uid or null",
  "description": "optional notes",
  "status": "active | done | skipped | removed",
  "isDrift": false,
  "carriedOver": false,
  "createdAt": "ISO date",
  "lastModified": "ISO date"
}
```

**Arc object:**
```json
{
  "id": "uid",
  "name": "Arc name",
  "category": "SKILL | ACTIVE | ADMIN",
  "description": "",
  "milestoneNotes": "",
  "status": "active | paused | complete",
  "targetCompletions": 20,
  "linkedDailyIds": [],
  "priority": 1,
  "createdAt": "ISO date"
}
```

**Log entry:**
```json
{
  "id": "uid",
  "dailyId": "uid",
  "arcId": "uid or null",
  "action": "done | skipped | removed | drift",
  "zone": "locked-in | processing | cruise | maintenance | null",
  "timestamp": "ISO datetime",
  "week": "2026-W20"
}
```

---

## BEHAVIORS & EDGE CASES

**First launch:** Check `visions-initialized`. If false, load all pre-loaded Dailies and Arcs, set the flag to true. Never overwrite user data after initialization.

**Daily reset:** On app open, check if `visions-quadrant.date` !== today's date. If so, process yesterday's quadrant: log all placed tasks (done/skipped/active), move skipped tasks back to bank with `carriedOver: true`, clear the quadrant, update `visions-quadrant.date` to today.

**Drift timer:** Store drift start time in quadrant state. Calculate elapsed time on render rather than using setInterval for time-keeping, to survive page refreshes. Use setInterval only for UI countdown display.

**Carry-over label:** Show a small amber pill "↩ carried" on any task where `carriedOver: true`. Remove the flag once the task is marked done or removed.

**Arc progress recalculation:** Recalculate on every app open by scanning the log for the past 30 days.

**Debrief generation:** On Sunday open (or manual trigger), scan the log for the current ISO week (`week` field), compute all metrics, store as a Debrief snapshot, render it. If a Debrief for the current week already exists, display it without regenerating.

**Morning Sync copy format:**
```
VISIONS OS — [DAY, MON DD]
DAILIES (Active): [task1, task2, task3...]
CARRIED OVER: [task1, task2...]
THE RUNWAY: [Arc 1 – 34%] · [Arc 2 – 12%] · [Arc 3 – 67%]
```

---

## INTERACTIONS & POLISH

- Smooth CSS transitions on all state changes (200–300ms)
- Drag-and-drop with visual feedback: drag ghost, drop zone highlight, shake animation on invalid drop
- Toast notifications (bottom-right, auto-dismiss 3s) for: task done, arc progress update, drift timer end, debrief ready, morning sync copied
- Hover states on all interactive elements
- All text inputs save on blur (no explicit save button required for edits)
- Keyboard shortcut: `N` opens new Daily form when TODAY tab is active; `Esc` closes any open modal or form
- On hover of an Arc's progress bar: tooltip showing "X of Y target completions this month"
- Empty states: if quadrant zone is empty, show faint italic text e.g. "Drop a Daily here"; if bank is empty, show "No tasks yet — add a Daily above"

---

## WHAT TO AVOID

- No external JS libraries (no jQuery, Lodash, Chart.js, etc.)
- No CSS frameworks (no Tailwind, Bootstrap)
- No API calls of any kind
- No `<form>` tags (use divs with click handlers)
- No alert() or confirm() — use inline UI instead
- Do not add features not described in this spec
- Do not simplify or omit sections for brevity — build the full spec

---

**Build the complete app as a single `.html` file. All CSS in a `<style>` block. All JS in a `<script>` block at the bottom of the body. Google Fonts loaded via a `<link>` tag in the `<head>`. The file should be immediately usable by double-clicking it on a Mac.**
