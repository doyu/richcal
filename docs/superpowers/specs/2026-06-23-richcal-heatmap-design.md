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

## Input format

- Strictly `Mapping[date, float]`, pre-aggregated to one value per day.
- No string keys, no raw event lists, no pandas. Aggregation is the caller's job.
  This keeps the core dependency-light (`levels`/`layout` need no third party).

## Value → level mapping

Levels are integers `0 .. level_max`. Level 0 is "empty". Higher = more activity.

Auto thresholds (when `thresholds is None`):
- Compute from **positive observed values only**: `value > 0` and `day <= as_of`
  (when `as_of is None`, all in-window days count as observed).
- `thresholds[k] = quantile(positives, (k+1)/level_max)` for `k in 0..level_max-1`
  (sorted ascending, length `level_max`).
- `level(v) = 0` when `v == 0`; otherwise `1 + (number of thresholds strictly
  less than v)`, capped at `level_max`.
- Future days and missing days are **excluded** from threshold computation.
- If there are no positive observed values, every cell is level 0 (no error).

Manual override: `thresholds` is an ascending sequence of length `level_max`
giving the lower bounds for levels `1..level_max`; same `level(v)` rule applies.

## Future / window semantics

- The drawn window is exactly `[start, end]` (both required, inclusive).
- A day `d` with `as_of is not None and d > as_of` is a **future cell**: it holds
  no value and renders as a dim placeholder glyph (e.g. `·`), distinct from a
  past empty (level-0) cell. This makes "the rest of the year" visible.
- With `as_of is None`, there is no future region; all in-window days render by level.

## Layout (auto-switch, GitHub style)

- Weeks run as columns, weekdays as rows; **week starts on Sunday** (top row `Su`),
  matching the reference `/usage` view.
- Range `<= ~1 year`: a single continuous strip — 7 rows × N week columns.
- Range `> 1 year`: stacked per calendar year — each year is its own 7×53 block
  with a year label on the left.
- Chrome (toggleable): month labels on top, weekday labels on the left,
  `Less → More` legend below.
- Cell glyph: `■` (plus spacing) colored by level; level 0 is a faint empty;
  future is the dim placeholder.

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
3. Pick layout — continuous strip vs per-year stacks by range length.
4. For each day in the window, compute `(block, row, col)` and either its level
   (observed) or future-placeholder (`day > as_of`); emit a Rich grid with
   month labels, weekday labels, and the legend per the `show_*` flags.

## Error handling

- `start > end` → `ValueError`.
- `thresholds` not strictly ascending, or length `!= level_max` → `ValueError`.
- `level_max < 1` → `ValueError`.
- Empty `data` (or all zeros) → render the window's empty frame; do **not** raise.

## Dependencies

- Add `rich` to `pyproject.toml` `[project].dependencies` (currently `[]`).
  Rich is a hard dependency because the public API is a Rich renderable.

## Testing (nbdev, stub-first TDD)

- Implement from `nbs/` with stub-first TDD; treat `richcal/*.py` as generated
  output (do not hand-edit the modules).
- Pure functions (`levels`, `layout`): assertion cells for quantile thresholds,
  `to_level` boundaries (including `v == 0` and all-zero), weekday-row / week-col
  mapping, and year splitting.
- Renderable: render into `Console(record=True)` and assert the **structure
  contract** — presence of weekday rows, week columns, month labels, weekday
  labels, legend; cell counts; a future region when `as_of` is set. Do not
  assert pixel/character-width parity.
