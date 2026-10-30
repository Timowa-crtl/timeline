# Timeline Planner — Prototype Spec v2

Version 2.0-proto · 2026-10-28

## 0. Why this version

We are in rapid-prototyping mode, exploring a simpler item model and UI changes with as little overhead as possible.

- **One file:** `index.html` with inline CSS and JS. Plain JavaScript, no modules, no build, no dependencies.
- **Runs from anywhere:** opened by double-click (`file://`) or hosted anywhere.
- **Storage:** `localStorage`, plus JSON export/import.
- The v1 PWA (GitHub Pages, service worker, IndexedDB, tests, CI) stays as it is and is **not** updated during prototyping. Ideas that work get ported back later.

## 1. Principles

1. **Simple beats complete.** Every field has to earn its place.
2. **Every item has a date.** Its own, or inherited from its parent. No "someday".
3. **Life is messy.** The app suggests order and timing, but never blocks anything.
4. **No assumptions.** A booked day passing doesn’t mean it happened, and it doesn’t mean it didn’t.
5. **Design over dialogs.** State is shown by styling, not by popups or nag lists.

## 2. Scope

**In:**
- items with a due date and an optional "not before" date
- optional booked date (appointment)
- one level of sub-items
- soft dependencies
- repeat after done
- projects and tags
- zoomable timeline
- "Coming up" list
- detail and editor sheets
- filters
- export/import (including v1 files)
- light/dark theme
- phone layout

**Out (for now):**
- habits
- "someday" items
- explicit quotas / counters
- postpone history
- hard vs. soft deadlines
- reminders and notifications
- offline/installable app (PWA), IndexedDB, automated tests
- dragging bars to change dates

## 3. Concepts

| Concept | Meaning |
|---|---|
| **Item** | Something to get done once. |
| **Due** | Latest day it should be done. Required on top-level items. |
| **Not before** | Earliest sensible day. If set, the item is drawn as a **bar** (a window); if not, as a **point** (a deadline). |
| **Booked for** | A concrete appointment or plan date (doctor, cinema). Doesn’t replace the due date; it is a plan within it. |
| **Done** | `completedAt` date + optional note. |
| **Sub-item** | A child of a top-level item (one level). Without its own dates it inherits the parent’s window. |
| **Dependency** | “Suggested order: A before B.” Drawn as an arrow. Never blocks. |
| **Repeat** | “Every N months”: ticking the item off creates the next occurrence. |
| **Project** | A container without dates (e.g. “Home improvement”). |
| **Tag** | A cross-cutting label (health, fun, money). |

The v1 kinds map onto this model as follows. The user never has to choose a kind.
- **Deadline:** due only.
- **Window:** not before + due.
- **Fixed date:** due + booked for, on the same day (or a one-day window).

## 4. Data model

```jsonc
{
  "schemaVersion": 2,
  "settings": { "theme": "system", "weekStart": 1, "dueSoonDays": 14 },
  "projects": [{ "id": "p1", "name": "Health", "color": "#d14d72", "order": 0, "archived": false }],
  "tags":     [{ "id": "t1", "name": "money",  "color": "#2e9e5b" }],
  "items": [
    {
      "id": "i1",
      "title": "Sell keyboards",
      "description": "",
      "parentId": null,          // id of a top-level item, or null
      "projectId": null,         // sub-items always use the parent’s project (field ignored)
      "tagIds": ["t1"],
      "notBefore": "2026-01-01", // optional
      "due": "2026-12-31",       // required for top-level; optional for sub-items
      "bookedFor": null,         // optional
      "completedAt": null,       // ISO date or null
      "note": "",                // completion note
      "repeatMonths": null,      // integer 1–60, top-level only
      "dependsOn": [],           // item ids, soft
      "order": 0,                // position among siblings (sub-items)
      "createdAt": "2026-10-28T09:00:00Z"
    }
  ]
}
```

Dates are `YYYY-MM-DD`. Internally they become day numbers (days since 1970-01-01, UTC), so time zones and daylight saving never matter.

### 4.1 Rules

**Hard: the item can’t be saved if these are broken**
- `title` is 1–200 characters.
- Top-level items have `due`.
- If both are set, `notBefore ≤ due`.
- A sub-item’s parent exists and is top-level. Sub-items have no children.
- `repeatMonths` is only allowed on top-level items, as an integer from 1 to 60.
- `completedAt` is not in the future.
- `dependsOn` contains no self-reference, no unknown ids and no cycles.

**Soft: a warning is shown, saving still works**
- A sub-item’s own dates lie outside its parent’s window.
- `bookedFor` lies outside `[notBefore, due]`.
- `bookedFor` is after `due`.

## 5. Derived values

| Value | Rule |
|---|---|
| `effNotBefore(i)` | `i.notBefore ?? parent.notBefore ?? null` |
| `effDue(i)` | `i.due ?? parent.due` |
| `hasOwnDate(i)` | `due`, `notBefore` or `bookedFor` is set on the item itself |
| `shape(i)` | **bar** if `effNotBefore` is set, else **point** |
| `nextDate(i)` | `bookedFor` if it is today or later, else `effDue` (used for sorting and “soon”) |
| `late(i)` | `completedAt > effDue` |
| `outOfOrder(i)` | done while some prerequisite was still open, or done before the prerequisite’s `completedAt` |
| `progress(parent)` | done sub-items / all sub-items (shown as `k/n`) |

### 5.1 Status (first match wins)

| # | Status | Condition | Look |
|---|---|---|---|
| 1 | **done** | `completedAt` set | green dot at `completedAt` (red dot if late) |
| 2 | **closed** | parent done, this sub-item open | greyed out, strikethrough, no warning colors |
| 3 | **overdue** | `effDue < today` | red |
| 4 | **unconfirmed** | `bookedFor < today` (due not yet passed) | **neutral**: hollow, dashed grey booking square; no red, no list, no popup |
| 5 | **soon** | `nextDate ≤ today + dueSoonDays` | amber |
| 6 | **open** | otherwise | neutral |

The **out of order** flag can appear on top of done: the arrow is drawn in a distinct style (dotted, with a small “↺” mark) and a note appears in the detail sheet. It is never shown as an error.

## 6. Behaviour

### 6.1 Ticking off
- Opens a small sheet with a “Done on” date and a note.
- Default date is `bookedFor` if that day has passed, otherwise today. It can be edited but not set in the future.
- Prerequisites never block ticking.
- **Parent with open sub-items:** the sheet shows a checkbox “Also mark the k open sub-items done”, off by default. Sub-items left open become **closed**.
- Ticking every sub-item does **not** tick the parent. The parent’s row and detail sheet show “all sub-items done”.
- **Untick:** clears `completedAt` and keeps the note. If this undoes a repeat, the next occurrence that was already created stays (the user can delete it).

### 6.2 Repeat after done (top-level only)
When an item with `repeatMonths = N` is ticked off, a new item is created:
- `due` = `completedAt + N months`. If the day doesn’t exist in that month, it becomes the month’s last day (Jan 31 + 1 → Feb 28/29).
- `notBefore`: if the original had a window, the new one keeps the same length (`due − (old.due − old.notBefore)`); otherwise none.
- Copied: title, description, project, tags, `repeatMonths`.
- **Not** copied: `bookedFor`, `completedAt`, note, `dependsOn`.
- Sub-items are copied, unticked, with their own dates removed.
- A toast says: “Next: Dentist checkup, due Sep 12, 2027”.

### 6.3 Sub-items
- Created from the parent’s detail sheet (“+ Sub-item”, a quick inline title field), or in the editor by choosing a parent.
- Sub-items can be reordered with ↑ ↓ in the parent’s detail sheet.
- A top-level item that has sub-items can’t itself become a sub-item.
- **Deleting a parent** asks whether to delete it together with its sub-items or keep the sub-items (they become top-level and need a due date: they take their own date if they have one, otherwise the parent’s due).

### 6.4 Dependencies
- Any item can depend on any other item, sub-items included. Cycles are rejected.
- Drawn as a curved arrow from the end of A to the start of B.
  - **solid:** A is done
  - **dashed:** A is open
  - **dotted ↺:** B was done out of order
- They have no effect on dates or ticking.

### 6.5 Delete
- Two-click confirmation, then a toast with Undo (7 s).

## 7. UI

### 7.1 Layout
- **Desktop (> 760 px):**
  - top bar: filters, zoom, Today, + New, ⚙
  - timeline on the left
  - “Coming up” panel on the right (330 px, can be collapsed)
- **Phone (≤ 760 px):**
  - two tabs: **Coming up** (default) and **Timeline**
  - filters and zoom in a bottom sheet
  - all other sheets also open as bottom sheets

### 7.2 Timeline
- Rows are grouped by project (projects in user order, then “No project”).
- Inside a group, rows are sorted by `effNotBefore ?? effDue`, then by `effDue`.
- **Label column** (sticky): status dot, title, `k/n` for parents, ▸/▾ to expand.
- **Zoom:** Day / Week / Month / Year presets, Ctrl/⌘ + wheel, pinch. Zooming keeps the date under the pointer in place.
- **Today:** a red line, and a “Today” button that scrolls today into view.
- **Drawing per item:**

| Element | Drawing |
|---|---|
| Window (`notBefore` → `due`) | rounded bar in the project color (tag color, then grey, as fallbacks), 20% fill |
| Deadline (due only) | diamond at `due` |
| Booked | rounded square at `bookedFor`; hollow and dashed grey once unconfirmed |
| Done | filled dot with a check mark at `completedAt`; a dashed connector to `due` when late |
| Item label | title written to the right of the item’s shapes |

- **Parent row, collapsed (default):**
  - its own shapes
  - plus small dots for done sub-items (at their `completedAt`)
  - plus small markers for open sub-items that have their own date
  - undated open sub-items aren’t drawn; they only count towards `k/n`
- **Parent row, expanded:** one indented row per sub-item.
  - Undated sub-items show as a faint, dotted copy of the parent’s window (“somewhere in here”).
  - Rows that are expanded are remembered in the view settings.
- Tapping a row or shape opens the detail sheet. Rows and shapes can be focused with the keyboard; Enter opens them.

### 7.3 Coming up
Shows open items in these sections:
- **Overdue**
- **Unconfirmed** — styled neutrally, placed after Overdue; only shown if there are any
- **Next N days**
- **Next 3 months**
- **Later**

Rules for the list:
- Sorted by `nextDate`.
- Includes:
  - top-level items
  - sub-items that have their **own** date
  - undated sub-items are represented only by their parent’s `k/n`
- Each row has:
  - a tick button
  - the title (plus the parent’s title in small text for sub-items)
  - a color dot
  - “booked Nov 3” / “by Dec 31” / “Dec 1 – Dec 31”
  - the relative time
- A section **Done · last 30 days** lists recent completions with their notes.

### 7.4 Detail sheet
- **Header:** title, project and tag chips, date line (“Dec 1 → Dec 31 · booked Dec 18”), repeat badge (“↻ every 6 mo”).
- **Description**
- **Sub-items:** checklist with tick boxes, inline “+ Sub-item”, ↑ ↓ to reorder, own dates shown where set.
- **Depends on / Blocks:** links to the other items; “out of order” note where it applies.
- **Status line:**
  - done date and note, or
  - “unconfirmed — booked Nov 3, not ticked yet”
- **Actions:** Done / Undo, Edit, Duplicate, Delete, Close.

### 7.5 Editor
Fields, top to bottom:
1. Title
2. **Due** (required; for sub-items optional, with a hint “empty = parent’s window”)
3. **Not before** (optional; a “clear” button)
4. **Booked for** (optional)
5. Parent (select; top-level items only; hidden for items that have sub-items)
6. Project (hidden for sub-items)
7. Tags
8. Repeat every [ ] months (top-level only)
9. Depends on (checklist)
10. Description

Above the date fields, a one-line live preview shows how the item will be drawn (“Window, 31 days · booked Dec 18”). Soft warnings appear inline in amber; hard errors in red, which block saving.

### 7.6 Settings (minimal)
- Projects and tags: add, rename, recolor, reorder (projects only), archive, delete.
- Theme, week start, “due soon” days.
- Data: export JSON, import JSON (v1 or v2; preview, then replace all), load demo data, delete all.

### 7.7 Visual language

| Meaning | Color | Shape |
|---|---|---|
| open | neutral (project color for the bar) | hollow diamond / square |
| soon | amber | filled diamond |
| overdue | red | filled diamond |
| unconfirmed | grey | dashed hollow square |
| done on time | green | dot with ✓ |
| done late | red | dot with ✓ + dashed connector |
| closed (with parent) | grey | strikethrough |
| dependency open / done / out of order | muted | dashed / solid / dotted ↺ |

Color is never the only signal: every state also differs in shape, line style or text.

## 8. Persistence & import

- `localStorage['tp2.state']` holds the whole document; `localStorage['tp2.ui']` holds view settings (filters, zoom, expanded rows, active tab).
- Saving happens 300 ms after the last change. If storage is unavailable, the app keeps working in memory and shows a warning banner.
- **Export:** `timeline-planner-YYYY-MM-DD.json`.
- **Import v2:** check the file against the rules, show a preview, replace everything.
- **Import v1** (`schemaVersion: 1`), converted as follows:

| v1 | v2 |
|---|---|
| `deadline` | `due` = checkpoint due; done date and note copied |
| `date` | `due` = `bookedFor` = date |
| `window` with 1 checkpoint | `notBefore` = start, `due` = checkpoint due |
| `window` with n > 1 checkpoints | a parent (`notBefore` = start, `due` = last due) with n sub-items `“<title> #k”` (each with its own `due`, done date, note). The parent is done only if all were done (done date = the last completion). |
| projects, tags, `dependsOn` | unchanged |

## 9. Code structure (inside the one file)

```
<style>     tokens (light/dark) · layout · timeline · sheets · phone
<body>      top bar · tabs · timeline panel · side panel · sheet container · toasts
<script>
  // 1 CONFIG      constants, zoom presets, palette
  // 2 DATES       day numbers, formatting, addMonths
  // 3 MODEL       pure: derived values, status, rules (validate), repeat, v1→v2 convert
  // 4 STORE       state, update(mutator) → validate → save → render, export/import
  // 5 VIEW        timeline(), upcoming(), detail(), editor(), settings(), sheet(), toast()
  // 6 EVENTS      event delegation, zoom/pinch, shortcuts (N, T, 1–4, Esc)
  // 7 DEMO        demo data built around today
  // 8 BOOT
</script>
```

- The MODEL and DATES sections contain no page code, so they can be moved back into modules later unchanged.
- One state object. `update(fn)` edits a copy, validates it, and only then replaces the state and redraws.
- Target size: well under 2,000 lines.

## 10. Demo data

The demo shows every concept around today’s date:

| Item | Shows |
|---|---|
| Dentist checkup | due Mar 31, booked Mar 12, done; repeat every 6 months → next one open |
| Blood checkup → Flu shot | dependency; flu shot done before the blood checkup = **out of order** |
| Watch Dune: Part Three | window Dec 1–31, booked Dec 18 |
| Sell keyboards | window Jan 1–Dec 31; 4 undated sub-items, 3 done |
| CS50W final project | parent with dated sub-items in a dependency chain |
| SKÅDIS #1 → #2 → Choose new TV | dependency chain in “Home improvement” |
| One booked item from last week, not ticked | **unconfirmed** |
| One overdue item | **overdue** |

## 11. Prototype experiments (UI questions to settle by trying)

1. Unconfirmed styling: dashed hollow square vs. faded square vs. a “?” badge.
2. Sub-items on the collapsed parent bar: dots only vs. dots + ticks for open dated ones vs. a progress fill.
3. Expanded sub-rows: indented with a connector line vs. just indented.
4. Out-of-order mark: dotted arrow vs. a small ↺ badge on the item.
5. The editor’s live preview line: is it enough, or should a mini timeline strip show the dates?
6. Phone: is “Coming up” as the start screen right, now that bookings exist?

A hidden **style lab** (`#lab` in the URL) shows a small panel with toggles for 1–4, so the variants can be compared live.

## 12. Decisions (from the interview)

| Topic | Decision |
|---|---|
| Item kinds | Replaced by fields: `due` (required), `notBefore` (optional), `bookedFor` (optional). |
| Someday items | Not supported. |
| Postpone history | Not now. |
| Dependencies | Suggested order only; never block; never move dates; out-of-order completion is marked quietly. |
| Booked date | An optional plan date on any item; the original due date stays. |
| Booked day passed, not ticked | Neither done nor missed is assumed; neutral “unconfirmed” styling only. |
| Done date when ticking a booked item afterwards | Defaults to the booked day. |
| Hard vs. soft deadlines | Not now. |
| Habits | Out of scope. |
| Quotas / checkpoints | Removed; use a parent item with sub-items. |
| Sub-items | One level; inherit the parent’s window; own dates optional (soft warning if outside); parent ticked manually; shown on the parent bar, expandable. |
| Recurring obligations | Repeat after done, every N months. |
| Parent ticked with open sub-items | Optional “also mark done” checkbox (off); leftovers become *closed*. |
| Repeating a parent | Sub-items are copied, unticked, without their own dates. |
| Repeating a window | The window keeps its length. |

## 13. Open questions

- Should a sub-item be able to belong to a different project than its parent? Currently: no.
- Should “unconfirmed” items fade into normal open styling after X days, or stay marked until the due date?
- Is a fixed-length window (“within 2 weeks after booking”) ever needed, or is due + booked enough?

## 14. Changes in v2.1 (2026-10-29)

| Topic | v2.1 behaviour |
|---|---|
| Repeating items | **One item, one row.** Ticking it off moves the current occurrence (`due`, `notBefore`, `bookedFor`, done date, `note`) into `history[]` and sets the next `due`. Past occurrences show as done dots on the same row (red if late). Nothing ahead of the current due date is drawn. |
| Next due | Per item: `repeatFrom: "done"` (default; done date + N months) or `"due"` (previous due + N months, a fixed rhythm). A window keeps its length. |
| Ticking a repeating item | One tap, no sheet. Done date = the booked day if it has passed, otherwise today. Toast with **Undo**. |
| Undo | “Undo last done” in the item sheet brings the last history entry back as the current occurrence. |
| History | “History” list in the item sheet; each entry’s date and note can be edited, and an entry can be deleted. |
| Sub-items of a repeating item | Unticked again on each repeat; their own dates and notes are cleared. Their old completions aren’t kept. |
| Schedule | “Schedule…” in the item sheet (any open item, sub-items included) opens a date picker. Once booked: “Reschedule…” and “Remove booking”. Date only. Warns if the date is after the due date. |
| Tags | “+ New tag” chip in the editor: type a name, press Enter; it gets the next palette color and is selected. An existing tag with the same name is reused. |
| Focus | Opening an item sheet (or the editor for an existing item) doesn’t focus any control. |
| Schema | `schemaVersion: 3`. New item fields: `repeatFrom`, `history: [{id, due, notBefore, bookedFor, completedAt, note}]`. A v2 document is converted automatically: separate items with the same title and project that repeat are merged into one item with a history, and dependencies on the merged items are redirected to it. v1 files are converted to v2 and then to v3. |

## 15. Changes in v2.2 — sorting (2026-10-29)

| Topic | Behaviour |
|---|---|
| Sort options (timeline rows) | **Due date** (default) · **Next date** (booked date if it’s still ahead, else due) · **Start date** (“not before”, else due). Earliest first. |
| Group by project | Switch, on by default. On: sorted within each project section. Off: one list, no section headers. |
| Done items | Always at the bottom of their section/list, ordered by done date. |
| Overdue | No special rule; they sort to the top by date. |
| Ties | Due date, then title. |
| Sub-items | Keep their manual ↑↓ order when expanded. |
| Coming up | Unchanged (urgency sections, sorted by next date). |
| Control | Top bar next to the filters (on a phone in the filter sheet); remembered in the view settings (`tp2.ui`). |

## 16. Changes in v2.3 — icons & colors (2026-10-29)

The goal is that no one needs a legend to read the timeline. Each shape says what kind of date it is, and color is only used where something needs attention.

| Meaning | Look |
|---|---|
| Deadline (due) | **Flag** at the due date: outline when open, **amber** when due soon, **red** when overdue |
| Booked (appointment) | **Calendar tile** showing the day number |
| Booked day passed, not ticked | Grey dashed tile with **“?”** |
| Done | **Check mark in quiet grey** (no green). Done items fade back. |
| Done after the deadline | A faded flag stays at the deadline. No red, no connector line. |
| Repeating item, past occurrences | Grey checks on the same row |
| Sub-items on a collapsed parent | Small grey checks (done) / small hollow dots (open, with their own date) |
| Window | Bar in the project color (says what the item belongs to, not how urgent it is) |
| Dependency, prerequisite open | Solid **dark** grey arrow |
| Dependency, prerequisite done | Solid **light** grey arrow |
| Done before its prerequisite | **Dotted light** grey arrow. Never a warning color. |

- The legend in the Coming up panel is removed.
- The style lab keeps only the two remaining open questions: how sub-items show on a collapsed parent, and the connector line on sub-item rows.
- Filter labels shortened to “grouped” and “done” so the top bar fits on one line at 1,400 px.

## 17. Changes in v2.4 — windows in the past (2026-10-29)

- **The part of a window before today** is drawn as a **dashed grey outline with no fill**. From today on it keeps the project colour (20% fill + outline).
- **A done window** is entirely dashed grey outline.
- **A window that is fully past but still open** is entirely outline; its red flag shows it’s overdue.
- **Undated sub-items** keep their faint dashed copy of the parent’s window.
- Chosen for calmness: urgency is carried by the flags, so the bars can stay quiet (design canvas, “Windows 3”).

## 18. v2.5 — sub-items in the bar, elbow dependencies, legend (2026-10-30, built)

Chosen from the design canvas (“Sub-items inside the bar”, “Dependencies 1 · rounded elbows”).

### 18.1 Sub-items on a collapsed parent row

| Rule | |
|---|---|
| Position | **Centred on the parent’s row** (on the bar), not below it. Size unchanged (small, ~8–10 px). |
| Open sub-item | Small **square check box**: white fill, grey outline, rounded corners (radius ≈ 2 px). |
| Done sub-item | Small **round grey tick**, the same as a done item, only smaller. |
| Where markers go | **Done** sub-items sit at their done date. **Open** sub-items with their own date sit at that date (booked date if it’s still ahead, otherwise due date). **Open sub-items without a date** are spread evenly over what’s left of the parent’s window: from the later of today and the window start, up to the due date. A deadline-only parent (no bar) spreads them from today to its flag, on the row line. |
| Movement | Undated boxes move along each day as today advances, so they always read as “still to do, somewhere in here”. |
| Expanded parent | Unchanged: each sub-item gets its own row (§7.2). |
| Labels | The parent’s row label keeps `k/n`. |
| Style lab | The “subs on parent” option is removed (decided). |

### 18.2 Dependency connectors — rounded elbows

Arrows run at right angles with rounded corners, like a project plan, and replace the curved arrows.

**Start point (prerequisite A)**
- **A done:** just right of A’s done tick (+8 px).
- **A open window:** A’s window end.
- **A open deadline:** just right of A’s flag pole (+4 px).

**End point (dependent B)** — the line stops short of B’s first shape, so it never runs into an icon:
- **B is a window:** 4 px before the window start.
- **B is a deadline:** 7 px before the flag pole.
- **B was done before A:** 10 px before B’s done tick.

**Route**

| Case | Route |
|---|---|
| B starts at least 16 px after A ends | Horizontal from A for 8 px → corner → straight down/up to B’s row → corner → horizontal into B. |
| B starts before (or just after) A ends | Horizontal 8 px → corner → down to the gap between A’s row and the next row → corner → back left along that gap to 10 px before B’s start → corner → down to B’s row → corner → horizontal into B. The line travels in the gaps between rows, never diagonally through other items. |
| A and B on the same row | No line, as before (collapsed sub-items map to their parent’s row). |

**Look**
- **Corners:** radius 5 px. Line width 1.5 px, round caps and joins.
- **End marker:** a small open **chevron** (›, 5 × 8 px). No filled arrowheads.
- **Line style** (unchanged from §16):
  - **dark grey solid:** A still open;
  - **light grey solid:** A done;
  - **light grey dotted:** B was done before A.
- **Undated sub-item as target:** the end point is “just after A” (A’s end + 30 px), as before.
- **Hover/tap** on a connector shows its tooltip: “A → B” (plus “(done before it)”).

### 18.3 Possible follow-up (not planned)

A setting “Show dependencies: always / on hover”. With *on hover*, lines appear only for the hovered or selected item’s chain, and the rest of the timeline shows small link marks (design canvas, “Dependencies 3”).

### 18.4 Legend returns (Coming up panel)

The timeline must be readable without it. The legend is a reference for checking, not a requirement.

- **Place:** at the bottom of the Coming up panel, under the last section, as in v1. On a phone it’s at the bottom of the Coming up tab.
- **Form:** a small collapsible block “Legend ▾” (a native `<details>`), **open by default**. Whether it’s open or closed is remembered in the view settings (`tp2.ui`).
- **Entries:** each one drawn with the real timeline shapes at timeline size, so it always matches what’s on screen.

| Icon | Text |
|---|---|
| Outline flag | due date |
| Amber flag | due soon |
| Red flag | overdue |
| Calendar tile with day number | booked |
| Grey dashed tile with “?” | booked day passed, not ticked |
| Grey round tick | done |
| Faded flag + grey tick | done after the due date |
| Coloured bar, dashed grey before today | time window (past part dashed) |
| Small square box / small round tick on a bar | sub-item open / done |
| Dark solid elbow › | do after (prerequisite open) |
| Light solid elbow › | prerequisite done |
| Light dotted elbow › | done before its prerequisite |

- **Below the legend:** the keyboard hint stays (desktop only): “Ctrl/⌘ + scroll or pinch to zoom · N new · T today · 1–4 zoom”.
- **Dark mode:** the entries use the same CSS colour variables as the timeline, so they adapt automatically.

### 18.5 Today line (v2.5.1)
The red today line (and the TODAY pill) sit at the **start** of today’s day column, not its middle. It matches the boundary where a window switches from dashed (past) to coloured, and today’s whole column counts as time still available.

## 19. v2.6 — streamlined item dialog (2026-10-30)

From the brainstorm (`notes-new-item-dialog.md`, decisions D1–D15) and the design canvas (“New item · quick”, “Edit item · all sections open”, “New item · phone bottom sheet”). The same dialog is used for **new** and **edit**.

### 19.1 Up front: title + due
- **Title** field. **Enter saves** (the whole dialog is one form).
- **Due chips:** `+1 week` · `+2 weeks` · `End of month` · `End of quarter` · `End of year` · `Pick date` (calendar).
  - Each chip has a tooltip with its resulting date.
  - The chip matching the current due date is highlighted.
  - A custom date shows as a selected date field.
- **No default due date** for new items.
- **Due line under the chips:** “by Mon, Nov 30 · in 31 days”.
- **Missing due date on save:** nothing is saved, the due line turns red (“Pick a due date”) and focus moves to the first chip.
- **Sub-items:** an extra first chip **“Parent’s window”** (selected = no own due date). The due date is optional.

### 19.2 Expandable sections (collapsed by default, also when editing)
Each section header shows a **summary** of what’s set, or a grey hint of what it contains.

| Section | Contents | Summary example |
|---|---|---|
| **When** | **Not before** chips: `Today` · `Start of next month` · `Pick date` · `Clear`. **Booked for** (date field, ✕). **Repeat after done** (every N months, top-level only) + **from done date / due date**. **Preview line**, shown only when one of these is used: “Window · Dec 1 – Dec 31 (31 days) · booked Dec 19 · ↻ 6 mo”. | “starts Dec 1 · booked Dec 19 · ↻ 6 mo” |
| **Organize** | **Project** as single-select chips (click again to deselect) + **“+ New project”**: an inline name field, Enter creates the project with the next palette colour and selects it. **Tags** chips + “+ New tag” (as before). **Sub-item of** select, **edit only** (hidden for items that have sub-items). For a sub-item, the project chips are replaced by a fixed line “Sub-item of X”. | “■ Health · #fun #money” |
| **Details** | **After (suggested order):** chosen items as removable chips, a **search field**, and results grouped as **this project** first, then **other**, newest first, at most 8, with the match highlighted, ticked to add. **Description.** | “after Blood checkup +1” |

- **Warnings and errors** (outside the parent’s window, booked after due, cycles…) appear above the footer.
- **Footer:** hint “Enter ↵ saves”, Cancel, Save. On a phone the footer is **pinned** to the bottom of the sheet and the due chips grow to touch size.
- **After saving:** the item sheet opens (as before).

### 19.3 Prefill from context
- **Active project filter** → project.
- **Double-click on empty timeline space** (desktop) or **long-press** (touch, 550 ms, cancelled by movement) → due = that day; project = that row’s project (a project section header, or the project of the item in that row).
- **Parent’s item sheet:** next to the inline “Add sub-item” field, a **“More…”** button opens the full dialog with the parent set.

### 19.4 Removed
- The always-visible fields (not before, booked, sub-item of, project select, repeat, after list, description), the +30 days default, and the always-visible preview line.

## 20. v2.7 — item sheet: act → look → manage (2026-10-30)

From `notes-item-sheet.md` (S1–S15) and the design canvas (“Item sheet · …” boards).

### 20.1 When it opens
- **Only when an existing item is clicked or tapped**: timeline row, bar, sub-item marker, or Coming up row.
- **After creating** an item the dialog **closes silently**.
- **After editing**, the sheet of that item opens again.

### 20.2 Layout
1. **Header:** title + **one status line**, with a ✕ in the corner (also Esc, or tap outside).
   - The line starts with a **small flag** in the item’s state colour.
   - Then the state, the first match of:
     - **done:** “Done Oct 30 · 5 d late”
     - **closed:** “Closed with its parent”
     - **repeating, just ticked:** “Next due Apr 30, 2027 · in 6 mo”
     - **overdue:** “Overdue · due Oct 21 (9 d ago)” (red)
     - **window:** “Oct 20 – **Dec 3** · in 5 wk”
     - **deadline:** “Due **Dec 31** · in 2 mo”
     - **sub-item without own dates:** “somewhere in Oct 20 – Dec 3”
   - Then, if they apply:
     - the **booked chip** “📅 booked Dec 19 ✕” (dashed with “· not ticked yet” once the day has passed; ✕ removes the booking);
     - the **project** “■ name”;
     - for parents, **“1 of 4 done”**;
     - for sub-items, a leading **“in Parent”** link.
   - **Tag chips** and a **repeat chip** (“↻ every 6 mo · from done date”) appear below, only if set.
2. **Inline rows**:
   - **Book:** “📅 Book for [date] · saves on pick ✕”. It opens from the **Book** button and saves on change.
   - **Done on:** “✓ Done on [date] · change if it was another day”.
     - It shows for every done item.
     - For a repeating item it shows right after ticking in the sheet, and edits that history entry.
     - For a done parent with open sub-items there’s an extra link: “also tick N open sub-items”.
3. **Sections, only with content**, in this order:
   - **Sub-items:**
     - each row has a square box (open) or a round tick (done), one tap to toggle;
     - the name opens the sub-item, the date shows on the right;
     - ↑↓ appear on hover (always faintly on touch);
     - the last row is “+ Add sub-item…” (Enter) with a **More…** button that opens the full dialog.
   - **Order:** “**After:** ✓ Blood checkup” / “**Before:** Choose new TV” (tappable), plus “· done before it” when it applies.
   - **Notes:** an inline text area that saves automatically (400 ms after typing stops) and grows with its content.
   - **History** (repeating items): date chips, newest first. A ✎ marks old entries that carry a note. Tapping a chip edits or deletes that entry.
4. **Quiet links** for what’s empty: “+ Sub-item · + Note”.
5. **Action row:**
   - open item: **[✓ Done] [📅 Book] [✎ Edit] … [⋯]**
   - done state: **[↶ Undo] [✎ Edit] … [⋯]**
   - closed item: [Edit] [⋯]
   - **⋯ menu:**
     - **Duplicate**
     - **Undo last done** (repeating items with history, when not just ticked)
     - **Delete…** (red; two clicks; with sub-items: Cancel / Keep sub-items / Delete all; Undo toast)
   - On a phone the row is **pinned to the bottom** of the sheet.

### 20.3 Done is one tap everywhere
- The **Done** button in the sheet and the **tick** in Coming up both mark the item done right away. The date is the booked day if it has passed, otherwise today.
- **Coming up** shows a toast with **Undo**.
- **The sheet** switches to the done state, where the done date can be changed inline.
- **Repeating items** move to their next due date as before.
- The separate “Done…” and “Schedule…” sheets are removed.

### 20.4 Notes (schema 4)
- `description` and the done `note` are merged into one **`notes`** field (max 4,000 characters).
- **Converting older data (schema 1–3):** `notes = description + "\n\n" + note`, leaving out empty parts.
- **Repeating items:** notes **stay** on the item when it’s ticked off. New history entries store only the date; notes already in old history entries stay readable and editable.
- The item dialog’s **Details** section edits the same Notes field.

### 20.5 Today line, again (v2.7.1)
Replaces §18.5. The today line and the TODAY pill sit in the **middle** of today’s day column again. The point where a window switches from dashed grey (past) to colour moves to the **same middle**, so the line and the switch align. A window that starts today shows half a day of dashed grey before the line.

## 21. v2.8 — wording, arrows toggle, tag dropdown

- **Wording:** every visible "book/booked/booking" is now "schedule/scheduled". Data field stays `bookedFor` (no migration). Sheet button "Schedule", inline row "Schedule for", chip "scheduled Dec 20 ✕" (aria "Unschedule"), legend "scheduled" / "scheduled day passed, not ticked", editor field "Scheduled for", warnings "Scheduled after the due date." / "Scheduled before “not before”."
- **Arrows toggle:** checkbox "arrows" next to grouped · done (top bar and phone Filters sheet). `UI.showDeps` (default true, saved in `tp2.ui`). Off hides only the timeline dependency arrows; item sheet still lists After/Before; legend unchanged.
- **Tag filter:** tag chips replaced by a multi-select dropdown styled like the project select. Button: "All tags" / "fun, money" (≤2) / "N tags". Menu: one checkbox per tag (color dot) + "Clear" when anything is selected. OR semantics (`UI.tags`). Closes on outside click or Esc (Esc in the phone sheet closes the menu first, then the sheet). In the phone sheet the button is full width and the menu renders inline.

## 22. v2.9 — every item is a window (2026-10-30)

Replaces the bar/point split from §3 and §5 (`shape(i)`): there are no point items any more.

| Topic | Behaviour |
|---|---|
| Default start | Every item with its own due date has a `notBefore`. If none is picked, it is the **creation day**, stored on save. The bar keeps that fixed start; its past part turns dashed as days go by (§17). |
| Past due date | If the due date is before the creation day, the default start is the due date (one-day window). |
| Existing data | On load and import (schema 1–4), items with a due date but no `notBefore` get `notBefore = min(createdAt day, due)`. No schema bump. |
| Editor | “Not before” chips: `Today` · `Start of next month` · `Pick date`. **Clear is removed** for items with their own due date; while nothing is picked, the default is pre-selected with a hint “default: today” / “default: the due date”. The When summary always shows “starts …”. |
| Sub-items | Undated sub-items still inherit the parent’s window. A sub-item that picks its own due date gets the same default start. Clear stays only for a sub-item that keeps the parent’s due date but had picked its own start. |
| Repeat | Unchanged: the next occurrence keeps the window’s length. |
| Labels | A one-day window (start = due) is written as a single date (“by Oct 25” / “Due Oct 25”), not “Oct 25 – Oct 25”. |
| Validation | Unchanged: an explicit start after the due date is a hard error. |

## 23. v2.10 — quick scheduling from the timeline (2026-10-30)

| Topic | Behaviour |
|---|---|
| Done + scheduled | Once an item is done, its calendar tile is no longer drawn; the done tick is enough. The `bookedFor` value is kept. |
| “Not scheduled” tile | Open **top-level** items without `bookedFor` show a grey calendar tile with **“?”** (no band, solid outline). Not shown on sub-item rows, parents with sub-items, overdue, closed or done items. |
| Position | Middle of what’s left of the window: `floor((max(today, notBefore) + due) / 2)`. Moves along as today advances. |
| Look | Faint (40% opacity) by default; full opacity when hovering the item’s row; outline darkens when hovering the tile. Distinct from the *unconfirmed* tile (dashed outline). |
| Click | Opens the native date picker at the pointer, preset to the tile’s day. Picking a date saves `bookedFor` immediately; toast “Scheduled Nov 2” with **Undo** (7 s). Cancelling does nothing. Clicking elsewhere on the row still opens the item sheet; Enter on a focused row too. |
| Legend | New entry: “not scheduled yet (click to schedule)”. |

## 24. v2.11 — pickers only, Schedule opens the picker (2026-10-30)

| Topic | Behaviour |
|---|---|
| Schedule button (item sheet) | Opens the native date picker directly, anchored at the button, preset to the scheduled day or `max(today, start)`. Picking saves; cancel does nothing. The inline “Schedule for …” row is removed. Label reads **Reschedule** when a date is set. |
| Date entry | Dates are only picked, never typed: every date field opens the picker on click, Space or ↓; other keys and paste are ignored (Tab, Esc, Enter still work). |
| “?” tile | Same look as the scheduled calendar tile (dark outline and band), with **“?”** instead of the day number. **Hidden** until the item’s row is hovered (or focused); not shown on touch devices — use the Schedule button there. Click behaviour as in §23. |
| Legend | “not scheduled — on hover, click to schedule”. |
