# `/rv-refactor <project>`

Batch-fix all notes files for `<project>` in `code/track/` that have a wrong name
or non-conforming content. Files that are already correct are silently skipped.

**Step 1 — Discover files.**
Glob `code/track/` for files that belong to this project:
- name starts with `<project>-` (already named correctly), OR
- front-matter contains `project: <project>` (old name, not yet renamed).

Also glob case-insensitively for the raw target token (e.g. `*MR31*`, `*issue75*`)
when the project field is missing from front-matter — infer project from context
only if unambiguous.

**Step 2 — Classify each file.** For every discovered file, check independently:

| Check | Pass condition |
|---|---|
| **Name** | Matches `<project>-(mr\|issue)<n>-<topic>.md`, lowercase, hyphenated topic |
| **Front-matter** | Has all required fields: `project`, `target`, `host`, `state`, `started`, `updated` |
| **Sections** | Has `## Verdict`, `## Rounds summary` table, at least one `## Round N` block |

Collect a summary list before touching anything:
```
NEEDS RENAME:   mr31-review-notes.md → dronava-mr31-rlc-design.md
NEEDS CONTENT:  dronava-mr25-pdcp.md  (missing Verdict section)
OK (skip):      dronava-mr21-rlc-design.md
```
**Show this list to the user and ask for confirmation before writing anything.**

**Step 3 — Apply fixes (after confirmation).**

For each file needing a fix, in this order:
1. **Rename first** (if needed): `git mv <old> <new>`. Update `target` in front-matter if it contains the old filename stem.
2. **Refactor content** (if needed): preserve all existing data (findings, round notes, dates, decisions). Only add missing structural elements — do NOT delete or reword existing content. Specifically:
   - Add missing front-matter fields with inferred values (e.g. `state: merged` if Verdict says APPROVE).
   - Add empty `## Verdict` placeholder if missing.
   - Add `## Rounds summary` table seeded from existing `## Round N` blocks if missing.
   - Reorder sections to match template order (Verdict → Rounds summary → Round blocks newest-first) if out of order.

**Step 4 — Report.**
After all writes:
```
Renamed:   2 files
Refactored: 1 file
Skipped:   3 files (already correct)
```
List each changed file with one-line description of what changed.
