# openwiki-cc

A native **Claude Code** and **OpenAI Codex** port of
[OpenWiki](https://github.com/langchain-ai/openwiki) — an agent that generates and maintains a
documentation wiki (`openwiki/`) for any repository.

OpenWiki ships as a standalone CLI on its own harness (DeepAgents/LangGraph, a local shell
backend, a SQLite checkpointer, provider adapters, an Ink TUI). Coding agents like Claude Code
and Codex already provide all of that plumbing. This repo extracts **the agent itself** — the
documentation system prompt, the git-evidence collection, the wiki structure, and the idempotence
logic — and re-expresses it natively for each host:

- **Claude Code** — a slash-command plugin: `commands/wiki.md` → `/openwiki:wiki`.
- **Codex** — a skill: `.agents/skills/openwiki/SKILL.md` → `$openwiki`.

The system prompt and the exact git commands are reproduced **verbatim from OpenWiki's real
source**, not from memory.

## What it does

Point it at a repository and it writes human- and agent-friendly Markdown documentation under
`openwiki/`, with `quickstart.md` as the entrypoint and thematic section pages (architecture,
workflows, operations, …). It grounds every claim in source files, existing docs, and git
history — and on later runs it updates only what actually changed.

## Install — Claude Code

### Via the plugin marketplace (recommended)

Inside Claude Code:

```
/plugin marketplace add SoulKyu/openwiki-cc
/plugin install openwiki@openwiki-cc
```

Then `/openwiki:wiki` is available in every project. `/plugin marketplace update openwiki-cc`
pulls new versions.

> Plugin commands are always namespaced `plugin:command`, so the command is `/openwiki:wiki`
> (never a bare `/openwiki`). Install manually instead if you want a bare `/wiki`.

### Manual (no marketplace)

Copy the single command file into the repo you want to document, or globally. Under
`.claude/commands/` the name is bare — the file becomes `/wiki`:

```bash
# per-project → /wiki
mkdir -p your-repo/.claude/commands
cp commands/wiki.md your-repo/.claude/commands/

# or global → /wiki everywhere
cp commands/wiki.md ~/.claude/commands/
```

## Install — Codex

Codex loads skills from `.agents/skills/`. Copy the skill folder globally (available in every
repo) or into a specific repo, then restart Codex:

```bash
# global → available everywhere
mkdir -p ~/.agents/skills
cp -r .agents/skills/openwiki ~/.agents/skills/

# or per-repo
mkdir -p your-repo/.agents/skills
cp -r .agents/skills/openwiki your-repo/.agents/skills/
```

Invoke it explicitly with `$openwiki` (or via the `/skills` menu); Codex may also trigger it
implicitly when you ask to "initialize / update the openwiki docs". It auto-detects init vs
update from whether `openwiki/` already exists.

> Codex skills don't take slash arguments, so the mode comes from your phrasing (or the
> `openwiki/` auto-detect) rather than an `init`/`update` token. Codex custom prompts
> (`~/.codex/prompts/`) are deprecated, so this ships as a skill.

## Usage

**Claude Code** — from the root of the target repository. Installed as a plugin the command is
`/openwiki:wiki`; copied under `.claude/commands/` it is `/wiki`. Both take the same arguments:

| Command | Behavior |
|---|---|
| `/openwiki:wiki` | **Auto-route**: if `openwiki/` exists → update, else → init. |
| `/openwiki:wiki init` | Build the wiki from scratch. |
| `/openwiki:wiki update` | Surgically refresh only the pages affected by recent changes. |
| `/openwiki:wiki update <instruction>` | Same, plus an extra instruction (e.g. `document the API routes first`). |

> **Note** — auto-route replaces upstream OpenWiki's interactive-chat default (a slash command
> can't be a bare `/openwiki` anyway; plugin commands are always namespaced). `init` and `update`
> remain explicit.

**Codex** — invoke `$openwiki` (or ask to "update the openwiki docs"). It auto-detects init
(no `openwiki/`) vs update (`openwiki/` exists); say "initialize" or "update" to force a mode,
and add any extra instruction in the same request.

## How it works

**Git evidence (before any write).** The command reads `openwiki/.last-update.json`, then runs
the exact upstream commands (all `git --no-pager`, all read-only):

- always: `git status --short`, `git rev-parse HEAD`, `git diff --name-status HEAD`
- init / no prior metadata: `git log --max-count=20 --name-status --oneline`
- update with a recorded head: `git log <gitHead>..HEAD --name-status --oneline`
- update with only a timestamp: `git log --since <updatedAt> --name-status --oneline`

If the target is not a git repo, it degrades gracefully to timestamps and source inspection.

**Parallel exploration.** For large repos, the agent fans out read-only subagents (Task tool),
each in its own context window, each with a narrow brief (existing docs, runtime architecture,
data/storage, API surface, integrations, tests, business workflows). Subagents only inspect and
summarize; the main thread synthesizes and does **all** writes. This is the substitute for
OpenWiki's context compaction — the main thread never loads the whole repo.

**Wiki structure.** `openwiki/quickstart.md` is the required entrypoint (overview + links to
every section). Section directories are created one per major domain, ≤ 8 pages on init, no stub
pages. Small repos get quickstart plus 1–2 pages.

**Wiring.** A short `## OpenWiki` reference section is added to top-level `AGENTS.md` / `CLAUDE.md`
(the wiki is never inlined). If neither exists, `AGENTS.md` is created with just that section.

**Idempotence.** The agent snapshots `openwiki/` content (excluding `.last-update.json`) with a
SHA-256 hash before and after the run:

```bash
find openwiki -type f -not -name .last-update.json -print0 | sort -z | xargs -0 sha256sum | sha256sum
```

If nothing changed → **no-op** (the wiki is already current). If it changed → write
`openwiki/.last-update.json`:

```json
{ "updatedAt": "<ISO>", "command": "init|update", "gitHead": "<HEAD>", "model": "<model>" }
```

`gitHead` is what the next `update` run diffs against — this file replaces OpenWiki's SQLite
checkpointer (durable crash-resume is intentionally dropped).

## Model tier

OpenWiki assumes a frontier coding model (default `z-ai/glm-5.2`, fallbacks
`openai/gpt-5.4-mini` / `anthropic/claude-sonnet-5`, provider list including Claude Opus 4.8 /
Sonnet 5 / GPT 5.5). **Run this command and its subagents on Opus 4.8** (Sonnet 5 minimum) for
comparable documentation quality.

## Headless / CI (`claude -p`)

To run non-interactively without permission prompts, grant a minimal allowlist in
`.claude/settings.json` (read-only git + snapshot tooling, writes scoped to `openwiki/`,
`CLAUDE.md`, `AGENTS.md`). The full snippet is documented inside
[`commands/wiki.md`](commands/wiki.md).

## Auto-run as a hook (keep the wiki fresh)

Claude Code hooks can invoke the command headlessly so the wiki refreshes itself after you work.
Wire a **`Stop`** hook (fires when Claude finishes a turn) — or **`SessionEnd`** (once, when the
session closes) — in `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "[ -n \"$OPENWIKI_HOOK\" ] || OPENWIKI_HOOK=1 claude -p '/wiki update' --permission-mode acceptEdits >/dev/null 2>&1 &"
          }
        ]
      }
    ]
  }
}
```

- **`OPENWIKI_HOOK` guard (required).** The headless `claude -p` run fires its own `Stop` hook;
  the env-var check makes the child skip re-triggering itself. Without it → infinite recursion.
- **Backgrounded (`&`).** Don't block your session on the doc run. On `SessionEnd`, wrap with
  `setsid` so it survives the closing session: `... setsid claude -p '/wiki update' ... &`.
- Use `/openwiki:wiki` instead of `/wiki` if you installed via the plugin marketplace.
- The idempotence no-op means an unchanged repo costs a cheap early exit — but each fired hook
  still spends tokens. **`Stop` runs every turn**; prefer `SessionEnd` (once per session) unless
  you want continuously-live docs, and mind the FinOps cost of frequent frontier-model runs.

## Fidelity to upstream

**Taken verbatim:** the full system prompt (`src/agent/prompt.ts`), the `## OpenWiki` section,
the git commands (`src/agent/utils.ts`), the `.last-update.json` shape, and the snapshot / no-op
logic (`src/agent/index.ts`).

**Adapted (and why):** DeepAgents virtual-filesystem tools and paths → native Read/Write/Edit/
Glob/Grep/Bash on real repo paths; the DeepAgents "task tool" → Claude Code subagents; the
in-process SHA-256 snapshot → a shell one-liner; provider/model plumbing → a model-tier
recommendation.

**Dropped as acceptable loss:** provider adapters, credential storage, the Ink TUI, OpenRouter
fallback, and LangSmith tracing (Claude Code transcripts cover debugging).

## Repository layout

```
.claude-plugin/
  plugin.json        # Claude Code plugin manifest
  marketplace.json   # single-plugin marketplace (source: ./)
commands/
  wiki.md            # Claude Code slash command (system prompt + git + idempotence)
.agents/skills/
  openwiki/SKILL.md  # Codex skill (same agent, adapted to Codex tools + context model)
README.md
```

The repo is both the Claude Code plugin and its marketplace, so
`/plugin marketplace add SoulKyu/openwiki-cc` exposes it directly. The Codex skill under
`.agents/skills/` is copied into `~/.agents/skills/` or a repo's `.agents/skills/`.

Both hosts carry the **same** verbatim OpenWiki system prompt, git commands, `.last-update.json`
shape and idempotence logic. They differ only where the hosts differ: Claude Code uses native
file tools + parallel read-only subagents for exploration; Codex uses its shell + `apply_patch`
and relies on its own native context compaction (no subagent tool), so its skill drops the
subagent section and generalizes the tool vocabulary.

## License

The upstream prompt and command semantics originate from
[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki).
