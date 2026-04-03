# Specification

This file defines the **v1** specification for the running project data-ingestion suite.

The suite is responsible for:

- ingesting Garmin split CSV exports into useful normalized CSV files
- organizing and collating those splits for later analysis
- validating generated CSV files
- exposing the above through a single top-level CLI called `lappy.py`

Plotting, automated analysis, and coaching are explicitly out of scope for this v1 specification.

---

## Scope

The v1 scope of this specification is limited to:

- ingesting single Garmin activity CSV files
- ingesting Garmin activity CSV files in batch mode
- producing normalized per-workout CSV files
- merging normalized per-workout CSV files into historical aggregate CSV files
- validating generated CSV files
- implementing the core logic as a Python library
- exposing the library through a thin CLI wrapper called `lappy.py`

The implementation should be **library-first**:

- the reusable data-processing logic belongs in an importable Python package
- the CLI should be a thin wrapper over that library
- the CLI should not contain the core business logic itself

This is intentional so that the same core code can later be reused from notebooks and other Python callers.

---

## Repository layout

The canonical repository layout is:

```text
repo/
├── lappy.py
├── src/
│   └── lappy/
│       ├── __init__.py
│       ├── cli.py
│       ├── schema.py
│       ├── ids.py
│       ├── io.py
│       ├── ingest.py
│       ├── merge.py
│       └── validate.py
├── tests/
└── data/
    └── [person]/
        ├── raw/
        ├── runs/
        └── strides/
```

Notes:

- `lappy.py` is the required top-level CLI entrypoint.
- `src/lappy/` contains the importable library implementation.
- `src/lappy/cli.py` contains CLI wiring and argument parsing.
- `src/lappy/schema.py` contains canonical column definitions and schema helpers.
- `src/lappy/ids.py` contains `RunDate` / `WorkoutID` parsing and inference helpers.
- `src/lappy/io.py` contains CSV loading and writing helpers.
- `src/lappy/ingest.py` contains ingest and normalization logic.
- `src/lappy/merge.py` contains merge and deduplication logic.
- `src/lappy/validate.py` contains schema and semantic validation logic.
- `tests/` contains the automated test suite.
- `data/[person]/raw/` contains renamed raw Garmin split exports.
- `data/[person]/runs/` contains normalized non-strides workout files and `all_splits.csv`.
- `data/[person]/strides/` contains normalized strides workout files and `all_strides.csv`.
- `[person]` may be omitted if the repository is only ever used for one runner, but the CLI should support the multi-runner layout above as canonical.

---

## Library design constraints

The v1 library must be designed so that the CLI can remain thin.

### Required separation of concerns

- `lappy.py` should delegate to library functions instead of implementing the data-processing logic inline.
- Ingest, merge, validation, schema, and ID inference should be implemented as normal Python functions in `src/lappy/`.
- The library code should be usable directly from Python without shelling out to the CLI.
- Tests should primarily target the library functions, with additional tests for CLI argument parsing and exit behavior.

### Non-goals for the library in v1

The library does **not** need to include:

- plotting helpers
- automated analysis helpers
- coaching logic
- database storage

---

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

---

## Canonical naming rules

- The canonical non-strides interval workout type is `intervals`.
- `interval` should be treated as invalid in output files.
- If the CLI accepts aliases, it may accept `interval` on input, but it must normalize it to `intervals` before writing output.
- `strides` is not a valid `WorkoutType` value for `all_splits.csv`; strides belong in `all_strides.csv`.

---

## File naming rules

### Raw files

Raw Garmin split exports stored in the repository must be renamed to one of these forms:

- `raw_splits_YYYYMMDD.csv`
- `raw_splits_YYYYMMDD_N.csv`

Where:

- `YYYYMMDD` is the workout date.
- `_N` is a 1-based same-day workout index and is required whenever more than one workout of the same family exists on the same date.
- If there is only one workout for that date and family, the unsuffixed form should be used.

Examples:

- `raw_splits_20260403.csv`
- `raw_splits_20260403_2.csv`

### Normalized non-strides files

Normalized non-strides files must be named:

- `running_splits_YYYYMMDD.csv`
- `running_splits_YYYYMMDD_N.csv`

Examples:

- `running_splits_20260403.csv`
- `running_splits_20260403_2.csv`

### Normalized strides files

Normalized strides files must be named:

- `strides_YYYYMMDD.csv`
- `strides_YYYYMMDD_N.csv`

Examples:

- `strides_20260403.csv`
- `strides_20260403_2.csv`

### Aggregate files

The aggregate files must be stored at:

- `data/[person]/runs/all_splits.csv`
- `data/[person]/strides/all_strides.csv`

---

## Workout identifier

A stable workout identifier is required to support:

- multiple workouts of the same family on the same date
- idempotent merge behavior
- conflict detection during merge
- traceable linkage between normalized files and aggregate rows

The canonical column name is `WorkoutID`.

### WorkoutID format

`WorkoutID` must be derived from the date and optional same-day index using these forms:

- `YYYYMMDD`
- `YYYYMMDD_N`

Examples:

- `20260403`
- `20260403_2`

Rules:

- `WorkoutID` must match the date and optional suffix encoded in the normalized output filename.
- `WorkoutID` must be written to every row of every normalized file.
- `WorkoutID` must be preserved when normalized files are merged into aggregate files.
- For a single workout on a date, the canonical `WorkoutID` has no `_1` suffix.
- For multiple same-day workouts of the same family, `_N` must be used consistently in the raw filename, normalized filename, and `WorkoutID`.

---

## Date and WorkoutID inference rules

The CLI and library should infer `RunDate` and `WorkoutID` from the input filename whenever possible.

### Recognized filename suffixes

A filename is considered inferable if its basename ends in one of these forms immediately before `.csv`:

- `YYYYMMDD`
- `YYYYMMDD_N`

Examples of inferable filenames:

- `raw_splits_20260403.csv`
- `raw_splits_20260403_2.csv`
- `activity_export_20260403.csv`
- `activity_export_20260403_2.csv`

Inference rules:

- `RunDate` is the `YYYYMMDD` portion.
- `WorkoutID` is `YYYYMMDD` or `YYYYMMDD_N`, matching the suffix.
- If the filename does not contain an inferable suffix, `--date` is required in single-file mode.
- In batch mode, each input file must either have an inferable suffix or the batch command must fail for that file.
- When `--date` is explicitly provided, it overrides any inferred date.
- When `--date` is explicitly provided and the input filename also carries an `_N` suffix, the output `WorkoutID` should still include that same `_N` suffix.
- If `--date` and the filename encode conflicting dates, the command must fail unless the spec is later extended with an explicit override flag.

---

## CSV format

### Running splits files

`running_splits_YYYYMMDD.csv` and `running_splits_YYYYMMDD_N.csv` must contain these columns in this exact order:

1. `RunDate`
2. `WorkoutID`
3. `Laps`
4. `Distancemi`
5. `Avg Pacemin/mi`
6. `Avg GAPmin/mi`
7. `Avg HRbpm`
8. `Max HRbpm`
9. `Avg Run Cadencespm`
10. `Avg Ground Contact Timems`
11. `Avg GCT Balance%`
12. `Avg Stride Lengthm`
13. `Avg Vertical Oscillationcm`
14. `Avg Vertical Ratio%`
15. `Avg PowerW`
16. `WorkoutType`
17. `Notes`

Allowed `WorkoutType` values for running splits files are:

- `easy`
- `tired`
- `long`
- `tempo`
- `intervals`
- `race`

### All Splits

`all_splits.csv` must contain the same columns, in the same order, as running splits files.

### Strides files

`strides_YYYYMMDD.csv` and `strides_YYYYMMDD_N.csv` must contain these columns in this exact order:

1. `RunDate`
2. `WorkoutID`
3. `Laps`
4. `Distancemi`
5. `Avg Pacemin/mi`
6. `Avg HRbpm`
7. `Max HRbpm`
8. `Avg Run Cadencespm`
9. `Avg Ground Contact Timems`
10. `Avg GCT Balance%`
11. `Avg Stride Lengthm`
12. `Avg Vertical Oscillationcm`
13. `Avg PowerW`
14. `Notes`

### All Strides

`all_strides.csv` must contain the same columns, in the same order, as strides files.

---

## Ingestion flow

## Single activity at a time

1. Download the splits CSV from Garmin Connect.
2. Store it in `data/[person]/raw/` using one of these canonical names:
   - `raw_splits_YYYYMMDD.csv`
   - `raw_splits_YYYYMMDD_N.csv`
3. Ingest the raw file into either a normalized running splits file or a normalized strides file.

### Non-strides run ingest

If this is a non-strides run, create a new file called one of:

- `running_splits_YYYYMMDD.csv`
- `running_splits_YYYYMMDD_N.csv`

It must contain the relevant laps from the raw activity CSV with the following processing:

1. Remove the warm-up laps.
2. Remove the cool-down lap and anything after cool-down.
3. Rename the remaining laps sequentially (`1`, `2`, `3`, ...).
4. Add `RunDate`.
5. Add `WorkoutID`.
6. Add `WorkoutType`.
7. Add `Notes`, if supplied.

The normalized file must be written under `data/[person]/runs/`.

### Strides ingest

If this is a strides workout, create a new file called one of:

- `strides_YYYYMMDD.csv`
- `strides_YYYYMMDD_N.csv`

It must contain the relevant laps from the raw activity CSV with the following processing:

1. Remove warm-up laps.
2. Remove the cool-down lap and anything after cool-down.
3. Remove recovery laps.
4. Keep only the strides laps.
5. Rename the remaining laps sequentially (`1`, `2`, `3`, ...).
6. Add `RunDate`.
7. Add `WorkoutID`.
8. Add `Notes`, if supplied.

The normalized file must be written under `data/[person]/strides/`.

### Single-file merge

If this is a non-strides run, merge the normalized running splits file into `all_splits.csv` with these rules:

1. Append new laps.
2. Make sure there are no duplicates.
3. Make sure the file is chronological using `RunDate`, `WorkoutID`, and `Laps`.
4. Do not modify already present laps.

If this is a strides workout, merge the normalized strides file into `all_strides.csv` with these rules:

1. Append new laps.
2. Make sure there are no duplicates.
3. Make sure the file is chronological using `RunDate`, `WorkoutID`, and `Laps`.
4. Do not modify already present laps.

---

## Batch mode

Batch modes should exist wherever they naturally fit the workflow.

### Batch ingest

Batch ingest must support processing all raw files under the canonical raw directory:

- `data/[person]/raw/raw_splits_YYYYMMDD.csv`
- `data/[person]/raw/raw_splits_YYYYMMDD_N.csv`

For each matched raw file:

- infer `RunDate` from the filename
- infer `WorkoutID` from the filename
- create the corresponding normalized file in `runs/` or `strides/`

### Batch merge

Batch merge must support merging all normalized files of a family into the family aggregate file:

- all `running_splits_*.csv` into `all_splits.csv`
- all `strides_*.csv` into `all_strides.csv`

Batch merge requirements:

1. Batch merge must be idempotent.
2. Already present laps must not change.
3. Existing rows must never be silently overwritten.

---

## Merge identity and conflict rules

To satisfy idempotent merge and the requirement that already present laps should not change, merge identity is defined as follows:

- for `all_splits.csv`: `(WorkoutID, Laps)`
- for `all_strides.csv`: `(WorkoutID, Laps)`

Merge behavior must follow these rules:

1. If an incoming row has the same identity and the same data as an existing row, it is a duplicate and must be skipped.
2. If an incoming row has the same identity but different data from an existing row, this is a conflict and the merge must fail.
3. Existing rows must never be silently overwritten.
4. Output ordering must be ascending by `RunDate`, then `WorkoutID`, then `Laps`.

---

## Validation rules

Validation commands must check at least the following:

- required columns exist
- columns are in canonical order
- `RunDate` is valid and normalized
- `WorkoutID` is valid and consistent with the date and optional suffix rules
- `Laps` are positive integers
- `WorkoutType` is valid where applicable
- aggregate files are sorted by `RunDate`, `WorkoutID`, and `Laps`
- aggregate files contain no conflicting duplicate identities

---

## Python implementation design

There must be one top-level Python script called `lappy.py`.

The problem should be decomposed into subcommands.

The CLI must include help.

The codebase must include tests under `tests/` and achieve 100% line coverage.

### CLI and library relationship

- `lappy.py` is the top-level executable interface.
- `src/lappy/cli.py` should contain the CLI parser and command dispatch.
- `lappy.py` may be a very small wrapper that imports and invokes the CLI entrypoint from the package.
- The underlying functionality must live in importable library modules under `src/lappy/`.
- The CLI should call those library functions rather than duplicating their logic.

### Suggested module responsibilities

The following module split is recommended for v1:

- `schema.py`: canonical column orders, allowed values, schema helpers
- `ids.py`: `RunDate` / `WorkoutID` parsing, formatting, inference, consistency checks
- `io.py`: CSV loading and writing helpers
- `ingest.py`: raw-to-normalized transformation logic
- `merge.py`: aggregate merge logic and conflict detection
- `validate.py`: file and row validation logic
- `cli.py`: argument parsing, command dispatch, output messages, exit codes

This module split is recommended, but minor naming variations are acceptable if the same separation of concerns is preserved.

---

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

---

## Command responsibilities

### `ingest`

`ingest` creates normalized per-workout CSV files from raw split CSV files.

#### `lappy.py ingest run`

Creates one normalized non-strides file from one raw split CSV file.

Required arguments:

- `--input PATH`
- `--type easy|tired|long|tempo|intervals|race`
- `--date YYYYMMDD|YYYY-MM-DD` only when the input filename does not already encode an inferable date

Optional arguments:

- `--workout-id ID`
- `--notes TEXT`
- `--output PATH`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- remove warm-up laps
- remove cool-down and anything after it
- rename kept laps to `1..N`
- add `RunDate`
- add `WorkoutID`
- add `WorkoutType`
- add `Notes`
- write normalized output in the canonical running splits column order

Default output name:

- `running_splits_YYYYMMDD.csv`
- `running_splits_YYYYMMDD_N.csv` when `WorkoutID` has a same-day suffix

#### `lappy.py ingest strides`

Creates one normalized strides file from one raw split CSV file.

Required arguments:

- `--input PATH`
- `--date YYYYMMDD|YYYY-MM-DD` only when the input filename does not already encode an inferable date

Optional arguments:

- `--workout-id ID`
- `--notes TEXT`
- `--output PATH`
- `--output-dir DIR`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- remove warm-up laps
- remove cool-down and anything after it
- remove recovery laps
- keep only strides laps
- rename kept laps to `1..N`
- add `RunDate`
- add `WorkoutID`
- add `Notes`
- write normalized output in the canonical strides column order

Default output name:

- `strides_YYYYMMDD.csv`
- `strides_YYYYMMDD_N.csv` when `WorkoutID` has a same-day suffix

#### `lappy.py ingest runs-batch`

Batch-ingests raw split files into normalized running splits files.

Required arguments:

- `--input-dir DIR`
- `--type easy|tired|long|tempo|intervals|race`

Optional arguments:

- `--glob PATTERN`
- `--output-dir DIR`
- `--notes TEXT`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- find raw files matching the canonical raw filename pattern
- infer `RunDate` and `WorkoutID` from each filename
- ingest each file as a non-strides run of the given type
- write one normalized running file per input file
- fail for any file whose date cannot be inferred

#### `lappy.py ingest strides-batch`

Batch-ingests raw split files into normalized strides files.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--output-dir DIR`
- `--notes TEXT`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- find raw files matching the canonical raw filename pattern
- infer `RunDate` and `WorkoutID` from each filename
- ingest each file as a strides workout
- write one normalized strides file per input file
- fail for any file whose date cannot be inferred

### `merge`

`merge` merges normalized per-workout files into the family aggregate file.

#### `lappy.py merge run`

Merges one normalized running splits file into `all_splits.csv`.

Required arguments:

- `--input PATH`

Optional arguments:

- `--output PATH`
- `--force`
- `--dry-run`
- `--quiet`

Behavior:

- load the input normalized run file
- load existing aggregate output if present
- merge using identity `(WorkoutID, Laps)`
- skip exact duplicates
- fail on conflicting duplicate identities
- write sorted aggregate output

Default output:

- `data/[person]/runs/all_splits.csv` when the canonical repository layout is used

#### `lappy.py merge strides`

Merges one normalized strides file into `all_strides.csv`.

Required arguments:

- `--input PATH`

Optional arguments:

- `--output PATH`
- `--force`
- `--dry-run`
- `--quiet`

Behavior mirrors `merge run`, but targets the strides aggregate.

Default output:

- `data/[person]/strides/all_strides.csv` when the canonical repository layout is used

#### `lappy.py merge runs-batch`

Batch-merges normalized running splits files into `all_splits.csv`.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--output PATH`
- `--dry-run`
- `--quiet`

Behavior:

- find normalized running files under the input directory
- merge all of them into the output aggregate file
- preserve idempotence
- fail on conflicts
- never overwrite differing existing rows silently

#### `lappy.py merge strides-batch`

Batch-merges normalized strides files into `all_strides.csv`.

Required arguments:

- `--input-dir DIR`

Optional arguments:

- `--glob PATTERN`
- `--output PATH`
- `--dry-run`
- `--quiet`

Behavior mirrors `merge runs-batch`, but targets the strides aggregate.

### `import`

`import` is a convenience command that performs ingest followed by merge.

#### `lappy.py import run`

Equivalent to:

1. `lappy.py ingest run`
2. `lappy.py merge run`

#### `lappy.py import strides`

Equivalent to:

1. `lappy.py ingest strides`
2. `lappy.py merge strides`

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

Validates that a normalized non-strides file:

- has the exact running-splits schema
- uses valid `WorkoutType` values
- uses a valid `WorkoutID`
- uses positive integer lap numbers

#### `lappy.py validate strides-splits`

Validates that a normalized strides file:

- has the exact strides schema
- uses a valid `WorkoutID`
- uses positive integer lap numbers

#### `lappy.py validate all-splits`

Validates that `all_splits.csv`:

- has the exact canonical schema
- uses only allowed `WorkoutType` values
- is ordered chronologically by `RunDate`, then `WorkoutID`, then `Laps`
- contains no duplicate identities
- contains no merge conflicts

#### `lappy.py validate all-strides`

Validates that `all_strides.csv`:

- has the exact canonical schema
- is ordered chronologically by `RunDate`, then `WorkoutID`, then `Laps`
- contains no duplicate identities
- contains no merge conflicts

---

## Help requirements

The CLI must provide help at every level:

- `lappy.py -h`
- `lappy.py ingest -h`
- `lappy.py ingest run -h`
- `lappy.py merge -h`
- `lappy.py import -h`
- `lappy.py validate -h`

---

## Exit behavior

- exit code `0` on success
- non-zero exit code on validation error, parse error, file error, or merge conflict

---

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
- removal of warm-up laps
- removal of cool-down and everything after it
- removal of recovery laps for strides
- lap renumbering
- date inference from filename
- `WorkoutID` inference from filename
- date insertion when `--date` is explicitly provided
- notes insertion
- `WorkoutType` normalization to `intervals`
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
- validation failures for malformed date, workout ID, or lap values
- `--dry-run`
- `--force`
- non-zero exit codes on errors
- direct library-function tests for ingest, merge, validate, and ID inference logic

---

## Non-goals for v1

The following are explicitly out of scope for this version of the spec:

- plotting command design
- automated analysis or coaching commands
- database storage
- notebook-specific helper APIs beyond what naturally follows from the reusable library design
