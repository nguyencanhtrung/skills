---
name: review-3gpp
description: 'Deep program-correctness + 100% 3GPP-compliance review of a GitLab MR/Issue, writing findings into the existing code/track review-notes file. Triggers: the command /rv-3gpp and phrases like "deep 3gpp review <project> <#n|!n>", "deep review dronava !25". Runs BETWEEN /rv-start and /rv-round-end; writes ONLY the review-notes file, never a tracker.'
---

# review-3gpp

Deep review step for an MR/Issue: correctness **and** 100% 3GPP compliance. It
reads the diff, maps every changed line to its spec authority (spec-model
assertion -> wiki baseline -> raw PDF clause), corroborates every numeric claim
from two independent sources, and writes structured findings into the existing
review-notes file. It does the analysis a human reviewer would otherwise do by
hand and hand-type into the `## Round N` block.

**Core principle:** this skill *produces findings*; it does **not** record
status. It writes ONLY into `code/track/<project>-<target>-<topic>.md`. It never
edits `gitlab/sum-<project>.md` (tracker) or `code/<project>-features.md`
(features) or the top-level `## Verdict` -- those belong to `/rv-round-end` and
`/rv-close`.

## Lifecycle position

```
/rv-start <project> <#n|!n>       pulls capture + creates review-notes framework  (review-tracker)
/rv-3gpp  <project> <#n|!n>        THIS skill: deep review -> fills Round-N findings
   (human reads + approves the findings)
/rv-round-end <project> <#n|!n>   locks findings into the tracker                  (review-tracker)
```

If the review-notes framework does not exist yet, **STOP** and tell the user to
run `/rv-start` first. Do not recreate it.

## Config + conventions

Config schema, the `#n`/`!n` number convention, and the rule for locating the
review-notes file (glob `code/track/*<target>*`; the target token decides the
file, so an MR fixing `#72` never reuses `issue72-*.md`) all follow
**review-tracker** — see `~/.claude/skills/review-tracker/SKILL.md`. This skill is
read-only on config: do not fork or duplicate it.

If the project entry is missing -> STOP, ask the user to add it via `/rv-start`;
do not guess `project_id`.

## Step 0 - Locate review-notes + determine round N

1. **Locate** the review-notes file per the glob rule above (review-tracker's
   convention): `code/track/*<target>*`, case-insensitive.
2. **No file found -> STOP.** Tell the user: "No review-notes file for
   `<target>`. Run `/rv-start <project> <#n|!n>` first." Write nothing.
3. **Round N** = (highest `R<k>` in the Rounds-summary table) + 1. If the only
   row is the `/rv-start` in-review placeholder with no findings yet, this is
   **R1**.

## Step 1 - Refresh capture + compute the delta (round-aware)

Pin provenance before reading any code. Use `git -C code/v-<project> rev-parse`
to confirm the clone is at the head you intend to review; never read the working
tree and attribute it to a different ref.

- **Round 1:** assume `/rv-start` just refreshed
  `gitlab/<project>/{mr,issue}-<n>.md`. Read it. Base..head = the full MR diff
  (or, for an issue, the diff resolved in Step 2).
- **Round n>1:** re-run the GitLab Query Capture (MCP, per CLAUDE.md) to refresh
  the capture file with new commits + comments. Then compute the **delta only** --
  commits/comments added *since the prior round's recorded `Head:` SHA*. Review
  only the delta, not the whole MR again.

Record `Base: <sha>` / `Head: <sha>` in the new Round-N block (matching
`dronava-mr31-retx-rb.md`).

**Pre-check local capture before any remote MCP call** (per CLAUDE.md): if a
fresh capture already exists and the user did not ask to refresh, use it; only
call `mcp__gitlab__*` when the capture is missing/stale or this is round n>1.

## Step 2 - Resolve the diff source

- **MR:** file list via `mcp__gitlab__get_merge_request_diffs`; read the actual
  changed code in the local clone `code/v-<project>/` at the pinned head.
- **Issue:** parse the capture's `## Raw discussion` + activity for referenced
  commit SHAs / branch names / linked MR; resolve to a concrete `base..head`
  diff. If nothing concrete is referenced -> **STOP and ask** which commits to
  review.

## Step 3 - Classify changed files by spec-area

Map each changed path to a layer (dronava OAI layout):

| Path prefix | Spec-area | Baselines (spec-model -> wiki T2 -> raw T1) |
|---|---|---|
| `openair2/LAYER2/nr_rlc/` | RLC | `code/v-dronava/docs/design/nr-sidelink/nr-rlc-layer/ts38322-rlc-spec-model.md`; `wiki/Topics/nr-rlc-layer.md`; `raw/ts_138322v170100p.pdf` (+ `raw/ts_138321v170400p.pdf` for SL LCID) |
| `openair2/LAYER2/nr_pdcp/` | PDCP | `.../nr-pdcp-layer/pdcp-spec-model.md`; `wiki/Topics/nr-pdcp-layer.md`; `raw/ts_138323v170100p.pdf` |
| `openair2/SDAP/nr_sdap/` | SDAP | `.../nr-sdap-layer/sdap-spec-model.md`; `wiki/Topics/nr-sdap-sidelink-canonical-vi.md`; `raw/ts_137324v170000p.pdf` (+ `raw/ts_124587v170600p.pdf`, `raw/ts_123287v170600p.pdf`) |
| `openair2/LAYER2/MAC/`, sched/LCP/HARQ | MAC | `wiki/Concepts/sl-logical-channel-prioritization.md`, `wiki/Concepts/sidelink-scheduler-paradigm.md`; `raw/ts_138321v170400p.pdf` |
| `openair2/PHY/NR_SIDELINK/` | PHY | `wiki/Concepts/pucch-sl-harq-feedback.md`; `raw/ts_138211/212/213/214 v170100p.pdf` |
| RRC IEs / ASN.1 | RRC | `raw/ts_138331v170100p.pdf` |
| `code/v2x-midware/...` | midware (ETSI/SAE) | no 3GPP layer baseline -- review correctness only; route spec-compliance to the completeness gate |

Verify each baseline path exists before handing it to a subagent (a missing
spec-model/wiki file is itself a gap to report, not a silent skip). Paths with no
mapping go to the completeness gate (Step 6), not the trash.

## Step 4 - Fan-out review (plain parallel Agent calls -- NO Workflow)

Spawn one `Agent` (`subagent_type: general-purpose`) **per touched spec-area**, in
a single message so they run in parallel. Each agent prompt MUST include:

- the changed hunks for its area at the pinned head SHA (paste them, or give the
  file:line ranges + clone path + head SHA),
- its authority chain: spec-model assertion index `#ASR-N` -> wiki T2 baseline ->
  raw T1 PDF -- with the instruction to walk the **mandatory chain from CLAUDE.md**
  for every spec-touching line, stopping at the first tier that fully resolves the
  claim, T1 always winning,
- the **corroboration rule**: every numeric / bit-field / enum / ASN.1 bound /
  timer / constant is a *candidate*, not T1, until a 2nd independent
  representation agrees. Columnar PDF data MUST be read via `pdftotext -layout`
  or a rendered page image (Read `pages:`), never reflowed text. Corroborate
  against: layout re-extract, generated ASN.1 header (`NR_*.h` / `E1AP_*.h`), code
  usage, or T3. Single-source -> tag provisional
  `<!-- verify: T1? [spec §X] -- single-extraction, unconfirmed -->`, state the
  uncertainty, never assert flat,
- known traps: **PDF superscript flattening** (`2^[X]-1` extracted may mean
  `2^([X]-1)` -- verify power-of-two formulas vs algorithm/code, not the literal);
  PDF columnar name<->value mis-pairing,
- the finding taxonomy + citation format (below),
- the instruction to return **raw structured findings** (a list of records), not
  prose.

Each finding record: `{tag, file:line, title, spec_area, clause, spec_model_asr,
wiki_line, t1_cite, verify_tier (T1|T1?|T2|T3), corroboration (2 sources or
"single"), verdict, note}`.

Merge + dedup across agents. Dedup key = `(file, line-or-clause, finding kind)`.

## Step 5 - Loop-until-dry (full)

Repeat Step 4 fan-out rounds, deduping each round's findings against **everything
already seen** (not just confirmed findings -- so the loop converges), until
**K=2 consecutive fan-out rounds surface no new finding**. Each loop also re-runs
the corroboration pass on any numeric claim still tagged `T1?`. Announce the
running finding count after each loop round so the user can interrupt. Guard
against runaway: cap at a small number of loop rounds (e.g. 6) and report if the
cap is hit before going dry.

## Step 6 - Completeness gate

Before writing: assert every changed hunk was either assigned to a spec-area and
reviewed, or explicitly marked "no spec baseline / not spec-relevant". Surface any
unaccounted hunk as a gap in the round block -- never silently truncate. State
coverage explicitly, e.g. "12/12 changed files reviewed; 1 has no 3GPP baseline
(midware ETSI)".

## Step 7 - Finding taxonomy + write-out

Severity tags (stable across rounds):

| Tag | Meaning |
|---|---|
| **B**n | Blocker -- crash, UB, hard fault, or interop-breaking spec violation |
| **D**n | Deviation -- code violates a 3GPP rule that **is** defined (cite the broken clause) |
| **U**n | **Undefined-in-3GPP** -- spec is silent; code made an implementation choice. Record: where the spec is silent (clause), what the code chose, interop risk (safe single-vendor? breaks vs a compliant peer?), and a recommendation (raise issue / document / OK). Keep distinct from D. |
| **M**n | Medium -- test efficacy, scope, bundled change |
| **L**n | Low -- cosmetic / nit |

Each finding carries its verification chain inline:
`code file.c:line -> spec-model #ASR-N -> wiki T2 line -> raw T1 [TS 38.xxx vX §Y]`
with `<!-- verify: T1 [...] -->` (or `T1?` if provisional, `T2`/`T3` as used).

Write into the **current `## Round N (YYYY-MM-DD)` block** of the review-notes
file, reusing the existing structures (see `dronava-mr31-retx-rb.md`):

- the `## Round N (YYYY-MM-DD)` heading with a `Base: <sha>` / `Head: <sha>`
  provenance line and a one-line scope,
- a `### Findings` table: `| # | Finding | Verdict | Note |` (use the B/D/U/M/L
  tags in the `#` column),
- a `### 3GPP / structural verification (T1 + code, head <sha>)` table:
  `| Claim | Evidence |`,
- add the `R<N>` row to the **top** of the Rounds-summary table with a 🟢/🟡/🔴
  status + short summary (newest round on top).

Bump front-matter `updated`. Do **not** set `## Verdict`, and do **not** touch the
tracker or features file.

## Step 8 - Hand-off

Report, in **plain ASCII, concise** (per `feedback-plain-ascii-no-ai-tells`):

- findings written to `<path>`,
- count by severity (e.g. `B:1 D:3 U:2 M:1 L:0`),
- the coverage statement from Step 6,
- next step: "Review the findings, then run `/rv-round-end <project> <#n|!n>` to
  lock them into the tracker."

## Common mistakes

| Mistake | Do instead |
|---|---|
| Re-review the whole MR on round n>1 | Refresh capture, diff only the delta since prior `Head:` SHA |
| Assert a numeric value from one PDF read | Corroborate with a 2nd source; else tag `T1?` provisional |
| Read reflowed `pdftotext` for tables/ASN.1 | Use `pdftotext -layout` or a page image |
| Trust literal `2^[X]-1` from a PDF | Verify power-of-two formulas vs algorithm/code (superscript trap) |
| Conflate undefined-in-spec with a violation | `U` = spec silent (impl choice); `D` = defined rule broken |
| Drop an unmapped changed file silently | Surface it at the completeness gate as a gap |
| Use Workflow for the fan-out | Plain parallel `Agent` calls (one per spec-area) |
