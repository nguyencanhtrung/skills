# Claude Code skills

Engineers, PMs, Technical Leaders [Claude Code](https://claude.com/claude-code) skills.

## review-tracker

Records the **final** review status of an issue/MR into a per-project tracker file at three
moments — start review / end of a round / close-or-merge — so you type one short command instead
of describing the update every session.

- Source of the status is the **current conversation** (the review just happened in-session); the
  command locks it in and writes it down.
- Works for **any project**: one global skill + one config file mapping each project to its tracker
  file, features file, and GitLab/GitHub host.
- Remote (GitLab/GitHub) MCP is used only to *verify* state and refresh captures, and only the
  remote steps require MCP — writing status from the conversation works without it.

### Commands

| Command | When |
|---|---|
| `/rv-start <project> <#issue\|!mr>`   | Beginning a review |
| `/rv-round-end <project> <#issue\|!mr>` | Finishing one review round |
| `/rv-close <project> <#issue\|!mr>`     | The issue/MR is closed/merged |

`#n` = issue, `!n` = MR. Natural-language phrases ("start review #42", "round done",
"merge !31") trigger the same commands if you add the phrase-trigger line below.

### Install

1. Run the [skills.sh](https://skills.sh) installer and pick `review-tracker`:

   ```bash
   npx skills@latest add nguyencanhtrung/skills
   ```

   This clones the repo, finds the skill, and installs it into your agent (Claude Code, etc.).
   No registration needed. To install only this skill non-interactively:
   `npx skills@latest add <you>/<repo> -s review-tracker -g -y`.

   *(Manual alternative, no npx: clone the repo and copy `skills/review-tracker/` into
   `~/.claude/skills/` — Claude Code auto-discovers it.)*

2. Configure on first use — **no setup command needed**. The first time you run a command for a
   project the skill doesn't know yet, it asks you for the fields (`host`, `project_id`, `tracker`,
   `features`) and writes `review-config.json` for you.

   To pre-fill instead, copy the example (the real config is git-ignored, never committed) and edit
   it — one entry per project:

   ```bash
   cd ~/.claude/skills/review-tracker
   cp review-config.example.json review-config.json
   ```

   ```json
   {
     "myproj": {
       "host": "gitlab",
       "project_id": 1234,
       "tracker": "gitlab/sum-myproj.md",
       "features": "code/myproj-features.md"
     }
   }
   ```

   - `host`: `gitlab` or `github` — selects which MCP to check for the remote steps.
   - `project_id`: the numeric project id on your host.
   - `tracker` / `features`: paths **relative to each project's root**. Set `features` to `null`
     if the project has no partner-facing features file.

3. (Optional) Add a phrase-trigger so natural language invokes the commands. Put this in your
   global `~/.claude/CLAUDE.md`:

   > When the user says, in any phrasing, "start review <issue/MR>" run `/rv-start`,
   > "round done / finished this round" run `/rv-round-end`, "merge/close <issue/MR>" run
   > `/rv-close` from the `review-tracker` skill.

### Requirements

- Claude Code with skills support.
- For the remote-verify steps: a GitLab or GitHub MCP server configured and authenticated. The
  status-writing steps work without it; the skill stops and tells you if a remote step needs an
  MCP that is missing.
