# Implementation instructions (v1)

This document guides implementers. The **authoritative functional specification** is
[`specs_v1.md`](specs_v1.md). If anything here disagrees with the spec, follow `specs_v1.md`.

## What you are building

Implement the v1 behaviors, CLI surface, and tests described in `specs_v1.md`.

## Recommended package layout

Use a `src/` layout with the library under `src/lappy/` and tests at the repository root:

```text
repo/
├── src/
│   └── lappy/
│       ├── __init__.py
│       ├── cli.py
│       ├── ids.py
│       ├── io.py
│       ├── ingest.py
│       ├── merge.py
│       ├── validate.py
│       └── schema.py
├── tests/
└── data/
    └── [goal]/
        └── …
```

`tests/` must not live inside `src/lappy/`. The `data/[goal]/…` tree is specified under **Data
directory layout** in `specs_v1.md`; mirror that shape when wiring paths in `io.py`.

Module responsibilities:

- `cli.py`: argument parsing and command dispatch
- `ids.py`: `workout_id` generation and related helpers
- `io.py`: CSV reading and writing, file discovery, filename parsing, and date inference
- `ingest.py`: trimming and normalization of Garmin activity CSVs into per-workout CSVs
- `merge.py`: idempotent merging of normalized per-workout CSVs into aggregate CSVs
- `validate.py`: schema validation and semantic validation
- `schema.py`: canonical column orders and field definitions

The CLI entrypoint is `lappy.py`; keep handlers thin and delegate to these modules.

## Example Garmin exports (`examples/`)

Use these as **fixtures** and for understanding raw formats. They are **not** exhaustive of every
Garmin export variant.

### Garmin workout exports vs manual (generic) activities

**Checked in this repo (examples only):** headers were read with Python’s `csv` module.

- **Garmin workout** lap exports (`activity_garmin_running_coach_plan_activity.csv` and
  `activity_strides_outside_garmin_coach_plan.csv`) share the **same layout**: single-line header,
  leading empty column, then `Interval`, `Step Type`, `Lap`, `Time`, … through `Avg Moving Pace`.
  The two files are **not** byte-identical: the coach plan sample includes one extra column,
  `Avg Vertical Ratio`, after `Avg Vertical Oscillation`; the strides-outside-coach sample **omits**
  that column, so later columns shift left but names stay aligned.
  `activity_22390785030.csv` matches the strides-outside-coach header **exactly** (29 columns).

- **Manual activity** lap export (`activity_running_generic.csv`) is a **different** format: 28
  columns, first column `Laps`, **multi-line** quoted header cells, **no** `Interval` or `Step Type`
  (lap index only, not the workout step grid).

Do not assume every Garmin Connect export matches these three files; treat this as evidence, not a
full survey.

A **manual** run saved as a normal activity (not as a Garmin workout) matches the generic pattern
above.

| File | Role |
| ---- | ---- |
| `activity_running_generic.csv` | Manual activity lap export: different shape, no `Step Type`. |
| `activity_garmin_running_coach_plan_activity.csv` | Coach plan Garmin workout (30 columns). |
| `activity_strides_outside_garmin_coach_plan.csv` | Strides Garmin workout, not coach (29 cols). |
| `activity_22390785030.csv` | Spec example; same header as strides sample. |

### Format differences that affect parsing

1. **Multi-line header row (`activity_running_generic.csv`)**  
   Quoted header fields contain **literal newlines** (e.g. `"Distance\nmi"`). A compliant CSV parser
   must treat the entire first logical row as one record; naive line-splitting will break. Data rows
   use normal `\r\n` line endings.

2. **Garmin workout exports (e.g. coach plan, strides outside coach)**  
   Single-line header. Columns include `Interval`, `Step Type`, `Lap`. Rows include **Warm Up**,
   **Run**, **Recovery**, **Cool Down**, and often a **Summary** row (and sometimes odd trailing
   laps). The spec’s ingest rules (drop warm-up, cool-down and after, drop recovery for strides,
   renumber laps) apply when **row roles are stated in the file** (e.g. via `Step Type`). When they
   are not, see **Exports without explicit lap roles** in `specs_v1.md`.

3. **Summary and non-lap rows**  
   `activity_running_generic.csv` ends with a row whose first column is `Summary`, not a lap index.
   Garmin workout files end with `Step Type` = `Summary`. These must **not** be treated as laps in
   normalized output.

4. **Grouped or compound lap labels**  
   In `activity_garmin_running_coach_plan_activity.csv`, `Lap` can be a range (e.g. `2 - 4`) for a
   single rolled-up interval. Treat those as **aggregate rows**, not per-lap data: **drop** them
   during ingest so the checked **examples/** files **succeed**. Only rows with a single-integer lap
   index are retained, then renumbered—see **Rolled-up or compound lap rows** in `specs_v1.md`.

5. **Column set varies between Garmin workout files**  
   In our samples, strides-outside-coach **omits** `Avg Vertical Ratio` present in the coach plan
   file. `specs_v1.md` lists `avg_vertical_ratio_pct` and `avg_gap_min_per_mi` on both
   `all_splits.csv` and `all_strides.csv`—same per-lap metric order as runs, except strides omit
   `workout_type` only. **Partial or missing raw columns** in the spec: use empty values or one
   documented placeholder consistently (including validation in Phase 7).

## Mapping raw columns → normalized splits

### Output column names

Normalized files use the **exact** header names and column order from `specs_v1.md` (e.g.
`distance_mi`, `avg_pace_min_per_mi`, `avg_gap_min_per_mi`, …). Raw Garmin headers differ: they
split units across lines (`Distance` + `mi`), use `"--"` for missing values, or use `Distance`
without the same spelling as the canonical header.

Implement a **single mapping layer** in ingest (or `io.py`): normalize header names to logical
keys, then project into the canonical schema. Suffixes such as `_mi`, `_bpm`, and `_pct` in the spec
column names are **naming conventions for the CSV header**, not a requirement that the raw file use
the same spelling.

### Is inference easy or hard?

- **Column semantics (pace, distance, HR, cadence, GCT, power, etc.)**  
  **Mostly easy:** Garmin workout exports in `examples/` share one header pattern (aside from the
  optional `Avg Vertical Ratio` column). Hard parts are header normalization (newlines, `®` in
  `Normalized Power®`), optional columns, and `"--"` / `No Weight` style sentinels. The manual
  activity export uses a different header shape.

- **Which laps to keep (warm-up, cool-down, recovery)**  
  - **Garmin workout exports with explicit step labels (`Step Type`, etc.):** filter rows using
    those labels, drop `Summary` and rolled-up lap rows, then apply the spec’s rules. Stray laps
    need explicit tests.
  - **Manual lap exports without step labels:** the spec defines behavior under **Exports without
    explicit lap roles**—no heuristic inference from pace, distance, duration, or order.

Overall: column mapping is **straightforward** for these Garmin workout examples. Without explicit
labels, follow the spec’s no-inference rule; do not “detect” lap trims from the CSV alone.

## Implementation discipline

- **Library-first:** required by **Architecture (library-first)** in `specs_v1.md`; use the package
  layout and modules above—CLI dispatches only.
- **Tests:** 100% line coverage and the checklist in `specs_v1.md` (help, ingest, merge, validate,
  `--dry-run`, `--force`, conflicts, etc.).
- **Python style:** follow repository rules (PEP 8, 90-char lines, import blocks, two blank lines
  between top-level definitions).
- **Markdown in `planning/`:** wrap prose to ~100 characters where practical (project convention).

## Suggested first steps after Phase 0

1. Read `specs_v1.md` sections on CSV formats, merge rules, and CLI.
2. Load each `examples/*.csv` with a proper CSV reader; print resolved header list and row counts.
3. Implement Phase 1 schema + IO helpers; round-trip empty and small synthetic CSVs.
4. Implement ingest against **Garmin workout** examples first (explicit `Step Type`). Define how
   **manual** exports interact with `specs_v1.md` trim rules without inferring warm-up or cool-down.

When in doubt, **`specs_v1.md` wins**.
