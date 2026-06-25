---
name: review-tracker
description: Use when starting a code review of an issue/MR, finishing a review round, or closing/merging one, and the project keeps a status tracker file. Triggers include the commands /rv-start /rv-round-end /rv-close and phrases like "start review", "round done", "merge/close" naming an issue/MR. GitLab or GitHub.
---

# review-tracker

Records the **final** review status of an issue/MR into a per-project tracker file at three moments (start / end-of-round / close), so the developer types one short command instead of describing the update every session.

**Core principle:** the source of the status is **this conversation** — the review just happened in-session. The command locks it in and writes it down. Remote (GitLab/GitHub) is for *verifying* state and refreshing captures, not the source of the round result.

## Config

Resolve every command through `~/.claude/skills/review-tracker/review-config.json` (lives next to this file):

```json
{ "<project>": { "host": "gitlab|github", "project_id": N, "tracker": "<path>", "features": "<path>|null" } }
```

`tracker` / `features` are **relative to the current project root (cwd)**. `features: null` means the project has no partner-facing features file — skip all features steps for it.

If config or the project's entry is missing, **do not guess** — see Bootstrap below.

## Number convention

`#n` = issue, `!n` = MR (e.g. `/rv-close dronava !31`). The `!`/`#` decides which capture and which row.

## Locating the review-notes file (DO NOT assume the path)

Notes filenames are inconsistent across history: `MR31-review-notes.md`, `mr25-review-notes.md`, `issue101-dci30-t1-verification.md`, `issue75-pdcp-review-notes.md`. **Glob `code/track/` for a file whose name contains the target** (e.g. `*MR31*` or `*issue101*`, case-insensitive) before creating one. Use the existing file if found — never create a second file that orphans real notes. Only if none matches, create a new one (template below).

**The target token decides the file, not relatedness.** `!n` -> an MR notes file, `#n` -> an issue notes file. An MR that fixes issue `#72` does NOT reuse `issue72-*.md` — that is the issue's record; the MR gets its own. Linked-item notes stay separate.

**Naming convention:** `code/track/<project>-mr<n>-<topic>.md` (MR) or `code/track/<project>-issue<n>-<topic>.md` (issue). Example: `dronava-mr21-rlc-design.md`, `dronava-issue21-pdcp-design.md`. Use `<project>` lowercase, topic short and hyphenated.

**If the found file's name does NOT match this convention** (e.g. `MR31-review-notes.md`, `mr25-review-notes.md`): rename it with `git mv` to the correct name before proceeding. Update any internal front-matter that references the old path.

## Commands

### `/rv-start <project> <#n|!n>`

Remote-first (this step needs MCP):

1. **MCP preflight.** Confirm the `host` MCP is reachable (GitLab: `mcp__gitlab__whoami`). Unreachable -> **STOP**, tell the user how to install/authenticate it. Write nothing.
2. **Refresh capture** -> `gitlab/<project>/mr-<n>.md` or `issue-<n>.md` per the GitLab Query Capture Protocol (CLAUDE.md). If remote already shows `merged`/`closed` (contradicting "about to review") -> surface it before proceeding.
3. **Find-or-create review-notes** (see globbing rule above). On create, fill the `## Round 1` Scope from the capture (factual), leave Findings/T1/Outcome as placeholders, and seed the R1 row in the Rounds-summary table (🟡 in-review) — results land at `/rv-round-end`.
4. **Tracker:** mark the row in-review. The reviewer is the **review owner** (current git user), not the MR author/assignee. If the tracker has no in-review state value (only `opened/merged/closed`), encode it in the row's Notes cell (e.g. `review started <date> (<owner>), R1 in-review`); leave round results blank there.
5. **Auto deep-review (project `dronava` only).** After steps 1-4 finish and the review-notes framework exists, immediately invoke the `review-3gpp` skill (its `/rv-3gpp` flow) for the same `<project> <#n|!n>`, to fill the Round-N findings. **Only for `<project> == dronava`** — for any other project, stop after step 4; do not run `/rv-3gpp`.

### `/rv-round-end <project> <#n|!n>`

From the conversation. Fixed write order:

1. **Review-notes first** — add a `R<N>` row to the **top** of the Rounds-summary table, then insert a new `## Round N` block (scope, findings, T1 cites, outcome) **above** the previous round (newest on top). Use 🟢/🟡/🔴 + text on the row status and each finding.
2. **Then tracker** — finalize the row from this round. Tracker layouts vary; update whatever columns it has (status/state cell + Notes), and reflect round outcome in Notes when there is no dedicated status column.

### `/rv-close <project> <#n|!n>`

1. **MCP preflight + refresh capture.** Verify the item is actually `merged`/`closed` remotely. **Reconcile linked items:** for an MR, also re-query each issue it closes. If a capture is stale (says `opened` but remote says closed) -> fix the capture. If remote disagrees with the in-session claim (item NOT actually closed/merged upstream) -> **STOP and surface the conflict**; the skill records status, the human closes/merges upstream (it does not merge/close for them).
2. **Review-notes** set final Verdict; **tracker** set state `closed`/`merged`. Adapt to the file's existing structure (e.g. an existing `## STATUS:`/`## Verdict` block) — normalize it, do not impose the template over rich existing content.
3. **Features** (skip if `null`): only if a **capability-level** status actually changes (e.g. a layer reaching "Supported"), **show a diff and ask** before writing the partner-facing file. Never auto-write it.

### `/rv-refactor <project>`

Batch-fix all notes files for `<project>` in `code/track/` with a wrong name or
non-conforming content (discover → classify → confirm → rename+refactor → report).
Full steps: `~/.claude/skills/review-tracker/reference/refactor.md`.

## A no-op is a valid outcome

Much of a close may already be done (a prior session edited the tracker by hand). If an artifact already reflects the new state, **say so in one line and do not rewrite it**. Do not manufacture edits to look busy. Bump a file's `updated`/`last_queried` only when you actually changed its content.

## Bootstrap (missing config / entry / file)

Never guess paths. Ask the developer, then write:

- Config file missing -> ask `{host, project_id, tracker, features}` for this project; create the file with that one entry.
- Entry for `<project>` missing -> ask the 4 fields; append the entry.
- `tracker` path not found in repo -> ask whether to create an empty tracker from the tracker structure (frontmatter `type: tracker` + feature-by-layer tables + Issues table + MRs table + Decisions log).
- `features` path not found and not `null` -> ask: create it / set the entry to `null` / fix the path. **Seed unknown features paths as `null`, not a dead path.**

After asking once and writing, later runs do not re-ask.

## Review-notes template (only when creating a new one)

```markdown
---
project: <project>
target: <MR31|issue101>
host: gitlab
state: in-review        # in-review | merged | closed
started: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
---

# <project> <target> — review notes

## Verdict
🟡 <set on /rv-close: 🟢 APPROVE / 🟡 APPROVE-WITH-NOTES / 🔴 CHANGES-REQUESTED + 1-2 lines>

## Rounds summary
<!-- newest round on top -->
| Round | Date | Status | Summary |
|---|---|---|---|
| R1 | YYYY-MM-DD | 🟡 in-review | <scope in a few words; open/fixed count> |

Legend: 🟢 done/approved · 🟡 in-review/notes/deferred · 🔴 blockers open

## Round 1 (YYYY-MM-DD)
- **Scope:** <what was reviewed>
- **Findings:**   (B = blocker, D = deviation; tags stable across rounds)
  - 🔴 B1 <blocker> — open
  - 🟡 D1 <deviation> — deferred
  - 🟢 D2 <deviation> — fixed
- **T1 cites:** [TS 38.211 sec 8.3.1.5-6], ...
- **Outcome:** <fixed / open / deferred per finding; next step>
```

## Common mistakes

| Mistake | Do instead |
|---|---|
| Trust the in-session "it's closed" claim | Re-query remote on close; reconcile linked issues; STOP on conflict |
| Auto-edit the features file | Diff + ask; skip if no capability change or `features: null` |
| Gate file-writes on MCP | Only remote steps gate on MCP; file writes from conversation do not |
