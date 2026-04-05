# Implementation instructions (v1)

This document guides implementers. The **authoritative functional specification** is
[`specs_v1.md`](specs_v1.md). Work proceeds in **phases** defined in
[`phases_v1.md`](phases_v1.md). If anything here disagrees with the spec, follow `specs_v1.md`.

## What you are building

A Python library under `src/lappy/` with a thin CLI (`lappy.py`) that:

1. Ingests Garmin Connect **split / lap CSV** exports into normalized per-workout CSVs.
2. Merges those into aggregate `all_splits.csv` and `all_strides.csv` with idempotent merge
   semantics.
3. Validates outputs.
4. Plotting and analysis are **out of scope** for v1 (see spec).

Canonical column orders, CLI command tree, merge identity `(workout_id, laps)`, and test checklist
are all defined in `specs_v1.md`—do not duplicate them here; implement against that document.

## Phase roadmap (do in order)

Follow [`phases_v1.md`](phases_v1.md) sequentially. Summary:

| Phase | Focus |
| ----- | ----- |
| 0 | Layout, packaging, runnable CLI, full command tree + help (handlers may stub). |
| 1 | `schema.py`, `ids.py`, `io.py`: columns, `workout_id`, paths, dates, batch discovery. |
| 2 | `ingest.py`, `ingest run`, `ingest strides` (lap rules, normalization). |
| 3 | Batch ingest (`runs-batch`, `strides-batch`). |
| 4 | `merge.py`, single-file merge. |
| 5 | Batch merge. |
| 6 | `import` = ingest then merge. |
| 7 | `validate` subcommands. |
| 8 | Coverage and hardening (100% line coverage, real-export edge cases). |

Dependency order is in `phases_v1.md`; do not skip prerequisites (e.g. merge needs schema and
ingest behavior).

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
   renumber laps) apply only when **row roles are stated in the file**—e.g. via `Step Type` (or an
   equivalent explicit label on the lap row). **Do not infer** warm-up or cool-down from pace,
   distance, duration, or lap order alone.

3. **Summary and non-lap rows**  
   `activity_running_generic.csv` ends with a row whose first column is `Summary`, not a lap index.
   Garmin workout files end with `Step Type` = `Summary`. These must **not** be treated as laps in
   normalized output.

4. **Grouped or compound lap labels**  
   In `activity_garmin_running_coach_plan_activity.csv`, `Lap` can be a range (e.g. `2 - 4`) for a
   single rolled-up interval. Decide how that maps to the spec’s per-lap normalized rows (split,
   expand, or reject) and document or test the chosen behavior.

5. **Column set varies between Garmin workout files**  
   In our samples, strides-outside-coach **omits** `Avg Vertical Ratio` present in the coach plan
   file. Normalized schemas in `specs_v1.md` still list `avg_vertical_ratio_pct` for
   `all_splits.csv`; missing raw fields should become empty values or a documented placeholder,
   consistent with validation rules you implement in Phase 7.

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
    those labels, drop `Summary`, then apply the spec’s rules. Edge cases (compound laps, stray
    laps) need explicit tests.
  - **Manual lap exports with no warm-up/cool-down on any row:** **do not infer** warm-up or
    cool-down. Lap length, order, or a trailing `Summary` row are **not** sufficient to classify
    laps. Any trim that matches `specs_v1.md` must come from **explicit user input** (e.g. flags
    naming which laps to drop) or a **spec/product decision** documented in code and tests—not
    from heuristics.

Overall: column mapping is **straightforward** for these Garmin workout examples. Without explicit
labels, **do not “detect” lap trims from the CSV**; treat trimming as a product and CLI design
problem, not an inference problem.

## Implementation discipline

- **Library-first:** business logic lives in `src/lappy/` modules per `specs_v1.md`; CLI dispatches
  only.
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

When in doubt, **`specs_v1.md` wins**; use `phases_v1.md` for sequencing and exit criteria per
phase.
