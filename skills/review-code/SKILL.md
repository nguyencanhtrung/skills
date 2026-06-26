---
name: review-code
description: 'Deep programming review of a GitLab MR/Issue PLUS a feature<->wiki divergence pass, writing findings into the existing code/track review-notes file. The wiki is a REFERENCE, not gospel: when the implementation is superior it proposes updating the wiki as the canonical solution. Triggers: the command /rv-code and phrases like "deep code review <project> <#n|!n>", "rv-code lekiwi !96". Runs BETWEEN /rv-start and /rv-round-end; writes ONLY the review-notes file, never a tracker, never the wiki.'
---

# review-code

Deep review step for an MR/Issue on a **wiki + codebase** repo (an LLM wiki whose
`code/` tree is one component). Two jobs in one pass:

1. **Programming correctness** — bugs, UB, concurrency/resource, error handling,
   security, performance, API/contract, tests. Independent of the wiki.
2. **Feature <-> wiki divergence** — compare the *implemented feature* against the
   wiki article that documents it. The wiki is a **reference for comparison, not a
   north star**. A divergence is not automatically a defect: the implementation may
   be a legitimate alternative or a **better** solution. When it is better, the
   outcome is a **proposal to update the wiki** so the implementation becomes the
   canonical design.

This is the inverse of `review-3gpp`. There, `raw/` (3GPP specs, T1) and the wiki
are the binding north star and code is judged against them. Here, programming
correctness is primary and the wiki is the thing that may need to change. This
matches the repo's Source Authority Hierarchy (CLAUDE.md): **compiled wiki = rank
7 (lowest); source code = rank 4** — so code *outranks* compiled wiki. The one
guard is the **authority tier** (Step 4): a wiki claim grounded in a raw rank-1/2
source is NOT free to overrule.

**Core principle:** this skill *produces findings + wiki-update proposals*; it does
**not** record status and it does **not** touch the wiki. It writes ONLY into
`code/track/<project>-<target>-<topic>.md` (the review-notes file). It never edits
the tracker, the features file, the top-level verdict, or any `wiki/` article.
Wiki changes happen later, gated, via the project's `correct wiki` / `compile`
workflow after a human approves the proposal.

## Lifecycle position

```
/rv-start <project> <#n|!n>      pulls capture + creates review-notes framework  (review-tracker)
/rv-code  <project> <#n|!n>       THIS skill: deep review -> fills Round-N findings + wiki proposals
   (human reads + approves the findings + wiki proposals)
/rv-round-end <project> <#n|!n>  locks findings into the tracker                  (review-tracker)
```

If the review-notes framework does not exist yet, **STOP** and tell the user to run
`/rv-start` first. Do not recreate it. Invoke manually — this skill is NOT
auto-run by `/rv-start` (unlike dronava -> review-3gpp).

## Config + conventions

Config schema, the `#n`/`!n` number convention, and the rule for locating the
review-notes file (glob `code/track/*<target>*`; the target token decides the file)
all follow **review-tracker** — see `~/.claude/skills/review-tracker/SKILL.md`. This
skill is read-only on config. If the project entry is missing -> STOP, ask the user
to add it via `/rv-start`; do not guess `project_id`.

## Step 0 — Locate review-notes + determine round N

1. **Locate** the review-notes file: `code/track/*<target>*`, case-insensitive.
2. **No file found -> STOP.** Tell the user: "No review-notes file for `<target>`.
   Run `/rv-start <project> <#n|!n>` first." Write nothing.
3. **Round N** = (highest `R<k>` in the `### Round timeline` table) + 1. If the only
   row is the `/rv-start` in-review placeholder with no findings, this is **R1**.

## Step 1 — Area map (auto-derive + confirm + cache)

The map ties code paths to the wiki article(s) that document them. It is
**per-project**, cached at `code/track/rv-code-areas.yaml`.

- **Cache hit:** read `code/track/rv-code-areas.yaml` and use it. (Let the user
  amend it by hand anytime.)
- **Cache miss:** derive a candidate map and **confirm before caching**:
  1. List code roots/packages (e.g. `code/<project>/packages/*`) and the wiki
     `wiki/Topics/*` + `wiki/Concepts/*`.
  2. Match by name similarity; if the `code-review-graph` MCP has a graph for this
     repo, use `semantic_search_nodes_tool` / `get_review_context_tool` to improve
     code<->doc matches.
  3. Show the candidate map (path-prefix -> wiki article(s)) and **ask the user to
     confirm/edit**. Mark unmatched paths as `wiki: none` (reviewed for
     programming only).
  4. On confirm, write `code/track/rv-code-areas.yaml`:
     ```yaml
     code_root: code/<project>          # where the implementation lives (read-only)
     areas:
       - path: packages/intelligence/   # prefix under code_root
         wiki: [wiki/Topics/intelligence-workstream-soc-one-poc.md]
       - path: packages/simulation/
         wiki: [wiki/Topics/simulation-workstream-soc-one-poc.md]
       - path: packages/contracts/
         wiki: [wiki/Topics/...]
       - path: packages/platform/
         wiki: none                      # no wiki baseline -> programming-only
     ```

Verify every `wiki:` path exists before handing it to a subagent (a missing article
is itself a gap to report, not a silent skip).

## Step 2 — Refresh capture + compute the delta (round-aware)

Pin provenance before reading any code. Use `git -C <code_root> rev-parse` to
confirm the implementation tree is at the head you intend to review; never read the
working tree and attribute it to a different ref. If `<code_root>` cannot be put at
the MR head, read changed files via `mcp__gitlab__get_file_contents` at the ref.

- **Round 1:** assume `/rv-start` just refreshed `gitlab/<project>/{mr,issue}-<n>.md`.
  Read it. Base..head = the full MR diff.
- **Round n>1:** re-run the GitLab Query Capture (MCP, per CLAUDE.md) to refresh the
  capture, then review the **delta only** — commits/comments since the prior round's
  recorded `Head:` SHA. (The wiki pass in Step 3B is still feature-level, see below.)

**Pre-check local capture before any remote MCP call** (per CLAUDE.md): if a fresh
capture exists and the user did not ask to refresh, use it; only call
`mcp__gitlab__*` when the capture is missing/stale or this is round n>1.

Resolve the diff source: **MR** -> file list via
`mcp__gitlab__get_merge_request_diffs`. **Issue** -> parse the capture's discussion
for referenced commit SHAs / branch / linked MR; resolve to a concrete `base..head`.
If nothing concrete is referenced -> **STOP and ask** which commits to review.

If a `code-review-graph` graph exists, call `get_impact_radius_tool` /
`get_affected_flows_tool` on the changed files to widen review context (callers,
affected flows) beyond the literal hunks.

## Step 3 — Two-wave fan-out (plain parallel Agent calls — NO Workflow)

Spawn `Agent`s (`subagent_type: general-purpose`) in a single message so they run in
parallel. Two waves:

### Wave A — programming dimensions (whole diff)

One agent per concern, each scanning the **entire diff** (with impact context):
`correctness` · `concurrency/resource/lifetime` · `error-handling/failure-modes` ·
`security/input-validation` · `performance` · `API/contract/compat` · `tests`.
Each agent returns programming findings tagged **B/C/M/L** (see taxonomy).

### Wave B — feature <-> wiki divergence (feature-level, per area)

One agent per touched area (from the map) **whose `wiki:` is not `none`**. Each agent:

- reads the **whole implemented feature** in that area's touched modules (NOT just
  the diff hunks — a divergence may pre-date this MR), and the mapped wiki article(s),
- lists every place the implementation and the wiki **disagree** (behavior,
  algorithm, parameters, data flow, structure, stated constraints),
- for each disagreement, runs the **authority-tier check** (Step 4) and assigns a
  verdict **D / W / N** with evidence.

Each Wave-B record: `{tag(D|W|N), code_ref(file:line), wiki_ref(article#section or
line), wiki_tier(raw-rank1-2 | compiled), claim_code, claim_wiki, evidence,
recommendation}`.

Merge + dedup across agents. Dedup key = `(file, line-or-feature, finding kind)`.

## Step 4 — Authority tier (the safety gate for D vs W)

For each wiki disagreement, trace the wiki claim to its source: read the article's
`compiled_from:` frontmatter -> the referenced `wiki/_index/*-index.xml` -> the raw
source. Classify:

- **Wiki claim grounded in raw rank 1-2** (spec / datasheet / peer-reviewed paper):
  the claim is **hard**. Default the divergence to **D — fix the code**. The code
  "wins" only with strong objective evidence (measurement, physics, a passing test
  that contradicts the source). If such evidence genuinely exists, do NOT silently
  flip to W — emit it as **D + escalation note**: "source may be outdated / code
  measurement contradicts [TS/paper §X]; needs human + possibly a raw re-ingest."
- **Wiki claim is compiled synthesis only** (rank 7, no hard source): the claim is
  **soft**. Then:
  - code is the better solution **with concrete evidence** -> **W** (propose wiki update),
  - code differs but is **equivalent** -> **N** (document the choice),
  - code is **clearly worse** -> **D** (fix code).

**W-evidence bar (mandatory):** a **W** requires concrete evidence — benchmark,
test, complexity argument, or an established best practice with a citation — NOT
reviewer preference. Preference-only -> downgrade to **N**.

## Step 5 — Loop-until-dry

Repeat the Step 3 fan-out, deduping each round against **everything already seen**
(not just confirmed findings, so the loop converges), until **K=2 consecutive
rounds surface no new finding**. Announce the running counts after each round so the
user can interrupt. Cap at ~6 loop rounds; report if the cap is hit before dry.

## Step 6 — Completeness gate

Before writing: assert every changed hunk was either reviewed under Wave A or
explicitly marked out of scope, and every touched area with a wiki baseline was
compared in Wave B. State coverage explicitly, e.g. "9/9 changed files reviewed;
2 areas compared to wiki; 1 package has no wiki baseline (platform)."

## Step 7 — Taxonomy + write-out (two tables)

Two orthogonal tag sets — keep them in two separate tables.

**Programming findings** (`### Findings`):

| Tag | Meaning |
|---|---|
| **B**n | Blocker — crash, UB, data loss, security hole, interop break |
| **C**n | Correctness — wrong behavior / logic bug (no crash) |
| **M**n | Medium — performance, test gaps, maintainability, scope |
| **L**n | Low — cosmetic / nit |

**Wiki divergences** (`### Wiki divergences & update proposals`):

| Tag | Meaning |
|---|---|
| **D**n | Code disagrees with wiki AND wiki is right (hard-sourced, or code clearly worse) -> **fix code**. Cite the wiki claim + its tier. |
| **W**n | Code is the better / canonical solution -> **propose wiki update**. Include: target article, old claim, proposed new content, the evidence. (Compiled-only wiki claims only; needs W-evidence.) |
| **N**n | Divergence but equivalent -> **document the choice**; no action. |

Write into the **current `## RN (YYYY-MM-DD)` block** of `# ARCHIVE` in the
review-notes file (review-tracker template), with:

- a `Base: <sha>` / `Head: <sha>` provenance line + one-line scope,
- the `### Findings` table (B/C/M/L),
- the `### Wiki divergences & update proposals` table (D/W/N) — each W row carries
  the proposed wiki edit in enough detail to apply later,
- a `### Programming verification (head <sha>)` table `| Claim | Evidence |` for the
  load-bearing correctness/perf claims,
- update the `### Per-finding status` table (stable IDs) and add the `R<N>` row to
  the **top** of the `### Round timeline` table with a 🟢/🟡/🔴 status + summary.

Bump front-matter `updated`. Do **not** set the `## FINAL STATUS` verdict, do **not**
touch the tracker / features file, and do **not** edit any `wiki/` article.

## Step 8 — Hand-off

Report in plain ASCII, concise:

- findings written to `<path>`,
- programming count by severity (e.g. `B:1 C:2 M:3 L:1`) and divergence count
  (e.g. `D:1 W:2 N:1`),
- the coverage statement from Step 6,
- a one-line list of the **W** proposals (article + gist) so the user can decide
  whether to run `correct wiki` after the round,
- next step: "Review the findings + wiki proposals, then run
  `/rv-round-end <project> <#n|!n>` to lock them into the tracker. Apply approved W
  proposals via `correct wiki` / `compile`."

## Common mistakes

| Mistake | Do instead |
|---|---|
| Treat the wiki as binding (like rv-3gpp) | Wiki is a reference; code may legitimately be better |
| Auto-edit the wiki | Read-only: propose in review-notes; wiki changes are gated `correct wiki` |
| Flip to W against a hard-sourced wiki claim | Hard source -> D + escalation note, never a silent W |
| Tag W on reviewer preference | W needs concrete evidence; preference -> N |
| Wiki pass only on the diff | Wave B is feature-level: read the whole implemented feature in touched modules |
| Re-review the whole MR on round n>1 | Programming pass = delta since prior `Head:`; wiki pass stays feature-level |
| Use Workflow for the fan-out | Plain parallel `Agent` calls (per dimension, then per area) |
| Hardcode the area map | Auto-derive + confirm + cache to `code/track/rv-code-areas.yaml` |
