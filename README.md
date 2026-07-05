# openwiki-cc

A native **Claude Code** port of [OpenWiki](https://github.com/langchain-ai/openwiki) — an agent
that generates and maintains a documentation wiki (`openwiki/`) for any repository.

OpenWiki ships as a standalone CLI on its own harness (DeepAgents/LangGraph, a local shell
backend, a SQLite checkpointer, provider adapters, an Ink TUI). Claude Code already provides all
of that plumbing. This repo extracts **the agent itself** — the documentation system prompt, the
git-evidence collection, the wiki structure, and the idempotence logic — and re-expresses it as a
single native slash command, `commands/openwiki.md`, packaged as an installable Claude Code plugin.

The system prompt and the exact git commands are reproduced **verbatim from OpenWiki's real
source**, not from memory.

## What it does

Point it at a repository and it writes human- and agent-friendly Markdown documentation under
`openwiki/`, with `quickstart.md` as the entrypoint and thematic section pages (architecture,
workflows, operations, …). It grounds every claim in source files, existing docs, and git
history — and on later runs it updates only what actually changed.

## Install

### Via the plugin marketplace (recommended)

Inside Claude Code:

```
/plugin marketplace add SoulKyu/openwiki-cc
/plugin install openwiki@openwiki-cc
```

Then `/openwiki` is available in every project. `/plugin marketplace update openwiki-cc` pulls
new versions.

### Manual (no marketplace)

Copy the single command file into the repo you want to document, or globally:

```bash
# per-project
mkdir -p your-repo/.claude/commands
cp commands/openwiki.md your-repo/.claude/commands/

# or global
cp commands/openwiki.md ~/.claude/commands/
```

## Usage

Inside Claude Code, from the root of the target repository:

| Command | Behavior |
|---|---|
| `/openwiki` | **Auto-route**: if `openwiki/` exists → update, else → init. |
| `/openwiki init` | Build the wiki from scratch. |
| `/openwiki update` | Surgically refresh only the pages affected by recent changes. |
| `/openwiki update <instruction>` | Same, plus an extra instruction (e.g. `document the API routes first`). |

> **Note** — bare `/openwiki` auto-routes here, unlike upstream OpenWiki where the bare command
> is an interactive chat. This is a deliberate divergence: as a slash command, auto-route is more
> useful. `init` and `update` remain explicit.

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
[`commands/openwiki.md`](commands/openwiki.md).

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
  plugin.json        # plugin manifest
  marketplace.json   # single-plugin marketplace (source: ./)
commands/
  openwiki.md        # the slash command (system prompt + git + idempotence)
README.md
```

The repo is both the plugin and its marketplace, so `/plugin marketplace add SoulKyu/openwiki-cc`
exposes it directly.

## License

The upstream prompt and command semantics originate from
[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki).
