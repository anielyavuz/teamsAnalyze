# UI shell and design system

Visual language: **calm corporate, dense analytics layout**. One theme source defines every colour,
size and component style; no component hard-codes a colour.

## 1. Constraints

- Server-rendered HTML + small vanilla JS. **No external runtime dependencies**: no CDN, no web fonts from
  third parties, no analytics. The font is embedded (Instrument Sans, SIL OFL 1.1, latin + latin-ext
  subsets, including Turkish characters; licence file shipped). Why: works offline/air-gapped; viewers'
  browsers contact nothing but this server.
- All data text escaped (autoescape); JS inserts text with `textContent`; JSON embedded only via the safe
  helper (`scope-masking.md` §5). CSP compatible: no inline event handlers; inline scripts carry the nonce.
- Real `<button>` and `<a>` elements; keyboard reachable; visible focus ring.

## 2. Design tokens (CSS custom properties on `:root`)

| Group | Light | Dark |
|---|---|---|
| Page / card / subtle / hover backgrounds | `#F6F7F9` / `#FFFFFF` / `#F9FAFB` / `#F2F4F7` | `#0B0D12` / `#11141B` / `#151922` / `#1A1F2A` |
| Text fg-1 / fg-2 / fg-3 / disabled | `#12151C` / `#344054` / `#6B7280` / `#9CA3AF` | `#E8EAEE` / `#C3C8D2` / `#9AA1AE` / `#5B6270` |
| Strokes 1 / 2 | `#D0D5DD` / `#E6E8EC` | `#343A4A` / `#232836` |
| Brand (single accent) | indigo `#4338CA`, hover `#3730A3`, bg `#EEF0FF`, fg `#3730A3` | teal `#2DD4BF`, hover `#5EEAD4`, bg teal 12 % |
| Success / danger / warning / incident | `#067647` / `#B42318` / `#8A4B00` / `#8A4B00` (+ tinted bgs) | `#47CD89` / `#F97066` / `#FDB022` / `#FDB022` |
| Data series s1 / s2 / s3 / axis | `#4338CA` / `#B42318` / `#A5A8F0` / `#D0D5DD` | `#12A594` / `#E5484D` / `#7A7CE0` / `#343A4A` |
| Heatmap hm0…hm5 | `#EEF0F4` → `#4338CA` indigo ramp | `#1A1F2A` → `#5EEAD4` teal ramp |

Typography: `--font` Instrument Sans then system UI fonts; mono stack for ids; numbers `tabular-nums`.
Sizes: title 28/36 600, section heading 16/22 600, body 14/20, label 12/16 in `--fg-3`.
Spacing scale 4, 8, 12, 16, 20, 24, 32 px. Radius: card 14 px, control 8 px, small 6 px. Shadows: card
(very light), flyout. Layout: rail width 68 px, drawer width 460 px.

## 3. Themes

- Modes: **light (default)**, dark, auto. A top-bar button cycles Light → Dark → Auto; stored in
  `localStorage` (`ta.theme`) — a per-viewer convenience; read/write wrapped in try/catch.
- Dark = `html[data-theme="dark"]`, or `data-theme="auto"` with `prefers-color-scheme: dark`. Dark values are
  a one-to-one counterpart of light tokens; components never branch on theme.
- `body` always has an explicit background token.

## 4. Layout

```
┌──────┬──────────────────────────────────────────────────────────────┐
│ rail │ top bar: page title · status/freshness line · tools · lang · theme
│ 68px ├──────────────────────────────────────────────────────────────┤
│ icons│ KPI strip (one container, 1px dividers)                       │
│      │ cards grid (16–20px gaps)                          ┌─────────┐│
│ …    │ tables / charts                                    │ detail  ││
│ admin│                                                    │ drawer  ││
│ out  │                                                    └─────────┘│
│ AB   │                                                               │
└──────┴──────────────────────────────────────────────────────────────┘
```

- **Icon rail (left):** one icon button per permitted module in manifest order; tooltip label on hover/focus
  and `aria-label`; active item uses brand bg/fg; badge dot for warnings; locked modules show a lock glyph
  in danger colour. Bottom: Admin, Logs and permissions (if permitted), sign-out, user initials. Rail links
  that are not module tabs use a distinct class from tab buttons (tab JS binds only to tabs). On phones the
  rail becomes a horizontal bar at the top.
- **Top bar:** page title = active module; status line (freshness, see `ingest-scheduler.md` §6; permission
  chip); module tools (search box, date-range segment, export) are rendered by the module into a top-bar slot;
  language and theme switches.
- **KPI strip:** all KPIs in one container separated by 1 px dividers; label, value, optional delta/note.
  Zero/empty values styled muted (`is-zero`), except semantically coloured KPIs.
- **Cards:** title, optional subtitle and info popover, body; consistent padding.
- **Tables (DataTable):** sticky header, numeric columns right-aligned, sortable headers as buttons with
  `aria-sort`, "-" cells muted, columns entirely empty collapsed; bounded rows with "show more"/paging via API.
- **Detail drawer:** right-side panel (≥ 1180 px) showing a record's detail; Escape closes; closes when the
  list re-renders; on narrow screens detail expands inline below the row.
- Lazy modules: shell renders the first permitted module; others load on first open via the tab API; a
  loading skeleton and an error state with retry.

## 5. States

| State | Rendering |
|---|---|
| No data | one grey line (`note-line`) explaining why (e.g. "No calls in this range. Records appear 15–30 min after a call ends.") — no large empty boxes |
| Locked (permission) | locked card (`permissions.md` §6) |
| Limited (optional/probe) | yellow info note inside the section |
| Small group | single note "Small group (fewer than k people) — hidden for privacy" |
| Aggregate | person-level blocks absent; a subtle note "Aggregate view" in the status line |
| Error | card-level error with request id and retry; full details only in the log |
| Stale data | freshness line in warning colour when the last successful run is older than 2× cadence |

## 6. Permission transparency popovers

Shared component used by every module screen (spec of content: `permissions.md` §8).

- **(A) "Which permissions does this need?"** — one per module screen, placed in the module's header/top-bar
  area. **(B) "What did this use?"** — one per section of live screens, in the card header.
- Built on native `<details>`/`<summary>` (works without JS, keyboard accessible); the summary is an info
  icon button with an accessible name ("Which permissions does this need?" / "What did this use?").
- Open state: an absolutely positioned card, **right-aligned** to the trigger, width ~720 px (max
  `calc(100vw − 32px)`), `--bg-card` background, `--stroke-2` border, `--r-card` radius, `--shadow-flyout`,
  above other content (z-index above drawer). On phones (< 600 px) it is full width below the trigger.
  Escape and outside click close it (small JS enhancement); only one open at a time.
- Inside: compact table (permission | required/optional | used for), or for (B) (step | method + endpoint
  template | permission | purpose | status | ms | calls | results). Status uses glyph + text + colour
  (ok success, denied danger, not_found muted, error danger).
- Permission cell (exact rule in permissions.md §8.3): status pill first ("✓ covered" success / "✗ missing"
  danger, 11px, 600 weight); least-privileged option next; alternatives in `--fg-3` "(alternative, one is enough:
  X, Y)"; each option followed by its own mark — green ✓ granted, muted ✗ not granted (name muted too).
  Glyphs have `title`/`aria-label` granted/not granted.
- Works in light and dark through tokens only; screenshot-tested open in both themes and at 390 px.

## 7. Data visualisation rules

- Colour carries meaning only: one brand accent; red (`--danger`) = problem; green = healthy; amber
  (`--incident`) = Microsoft service incident. Series colours are bound to **entities** and fixed across the
  app (e.g. answered `--s1`, missed `--s2`, meetings `--s3`); module docs declare their series mapping.
- Red is used only for problem series. Text is never drawn in a series colour.
- Every chart has a table equivalent (toggle or adjacent) and tooltips; charts have `role="img"` + summary.
- Do not show rates computed from tiny samples (module docs set minimums, e.g. hourly missed-rate needs
  ≥ 3 calls).
- New palettes/series are checked with a data-viz contrast validator against card surfaces `#FFFFFF`
  (light) and `#11141B` (dark); results recorded in the module doc. No judgement by eye.
- Charts are inline SVG/CSS generated server-side or by small JS; no chart library from a CDN.

## 8. CSS naming and safety

- Component classes are prefixed per component (`dt-` tables, `dr-` drawer, `kpi-`, `tl-` timeline,
  `pop-` popovers, `<module>-` for module-specific). Generic names (`.col`, `.row`) are forbidden: a
  collision once broke the detail panel.
- Renaming a class: search for the exact class token (not substring — renaming `col` also hit `cols` once
  and broke every chart); count usages in HTML/CSS/JS before and after; screenshot every screen.
- JS that re-renders a region calls the shared "mark view" helper afterwards (zero/empty markers).

## 9. Accessibility

Contrast ≥ 4.5:1 for text (tokens chosen accordingly; the light `--s3` series colour is below that and is
therefore never used for text). Focus visible. Landmarks (`nav` rail, `header` top bar, `main`). Tables
have header cells and captions. Motion minimal; respect `prefers-reduced-motion`.

## 10. Screenshot checklist (per change)

Light, dark, phone width 390 px (rendered inside an iframe because headless browsers have a minimum window
size), every screen touched; if shared CSS changed, every screen; popovers open; locked, empty and
aggregate states.
