# richcal — GitHub-style annual activity heatmap (design)

- Date: 2026-06-23
- Status: approved design, pre-implementation
- Repo: nbdev project (`richcal`), notebooks under `nbs/`, generated modules under `richcal/`

## Purpose

Let people visualize their annual activity over multiple years at a glance — how
much progress / effort they put in — directly in the terminal, in the
**GitHub contribution-graph ("annual usage") style**. It can also show the
remaining (future) part of the window so upcoming time is visible.

richcal is a **pure rendering library**. It does not collect data, provide a
CLI, or run an interactive TUI. Those can be layered on later without changing
the core.

## Scope (v1)

In scope:
- A single Rich renderable, `CalendarHeatmap`, producing the GitHub-style grid.
- Value → level bucketing (auto quantile, with manual override).
- Future region rendering (dim placeholder) bounded by `as_of`.
- Month labels, weekday labels, legend (`Less → More`).

Explicitly out of scope (YAGNI):
- Data-source adapters (git log, JSON, CSV, …) — caller pre-aggregates.
- CLI and interactive TUI (Textual).
- Summary line (total / peak / streak) and title rendering.
- Alternate views (month-calendar / weekly-bars / cumulative).
- A separate `due` dataset and a `today` highlight marker.

Output targets come "for free" via Rich: terminal plus `console.save_svg()`,
`console.save_html()`, `console.export_text()`. No extra implementation.

## Public API

```python
from datetime import date
from collections.abc import Mapping, Sequence

CalendarHeatmap(
    data: Mapping[date, float],          # pre-aggregated, one value per day
    start: date,                         # window start (inclusive), required
    end: date,                           # window end (inclusive), required
    level_max: int = 4,                  # levels are 0..level_max (5 buckets by default)
    thresholds: Sequence[float] | None = None,  # override auto quantile when given
    palette: Sequence[str] | None = None,        # styles for levels 0..level_max; default GitHub green
    as_of: date | None = None,           # day > as_of is a future cell; None = no future region
    show_months: bool = True,
    show_weekdays: bool = True,
    show_legend: bool = True,
)
```

- Returns a Rich renderable (implements `__rich_console__`).
  `console.print(CalendarHeatmap(...))` draws it.
- Keys are `datetime.date`. Missing days inside the window mean "no activity".

Public import surface: `from richcal import CalendarHeatmap` (re-exported from the
package `__init__`). `CalendarHeatmap` is the only public name; `richcal.levels`
and `richcal.layout` are internal and not part of the stable surface.

## Input format

- Strictly `Mapping[date, float]`, pre-aggregated to one value per day.
- No string keys, no raw event lists, no pandas. Aggregation is the caller's job.
  This keeps the core dependency-light (`levels`/`layout` need no third party).
- Keys must be `datetime.date` objects. `datetime.datetime` is a subclass of
  `date` in Python, so it satisfies the type hint, **but** a `datetime` does not
  compare equal to or hash identically with the matching `date`
  (`date(2024,1,1) == datetime(2024,1,1)` is `False`) — it would read as missing.
  Rather than fail silently, the constructor rejects non-`Mapping` `data` and any
  key that is not a `datetime.date` (including `datetime`) with `ValueError`.
  Normalize to `date` before passing data in.
- Keys outside `[start, end]` are ignored for rendering, but key *type* is
  validated for all entries.

## Value → level mapping

Levels are integers `0 .. level_max`. Level 0 is "empty". Higher = more activity.
`level_max` positive levels (`1..level_max`) are separated by `level_max - 1`
boundaries (n groups need n−1 cut points).

`thresholds` is a **non-decreasing** sequence of length `level_max - 1` (ties
allowed; a tie just leaves a level empty). The level of any value:

- `value < 0` or non-finite (`NaN`/`inf`) → invalid, see Error handling.
- `value == 0` → level 0 (empty).
- `value > 0` → `level(v) = 1 + bisect_right(thresholds, v)`, naturally capped
  at `level_max` because there are `level_max - 1` boundaries.

Auto thresholds (when `thresholds is None`):
- Computed from **positive observed values only**: `value > 0` and, when
  `as_of is not None`, `day <= as_of` (future days excluded; missing days
  contribute nothing). When `as_of is None`, all in-window days are observed.
- `thresholds = statistics.quantiles(positives, n=level_max, method="inclusive")`,
  which yields exactly `level_max - 1` ascending cut points.
- Degenerate cases (deterministic, no error): `level_max == 1` → `thresholds = []`
  (every positive value is level 1); fewer than 2 positive observed values →
  `thresholds = []` likewise.

Manual override: pass a non-decreasing `thresholds` of length `level_max - 1`;
the same `level(v)` rule applies. (`level_max == 1` → pass `[]`.)

## Cell kinds & window semantics

The active window is exactly `[start, end]` (both required, inclusive). Every
drawn cell is exactly one of three kinds:

- **padding** — a grid position **outside** `[start, end]` that exists only to
  complete the Sunday-first week columns (see Layout). Rendered as **blank**
  (spacing only). Not level 0, not future.
- **future** — in-window, with `as_of is not None and day > as_of`. Holds no
  value (any `data` entry for a future day is ignored); rendered as a dim
  placeholder glyph (e.g. `·`) so "the rest of the year" is visible. With
  `as_of is None` there is no future region.
- **observed** — in-window and not future. Rendered by its level (`0..level_max`);
  level 0 is a faint empty square.

`as_of` slides the boundary continuously and all positions are valid:
`as_of is None` or `as_of >= end` → no future region (whole window observed);
`start <= as_of < end` → split; **`as_of < start` → the whole window is future**
(every cell a dim placeholder — a legitimate "fully-planned-ahead, no data yet"
view, not an error).

## Layout (auto-switch, GitHub style)

- Weeks run as columns, weekdays as rows; **week starts on Sunday** (top row `Su`),
  matching the reference `/usage` view.
- Auto-switch (testable, calendar-year based):
  - `start.year == end.year` → a single continuous strip.
  - otherwise → one block **per calendar year** from `start.year` to `end.year`,
    stacked vertically with a year label on the left. Each block's active range
    is that year clipped to `[start, end]`.
- Week-column origin / padding: each block's columns span from the Sunday on or
  before its first active day to the Saturday on or after its last active day, so
  every column is a full 7-cell week. Positions before the first active day and
  after the last are **padding** (blank). Thus the grid is rectangular and the
  in-window cell count equals the inclusive day count of the block's active range.
- Chrome (toggleable): month labels on top, weekday labels on the left,
  `Less → More` legend below.
- Cell glyph: `■` (plus spacing) colored by level; level 0 is a faint empty;
  future is the dim placeholder; padding is blank.

## Architecture (three focused modules)

| notebook | module | responsibility | depends on |
|---|---|---|---|
| `00_levels.ipynb` | `richcal/levels.py` | `quantile_thresholds(values, level_max)`, `to_level(value, thresholds)`. Pure. | stdlib only |
| `01_layout.ipynb` | `richcal/layout.py` | date → (year-block, weekday-row, week-col); year splitting; month-label positions. Pure, Sunday-first. | stdlib only |
| `02_heatmap.ipynb` | `richcal/heatmap.py` | `CalendarHeatmap` renderable: composes levels + layout, applies palette, draws labels/legend/future placeholders. | `rich`, `levels`, `layout` |

Keeping `levels` and `layout` Rich-free lets the logic be unit-tested without
Rich, and leaves a clean seam for future views (month-calendar, weekly bars,
cumulative) to reuse them next to `heatmap.py`.

## Data flow (on render)

1. Resolve the window from required `start` / `end`.
2. Resolve thresholds — auto from positive observed values, or use the override.
3. Pick layout — single strip when `start.year == end.year`, else per-year stacks.
4. For each day in the window, compute `(block, row, col)` and either its level
   (observed) or future-placeholder (`day > as_of`); emit a Rich grid with
   month labels, weekday labels, and the legend per the `show_*` flags.

## Error handling

- `data` not a `Mapping`, or any key not a `datetime.date` (including a
  `datetime.datetime`) → `ValueError`.
- `start > end` → `ValueError`.
- `level_max < 1` → `ValueError`.
- `thresholds` (manual) not non-decreasing, non-finite (`NaN`/`inf`), or length
  `!= level_max - 1` → `ValueError`.
- `palette` length `!= level_max + 1` → `ValueError`.
- A `value < 0` or non-finite (`NaN`/`inf`) → `ValueError`, but **only for
  observed in-window days** (`start <= day <= end` and not future). Values on
  future days and on out-of-window keys are never consumed and so are not
  validated — consistent with the auto-threshold carve-out.
- Empty `data` (or all zeros) → render the window's empty frame; do **not** raise.

## Dependencies

- Add `rich` to `pyproject.toml` `[project].dependencies` (currently `[]`).
  Rich is a hard dependency because the public API is a Rich renderable.
- No `pydantic`. Inputs are Python objects from the caller's own code (not an
  untrusted parse boundary), so validation is plain constructor checks raising
  `ValueError`, using stdlib only (`bisect`, `statistics`, `datetime`). pydantic
  would add a heavy compiled dependency, contaminate the stdlib-only
  `levels`/`layout` core, and its coercion would conflict with the strict
  `date`-keys-only rule. Revisit pydantic only if a future CLI / config-file /
  JSON-data layer is added — that is an untrusted boundary outside the core
  renderable.

## Testing (nbdev, stub-first TDD)

- Implement from `nbs/` with stub-first TDD; treat `richcal/*.py` as generated
  output (do not hand-edit the modules).
- Pure functions (`levels`, `layout`): assertion cells for quantile thresholds
  (length `level_max - 1`, ties), `to_level` boundaries (including `v == 0`,
  all-zero, degenerate `level_max == 1`, the `bisect_right` boundary, and
  `ValueError` on negative / non-finite), weekday-row / week-col mapping, leading/
  trailing padding, and year splitting.
- Renderable: render into `Console(record=True)` and assert the **structure
  contract** — presence of weekday rows, week columns, month labels, weekday
  labels, legend; in-window cell count equals the active-range inclusive day
  count; padding cells are blank; a future region appears when `as_of` is set;
  `as_of < start` renders the whole window as future; a negative/non-finite value
  on a future day does **not** raise. Test top-level
  `from richcal import CalendarHeatmap`. Do not
  assert pixel/character-width parity.
