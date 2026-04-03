# Lappy

Lappy is a tool for organizing and working with running workout split data.

Its purpose is to help users keep raw Garmin split exports and cleaned workout data in a
consistent local folder structure, so that runs and strides can be processed and analyzed more
easily.

## Data folder setup

Create a local `data/` folder for your workout files.

This folder is for local use only and should **not** be checked into git.  
The repository should ignore it via `.gitignore`.

## Layout

```text
data/
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

## Meaning

- `data/[goal]/raw/`  
  Raw Garmin split CSV exports

- `data/[goal]/runs/`  
  Processed non-strides workout split files and merged run history

- `data/[goal]/strides/`  
  Processed strides workout split files and merged strides history

## Notes

- `[goal]` is a user-chosen folder name for a training goal, season, race, or similar grouping
- You may create multiple `[goal]` folders under `data/`
- Keep all files under `data/` local to your machine
