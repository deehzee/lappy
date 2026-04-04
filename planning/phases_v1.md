# Implementation phases

This document breaks the [specification](specs.md) into implementation phases. It does not replace
the spec; when they disagree, follow `specs.md`.

## Phase 0 — Project skeleton

- Repository layout: `src/lappy/` modules, `tests/`, and `data/` as in the spec.
- Packaging, dependencies, and a runnable CLI entry (`lappy.py` or equivalent).
- CLI structure with the full command tree and help at every level required by the spec; stub
  handlers are acceptable until later phases if commands fail clearly.
- Minimal smoke tests: top-level and nested help, basic exit codes.

**Exit criteria:** `lappy.py -h` and nested `-h` work; the project installs and runs.

---

## Phase 1 — Schema, IDs, and IO

- `schema.py`: canonical column order and field definitions for normalized runs, strides, and
  aggregates.
- `ids.py`: `WorkoutID` generation and disambiguation (including same-day, same-family workouts).
- `io.py`: CSV read/write with exact column order, path and filename parsing, date handling
  (`YYYYMMDD` and `YYYY-MM-DD`), and helpers for batch discovery (globs, directories).

**Exit criteria:** Unit tests for schema constants, ID edge cases, and IO round-trips without full
ingest or merge logic.

---

## Phase 2 — Single-file ingest

- `ingest.py`: Garmin activity CSV to `running_splits_YYYYMMDD.csv` and `strides_YYYYMMDD.csv`.
- Lap handling: remove warm-up and cool-down (and after), strides recovery removal, strides-only
  laps, sequential lap renumbering, `RunDate`, `WorkoutID`, `WorkoutType` (runs), `Notes`;
  `interval` normalized to `intervals` on output.
- CLI: `ingest run`, `ingest strides` with the flags and behavior described in the spec.

**Exit criteria:** Tests covering lap trimming, renumbering, dates, notes, and workout type
normalization; representative fixtures as needed.

---

## Phase 3 — Batch ingest

- `ingest runs-batch`, `ingest strides-batch`: discover activity files, derive date from filename,
  one normalized output per input, `--force` and related flags.
- Reuse Phase 2 ingest logic; no duplicate domain rules.

**Exit criteria:** Batch discovery, idempotent re-runs with `--force`, and date-from-filename
behavior are tested.

---

## Phase 4 — Single-file merge

- `merge.py`: merge one normalized file into `all_splits.csv` or `all_strides.csv` with duplicate
  skip, conflict failure, sort order, and no silent overwrites.
- CLI: `merge run`, `merge strides`.

**Exit criteria:** Tests for duplicates, conflicts, idempotent merge, chronological sorting, and
preservation of existing rows.

---

## Phase 5 — Batch merge

- `merge runs-batch`, `merge strides-batch`: discover normalized files and merge into aggregates
  with the same semantics as single-file merge.
- CLI wiring and glob/`--input-dir` behavior per spec.

**Exit criteria:** Batch idempotence and conflict behavior covered; ordering matches the spec.

---

## Phase 6 — Import (convenience)

- `import run`, `import strides`, `import runs-batch`, `import strides-batch`: ingest then merge,
  including `--merged-output`, `--output-dir`, and shared flags.

**Exit criteria:** Integration-style tests showing import matches the equivalent ingest followed
by merge for representative cases.

---

## Phase 7 — Validate

- `validate.py`: schema and semantic checks for normalized and aggregate files, including ordering,
  allowed types, duplicate identities, and conflict detection as defined for aggregates.
- All `validate` subcommands from the spec.

**Exit criteria:** Tests for each validation failure mode the spec requires.

---

## Phase 8 — Coverage and hardening

- Close gaps against the spec’s test checklist (including line coverage if that remains a
  requirement).
- Edge cases from real Garmin exports; consistent non-zero exit codes on errors.

**Exit criteria:** Coverage target met; error paths behave consistently.

---

## Dependency summary

| Phase | Depends on |
|-------|------------|
| 0 | — |
| 1 | 0 |
| 2 | 1 |
| 3 | 2 |
| 4 | 1, 2 |
| 5 | 4 |
| 6 | 2, 3, 4, 5 |
| 7 | 1–6 |
| 8 | 7 |

Phase 6 wraps ingest and merge for both single-file and batch flows, so it requires phases 2–5.
