# VISIONS OS — Update Patch v1.1

You have already built VISIONS OS as a single self-contained HTML file based on the original spec. Do NOT rebuild it from scratch. Read the existing code carefully, then apply ONLY the changes described below. Do not alter any existing functionality that is not mentioned in this patch.

There are three updates and one new tab to add:

---

## UPDATE 1 — THE RUNWAY: Delete Arc + Unlink Daily

### 1a. Delete Arc

Add a delete button to each Arc card in THE RUNWAY tab.

**Placement:** In the Arc card header row, add a `✕ DELETE ARC` button on the far right, next to the existing status tag. Style it as a small muted red text button (no background, no border). Color: `#a82b1e`. Font size: 10px. Space Mono. Uppercase.

**Behavior when clicked:**
- Do NOT delete immediately. Show an inline confirmation prompt directly below the Arc card header (not a browser alert or modal). The prompt reads: `"Delete [Arc Name]? This cannot be undone."` with two buttons: `CONFIRM DELETE` (red) and `CANCEL` (muted).
- If CANCEL: dismiss the prompt, no changes.
- If CONFIRM DELETE:
  - Remove the Arc object from `visions-arcs` in localStorage
  - Find all Daily objects in `visions-dailies` where `arcId` matches the deleted Arc's id — set their `arcId` to `null`
  - Remove any log entries in `visions-log` where `arcId` matches the deleted Arc's id
  - Re-render THE RUNWAY and TODAY panels
  - Show a toast notification: `"Arc deleted. X linked Dailies unlinked."` (amber color, 3s)
  - Re-number the remaining Arc priority numbers (01, 02, 03...) sequentially

### 1b. Unlink Daily from Arc

Inside the expanded Arc detail view (the `‹ DETAILS` section), each linked Daily currently shows as a list item. Update this list so each Daily has an unlink button.

**Placement:** To the right of each linked Daily name in the LINKED DAILIES list, add a small `✕` icon button. Color: `#666`, changes to `#a82b1e` on hover.

**Behavior when clicked:**
- Remove the Daily's id from the Arc's `linkedDailyIds` array in `visions-arcs`
- Set the Daily object's `arcId` to `null` in `visions-dailies`
- Re-render the Arc detail view and the TODAY Dailies Bank (so the Arc tag disappears from the Daily card)
- Show a toast: `"Daily unlinked from Arc."` (muted, 3s)
- Do NOT delete the Daily — it remains in the Dailies Bank, just no longer linked to this Arc

---

## UPDATE 2 — TODAY: Arc Reassignment in Daily Edit

Currently, expanding a Daily card in the Dailies Bank shows only an editable description textarea. Update this expanded view to also include an Arc selector.

**Add below the description textarea:**
- A labeled dropdown: label text `"LINKED ARC"` in Space Mono, 9px, uppercase, color `#666`
- A `<select>` element styled to match the existing app inputs (dark background, Space Mono font, `#f0ece3` text, `#2a2a2a` border)
- The dropdown options: `None` (value: null) followed by all existing Arc names pulled from `visions-arcs`, sorted by priority order
- The dropdown should show the currently linked Arc as the selected option (or "None" if `arcId` is null)

**Behavior on change:**
- When the user selects a different Arc (or None):
  - Update the Daily object's `arcId` in `visions-dailies`
  - If the Daily was previously linked to a different Arc, remove its id from that Arc's `linkedDailyIds` array in `visions-arcs`
  - If the new selection is an Arc (not None), add the Daily's id to the new Arc's `linkedDailyIds` array in `visions-arcs`
  - Save changes to localStorage immediately on change (no save button needed)
  - Re-render the Arc tag below the Daily's name in the bank to reflect the new Arc (or remove it if None)
  - Show a toast: `"Arc updated."` (green, 2s)

---

## UPDATE 3 — NEW SETTINGS TAB

Add a fourth tab to the tab bar:

```
[ TODAY ]     [ THE RUNWAY ]     [ DEBRIEF ]     [ SETTINGS ]
```

The SETTINGS tab contains one section for now: **QUADRANT THEME**.

---

### SETTINGS TAB — Layout

**Header:** "SETTINGS" in Permanent Marker font, same style as other tab headers.

**Section header:** "QUADRANT THEME" — Space Mono, 10px, uppercase, `#666`, with a thin `#2a2a2a` divider line below it.

**Intro line:** Small muted text below the section header: `"Customize the background color of each quadrant zone."` Space Mono, 11px, color `#555`.

---

### QUADRANT THEME — Color Editor

Display four color editor rows, one per quadrant zone, in this order:
1. LOCKED IN
2. PROCESSING
3. CRUISE
4. MAINTENANCE

**Each row contains:**
- Zone name: Space Mono, 12px, `#f0ece3`, uppercase, left-aligned, width ~160px
- A native color picker input (`<input type="color">`) — styled as a small square swatch (32px × 32px, no border, cursor pointer, border-radius 4px). The input's value is initialized from the current saved color for that zone.
- A hex value text input next to the color picker (width ~90px) showing the current hex code. Editable — typing a valid hex code updates the color picker and the preview in real time.
- A small `RESET` text button (Space Mono, 9px, `#444`, no background) that resets that zone to its default color.

**Default colors (match original spec):**
- LOCKED IN: `#1d3d2a`
- PROCESSING: `#1a2d45`
- CRUISE: `#2a2a1a`
- MAINTENANCE: `#1e1e1e`

**Below the four rows:**
- A `APPLY THEME` button (primary style — white background, dark text, Space Mono, uppercase). Width: 140px.
- A `RESET ALL TO DEFAULT` text button (muted, `#444`, no background), right-aligned or below APPLY.

**Behavior:**

- Color picker and hex input stay in sync in real time as the user edits either one.
- Validation: if the hex input contains an invalid hex code (not matching `#` followed by 3 or 6 hex characters), show a small inline error in red: `"Invalid hex"` and do not apply.
- `APPLY THEME` button:
  - Saves the four color values to localStorage under key `visions-settings` as: `{ quadrantColors: { lockedIn: "#...", processing: "#...", cruise: "#...", maintenance: "#..." } }`
  - Immediately applies the new colors to the quadrant zones in the TODAY tab by updating the background-color style of each zone element
  - Shows a toast: `"Theme applied."` (green, 2s)
- `RESET ALL TO DEFAULT`:
  - Resets all four color inputs to their default values
  - Does NOT apply until the user clicks `APPLY THEME`
- Individual `RESET` button per row:
  - Resets only that row's color to its default
  - Does NOT apply until the user clicks `APPLY THEME`

**On app load:**
- Read `visions-settings.quadrantColors` from localStorage
- If values exist, apply them to the quadrant zones immediately before rendering TODAY
- If not found, use the default colors

---

## VISUAL / STYLE NOTES FOR NEW ELEMENTS

All new UI elements must match the existing app's aesthetic:
- Font: Space Mono throughout
- Background inputs: `#1a1a1a` or `#111`
- Borders: `1px solid #2a2a2a`
- Text: `#f0ece3` primary, `#666` secondary
- Buttons follow existing app button styles (no new design patterns)
- Toast notifications follow the existing toast system (bottom-right, auto-dismiss)
- No new external libraries, no new CDN links

---

## WHAT NOT TO CHANGE

- Do not modify any existing tab (TODAY, THE RUNWAY, DEBRIEF) beyond what is explicitly described in this patch
- Do not alter the data model structure beyond what is described (only adding/modifying fields mentioned above)
- Do not change any existing drag-and-drop behavior
- Do not change any existing localStorage keys — only add to `visions-settings`
- Do not change typography, spacing, or color of any existing elements not mentioned in this patch
- Do not simplify or remove any existing features

---

**Output:** The complete updated single HTML file with all four updates applied. Same file format as original — all CSS in `<style>`, all JS in `<script>` at bottom of body, Google Fonts in `<head>`. Immediately usable by double-clicking on a Mac.**
