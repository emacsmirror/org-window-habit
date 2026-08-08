---
name: org-window-habit
description: Edit Org-mode data for org-window-habit safely. Use when creating or changing window-based habits, CONFIG or OWH_CONFIG properties, completion windows, versioned habit requirements, reset/start dates, or dated pauses and resumptions in .org files. Preserve Org history and repository-specific property-prefix conventions. Do not use for ordinary org-habit entries that have no window-habit configuration.
---

# Org Window Habit

Edit habit data without requiring the package source repository to be the current workspace. Focus on the target Org heading and its configuration history; do not reproduce the human README.

## Inspect before editing

1. Locate the exact heading and read the whole entry, including planning timestamps, property drawer, logbook, and state history.
2. Determine the configured property prefix from nearby window habits. The unified property may be `CONFIG`, `OWH_CONFIG`, or another prefixed form. Preserve the local convention.
3. Preserve unrelated properties, IDs, scheduling timestamps, log entries, heading state, and file formatting unless the request requires changing them.
4. Prefer the unified `CONFIG` property when creating new data. Do not migrate legacy scattered properties merely because they exist.

## Maintain a valid habit entry

For a new active habit, provide:

- a TODO heading;
- a `SCHEDULED` or `DEADLINE` timestamp with any repeater;
- `STYLE` set to `habit` when that is the repository convention;
- a unified config containing `:window-specs`; and
- TODO-state logging so completions appear in the state history.

Use a bare plist for one configuration:

```org
:CONFIG: (:window-specs ((:duration (:days 7) :repetitions 3)) :assessment-interval (:days 1))
```

Use a list of plists for versioned configurations. Put the most recent version first. Every version is a complete configuration, not a patch: retain `:window-specs` and copy any other parameters that should continue to apply.

## Apply `:from` and `:until` precisely

Treat each version as the half-open interval `[from, until)` in local time:

- `:from` is inclusive. The version applies at midnight on that date.
- `:until` is exclusive. The version no longer applies at midnight on that date.
- Missing `:from` means unbounded past.
- Missing `:until` means unbounded future.
- A lone `:from` on a single config starts or resets the habit and ignores earlier completions; it does not represent a pause.

Accepted date values include ISO strings and Org timestamps. Preserve the file's style, normally `"YYYY-MM-DD"` or `"[YYYY-MM-DD Day]"`. Because dates normalize to midnight, use the following day when the user means “through the end of this date.”

For a continuous rule change, pair the newer version's `:from` with the older version's `:until` at the same date:

```org
:CONFIG: ((:from "2026-06-01" :window-specs ((:duration (:days 7) :repetitions 5))) (:until "2026-06-01" :window-specs ((:duration (:days 7) :repetitions 3))))
```

The parser can infer the newer `:from` from an adjacent older `:until` in simple chains, but write both sides when editing data directly. Explicit boundaries make pauses and later edits unambiguous.

Avoid overlaps. A later version must not start before the preceding version ends. Do not put `:from` on the older version to describe the start of a forward pause; that bounds the older version's beginning instead.

## Pause and resume habits

Pause an active single-config habit at date `P` by adding `:until P`. The old configuration remains as history and no configuration is active at or after `P`:

```org
:CONFIG: (:window-specs ((:duration (:days 7) :repetitions 2)) :until "2026-05-21")
```

Pause the current version of an already-versioned habit by adding `:until P` to its first plist. Do not discard older versions.

Resume at date `R` by prepending or restoring a complete newest config with `:from R`, while retaining `:until P` on the previously active config:

```org
:CONFIG: ((:from "2026-08-09" :window-specs ((:duration (:days 7) :repetitions 2))) (:until "2026-05-21" :window-specs ((:duration (:days 7) :repetitions 2))))
```

The habit is inactive throughout `[P, R)`. If requirements change on resume, change only the new first plist and preserve the historical plist verbatim.

Do not treat `STYLE: habit-paused`, removing a repeater, or changing the TODO keyword as equivalent to a dated config pause. Those may hide or disable an entry under local Org conventions, but they do not record an inactive interval for window-habit history. If the repository intentionally uses one of those mechanisms in addition to config dates, preserve that convention and still encode the hiatus with `:until`/`:from` when historical scoring matters.

## Verify the edit

1. Re-read the complete property value and check balanced parentheses and quoting.
2. Confirm version order is newest first and every version contains `:window-specs`.
3. Confirm continuous boundaries are equal, pauses satisfy `until < from`, and no ranges overlap.
4. Confirm a pause date matches midnight semantics; adjust by one day if the user meant the end of a day.
5. Review the diff and ensure no logbook entries or unrelated Org data changed.
6. When the package repository and Nix environment are available, run the narrow ERT/config checks or `just test` for behavior changes. For data-only edits, syntax and diff review are normally sufficient.
