# Specification

This file defines the v1 specification for the running project library and its thin CLI.
The project is responsible for:

- ingesting Garmin split CSV exports into useful normalized CSV files
- organizing and collating those splits for later analysis
- supporting recurring plotting and analysis workflows

## Scope

The v1 scope of this specification is limited to:

- ingesting single Garmin activity CSV files
- ingesting Garmin activity CSV files in batch mode
- producing normalized per-workout CSV files
- merging normalized per-workout CSV files into historical aggregate CSV files
- validating generated CSV files
- exposing all of the above through a single top-level CLI called `lappy.py`

Plotting and analysis commands may be added later, but they are not required by this v1 spec.

## Run types

Canonical run types are:

- Easy run (`easy`)
- Tired run (`tired`)
- Long run (`long`)
- Tempo run (`tempo`)
- Interval run (`intervals`)
- Race (`race`)
- Strides (`strides`)

`strides` is handled separately from the other run types because strides are stored in
`all_strides.csv`, not in `all_splits.csv`.

For all runs except strides, the historical and chronological splits are recorded in
`all_splits.csv`.

For stride workouts, the historical and chronological splits are recorded in
`all_strides.csv`.

## Canonical naming rules

- The canonical non-strides interval workout type is `intervals`.
- `interval` should be treated as invalid in output files.
- If the CLI accepts aliases, it may accept `interval` on input, but it must normalize it to
  `intervals` before writing output.
- `strides` is not a valid `workout_type` value for `all_splits.csv`; strides belong in
  `all_strides.csv`.

## Repository layout

The repository should use a library-first layout for v1:

```text
repo/
├── src/
│   └── lappy/
│       ├── __init__.py
│       ├── cli.py
│       ├── io.py
│       ├── ingest.py
│       ├── merge.py
│       ├── validate.py
│       ├── schema.py
│       └── ids.py
├── tests/
└── data/
    └── [goal]/
        ├── raw/
        │   ├── raw_splits_[YYYYMMDD].csv
        │   └── raw_splits_[YYYYMMDD]_[N].csv
        ├── runs/
        │   ├── running_splits_[YYYYMMDD].csv
        │   ├── running_splits_[YYYYMMDD]_[N].csv
        │   └── all_splits.csv
        └── strides/
            ├── strides_[YYYYMMDD].csv
            ├── strides_[YYYYMMDD]_[N].csv
            └── all_strides.csv
```

`[goal]` is a placeholder for the training target or grouping key. It may represent a person, a
race, a season, or another training goal.

`tests/` must remain at the repository root, not inside `src/lappy/`.

## CSV format

### All Splits

`all_splits.csv` must contain these columns in this exact order:

1. `run_date`
2. `workout_id`
3. `workout_type`
4. `laps`
5. `distance_mi`
6. `avg_pace_min_per_mi`
7. `avg_gap_min_per_mi`
8. `avg_hr_bpm`
9. `max_hr_bpm`
10. `avg_run_cadence_spm`
11. `avg_ground_contact_time_ms`
12. `avg_gct_balance_pct`
13. `avg_stride_length_m`
14. `avg_vertical_oscillation_cm`
15. `avg_vertical_ratio_pct`
16. `avg_power_w`
17. `notes`

Allowed `workout_type` values for `all_splits.csv` are:

- `easy`
- `tired`
- `long`
- `tempo`
- `intervals`
- `race`

### All Strides

`all_strides.csv` must contain these columns in this exact order:

1. `run_date`
2. `workout_id`
3. `laps`
4. `distance_mi`
5. `avg_pace_min_per_mi`
6. `avg_hr_bpm`
7. `max_hr_bpm`
8. `avg_run_cadence_spm`
9. `avg_ground_contact_time_ms`
10. `avg_gct_balance_pct`
11. `avg_stride_length_m`
12. `avg_vertical_oscillation_cm`
13. `avg_power_w`
14. `notes`

## Ingestion flow

Some activity CSVs include a trailing **Summary** row (for example `Step Type` = `Summary`, or a
lap column value of `Summary`); others do not. When absent, this is fine—ingest must not require a
Summary row. When present, those rows are workout-level aggregates, not laps. Remove them during
ingest. They must not appear in normalized per-workout files or in aggregate files.

## Single activity at a time

1. Download the splits CSV from Garmin Connect into `~/Downloads`.
   The file name is expected to be of the form `activity_xxxxxxxxxxx.csv`.

   Example: `examples/activity_22390785030.csv`

2. If this is a non-strides run, create a new file called `running_splits_YYYYMMDD.csv`
   or `running_splits_YYYYMMDD_N.csv` containing the relevant laps from the activity CSV with the
   following processing:

   1. Remove any summary rows and any other non-lap aggregate rows (no-op if none are present).
   2. Remove the warm-up laps.
   3. Remove the cool-down lap and anything after cool-down.
   4. Rename the remaining laps sequentially (`1`, `2`, `3`, ...).
   5. Add run date.
   6. Add workout identifier.
   7. Add workout type.
   8. Add notes, if supplied.

3. If this is a strides workout, create a new file called `strides_YYYYMMDD.csv`
   or `strides_YYYYMMDD_N.csv` containing the relevant laps from the activity CSV with the
   following processing:

   1. Remove any summary rows and any other non-lap aggregate rows (no-op if none are present).
   2. Remove warm-up laps.
   3. Remove the cool-down lap and anything after cool-down.
   4. Remove recovery laps.
   5. Keep only the strides laps.
   6. Rename the remaining laps sequentially (`1`, `2`, `3`, ...).
   7. Add run date.
   8. Add workout identifier.
   9. Add notes, if supplied.

4. If this is a non-strides run, merge `running_splits_YYYYMMDD.csv` into `all_splits.csv`
   with the following processing:

   1. Append the running splits.
   2. Make sure there are no duplicates.
   3. Make sure the file is chronological using `run_date`, `workout_id`, and `laps`.
   4. Merge operation is idempotent: do not modify already present laps.

5. If this is a strides workout, merge `strides_YYYYMMDD.csv` into `all_strides.csv`
   with the following processing:

   1. Append the strides splits.
   2. Make sure there are no duplicates.
   3. Make sure the file is chronological using `run_date`, `workout_id`, and `laps`.
   4. Do not modify already present laps.

## Batch mode

Batch modes should exist wherever they naturally fit the workflow.

1. Batch ingest from activity files. These files are expected to have a date in their file name
   at the end before the extension. That date should be used to create the corresponding
   `running_splits_YYYYMMDD.csv` or `strides_YYYYMMDD.csv` file.

2. Batch merge all `running_splits_YYYYMMDD.csv` or `strides_YYYYMMDD.csv` files into
   `all_splits.csv` or `all_strides.csv` respectively.

3. Batch merge must be idempotent. Already present laps must not change.

## Merge identity and conflict rules

To satisfy idempotent merge and the requirement that already present laps should not change,
merge identity is defined as follows:

- for `all_splits.csv`: `(workout_id, laps)`
- for `all_strides.csv`: `(workout_id, laps)`

Merge behavior must follow these rules:

1. If an incoming row has the same identity and the same data as an existing row, it is a duplicate
   and must be skipped.
2. If an incoming row has the same identity but different data from an existing row, this is a
   conflict and the merge must fail.
3. Existing rows must never be silently overwritten.

## Workout identifier rules

`workout_id` is the stable workout key used for merge identity.

- It must be present in every normalized per-workout file and in every aggregate file.
- It must be unique within a workout family for a given goal folder.
- It should be derived from the normalized output filename stem, for example
  `running_splits_YYYYMMDD`, `running_splits_YYYYMMDD_N`, `strides_YYYYMMDD`, or
  `strides_YYYYMMDD_N`.
- Multiple same-family workouts on the same date must use distinct `workout_id` values.
- It can be implemented as [YYYYMMDD] or [YYYYMMDD]_[N].


## Implementation layout

V1 should be implemented as a Python library with a thin CLI on top.

Recommended package layout:

```text
src/lappy/
├── __init__.py
├── cli.py
├── io.py
├── ingest.py
├── merge.py
├── validate.py
├── schema.py
└── ids.py

tests/
```

Module responsibilities should be organized as follows:

- `cli.py`: argument parsing and command dispatch
- `io.py`: CSV reading and writing, file discovery, filename parsing, and date inference
- `ingest.py`: trimming and normalization of Garmin activity CSVs into per-workout CSVs
- `merge.py`: idempotent merging of normalized per-workout CSVs into aggregate CSVs
- `validate.py`: schema validation and semantic validation
- `schema.py`: canonical column orders and field definitions
- `ids.py`: `workout_id` generation and related helpers

The CLI entrypoint remains `lappy.py`, but command handlers should call importable library code
in `src/lappy/` rather than embedding business logic directly in the CLI layer.

# Python package and CLI design

There must be one top-level CLI called `lappy.py`.

The problem should be decomposed into subcommands.

The CLI must include help.

The codebase must include tests and achieve 100% line coverage.

## CLI structure

The required command structure is:

```text
lappy.py
├── ingest
│   ├── run
│   ├── strides
│   ├── runs-batch
│   └── strides-batch
├── merge
│   ├── run
│   ├── strides
│   ├── runs-batch
│   └── strides-batch
├── import
│   ├── run
│   ├── strides
│   ├── runs-batch
│   └── strides-batch
└── validate
    ├── running-splits
    ├── strides-splits
    ├── all-splits
    └── all-strides
```

## Command responsibilities

### `ingest`

`ingest` creates normalized per-workout CSV files from Garmin activity CSV files.

#### `lappy.py ingest run`

Creates `running_splits_YYYYMMDD.csv` from one non-strides Garmin activity CSV.

Required arguments:

- `--input PATH`
- `--date YYYYMMDD|YYYY-MM-DD`
- `--type easy|tired|long|tempo|intervals|race`

Optional arguments:

- `--notes TEXT`
- `--output PATH`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- remove any summary rows when present (non-lap aggregate rows; e.g. `Step Type` or lap column =
  `Summary`)
- remove warm-up laps
- remove cool-down and anything after it
- rename kept laps to `1..N`
- add `run_date`
- add `workout_id`
- add `workout_type`
- add `notes`
- write normalized output in the canonical `all_splits.csv` column order

Default output name:

- `running_splits_YYYYMMDD.csv`

#### `lappy.py ingest strides`

Creates `strides_YYYYMMDD.csv` from one strides Garmin activity CSV.

Required arguments:

- `--input PATH`
- `--date YYYYMMDD|YYYY-MM-DD`

Optional arguments:

- `--notes TEXT`
- `--output PATH`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- remove any summary rows when present (non-lap aggregate rows; e.g. `Step Type` or lap column =
  `Summary`)
- remove warm-up laps
- remove cool-down and anything after it
- remove recovery laps
- keep only strides laps
- rename kept laps to `1..N`
- add `run_date`
- add `workout_id`
- add `notes`
- write normalized output in the canonical `all_strides.csv` column order

Default output name:

- `strides_YYYYMMDD.csv`

#### `lappy.py ingest runs-batch`

Batch-ingests multiple activity CSV files into multiple `running_splits_YYYYMMDD.csv` files.

Required arguments:

- `--input-dir DIR`
- `--type easy|tired|long|tempo|intervals|race`

Optional arguments:

- `--glob PATTERN`
- `--notes TEXT`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- discover matching activity files
- extract the date from each file name
- ingest each file as a non-strides run
- write one output file per activity
- skip existing outputs unless `--force` is specified

#### `lappy.py ingest strides-batch`

Batch-ingests multiple activity CSV files into multiple `strides_YYYYMMDD.csv` files.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--notes TEXT`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- discover matching activity files
- extract the date from each file name
- ingest each file as a strides workout
- write one output file per activity
- skip existing outputs unless `--force` is specified

### `merge`

`merge` merges normalized per-workout CSV files into aggregate historical CSV files.

#### `lappy.py merge run`

Merges one `running_splits_YYYYMMDD.csv` file into `all_splits.csv`.

Required arguments:

- `--input PATH`

Optional arguments:

- `--output PATH`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- load existing `all_splits.csv` if present
- load the input file
- validate rows and schema
- append non-duplicate rows
- fail on merge conflict
- sort by `run_date`, then `workout_id`, then `laps`
- write the result

Default output name:

- `all_splits.csv`

#### `lappy.py merge strides`

Merges one `strides_YYYYMMDD.csv` file into `all_strides.csv`.

Required arguments:

- `--input PATH`

Optional arguments:

- `--output PATH`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- load existing `all_strides.csv` if present
- load the input file
- validate rows and schema
- append non-duplicate rows
- fail on merge conflict
- sort by `run_date`, then `workout_id`, then `laps`
- write the result

Default output name:

- `all_strides.csv`

#### `lappy.py merge runs-batch`

Batch-merges all `running_splits_YYYYMMDD.csv` files into `all_splits.csv`.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--output PATH`
- `--dry-run`
- `--quiet`

Behavior:

- discover all matching files
- load existing aggregate output if present
- merge all normalized run files
- preserve existing laps unchanged
- skip duplicates
- fail on conflict
- sort by `run_date`, then `workout_id`, then `laps`
- write the result

#### `lappy.py merge strides-batch`

Batch-merges all `strides_YYYYMMDD.csv` files into `all_strides.csv`.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--output PATH`
- `--dry-run`
- `--quiet`

Behavior:

- discover all matching files
- load existing aggregate output if present
- merge all normalized strides files
- preserve existing laps unchanged
- skip duplicates
- fail on conflict
- sort by `run_date`, then `workout_id`, then `laps`
- write the result

### `import`

`import` is the convenience command that performs ingest followed by merge.

#### `lappy.py import run`

Equivalent to:

1. `lappy.py ingest run`
2. `lappy.py merge run`

Required arguments:

- `--input PATH`
- `--date YYYYMMDD|YYYY-MM-DD`
- `--type easy|tired|long|tempo|intervals|race`

Optional arguments:

- `--notes TEXT`
- `--output-dir DIR`
- `--merged-output PATH`
- `--force`
- `--dry-run`
- `--quiet`

#### `lappy.py import strides`

Equivalent to:

1. `lappy.py ingest strides`
2. `lappy.py merge strides`

Required arguments:

- `--input PATH`
- `--date YYYYMMDD|YYYY-MM-DD`

Optional arguments:

- `--notes TEXT`
- `--output-dir DIR`
- `--merged-output PATH`
- `--force`
- `--dry-run`
- `--quiet`

#### `lappy.py import runs-batch`

Equivalent to:

1. `lappy.py ingest runs-batch`
2. `lappy.py merge runs-batch`

#### `lappy.py import strides-batch`

Equivalent to:

1. `lappy.py ingest strides-batch`
2. `lappy.py merge strides-batch`

### `validate`

`validate` checks schema and semantic correctness of generated CSV files.

#### `lappy.py validate running-splits`

Validates that a normalized non-strides file has the exact `all_splits.csv` schema and valid
`workout_type` values.

#### `lappy.py validate strides-splits`

Validates that a normalized strides file has the exact `all_strides.csv` schema.

#### `lappy.py validate all-splits`

Validates that `all_splits.csv`:

- has the exact canonical schema
- uses only allowed `workout_type` values
- is ordered chronologically by `run_date`, then `workout_id`, then `laps`
- contains no duplicate identities
- contains no merge conflicts

#### `lappy.py validate all-strides`

Validates that `all_strides.csv`:

- has the exact canonical schema
- is ordered chronologically by `run_date`, then `workout_id`, then `laps`
- contains no duplicate identities
- contains no merge conflicts

## Help requirements

The CLI must provide help at every level:

- `lappy.py -h`
- `lappy.py ingest -h`
- `lappy.py ingest run -h`
- `lappy.py merge -h`
- `lappy.py import -h`
- `lappy.py validate -h`

## Exit behavior

- exit code `0` on success
- non-zero exit code on validation error, parse error, file error, or merge conflict

## Test requirements

The implementation must include tests and achieve 100% line coverage.

The test suite must cover at least:

- top-level help output
- nested subcommand help output
- argument parsing for every command and subcommand
- successful `ingest run`
- successful `ingest strides`
- successful `ingest runs-batch`
- successful `ingest strides-batch`
- removal of summary rows
- removal of warm-up laps
- removal of cool-down and everything after it
- removal of recovery laps for strides
- lap renumbering
- date insertion
- notes insertion
- `workout_type` normalization to `intervals`
- successful single-file merge for runs
- successful single-file merge for strides
- successful batch merge for runs
- successful batch merge for strides
- duplicate skipping
- merge idempotence
- merge conflict failure when identity matches but data differs
- chronological sorting
- validation failures for missing columns
- validation failures for invalid workout type
- validation failures for malformed date or lap values
- `--dry-run`
- `--force`
- non-zero exit codes on errors

## Non-goals for v1

The following are explicitly out of scope for this version of the spec:

- plotting command design
- automated analysis or coaching commands
- database storage
