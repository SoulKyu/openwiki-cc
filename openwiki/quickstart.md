# openwiki-cc — quickstart

**openwiki-cc** is a native **Claude Code** and **OpenAI Codex** port of
[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki): an agent that generates and
maintains a documentation wiki (an `openwiki/` directory) for *any* repository — the same wiki
you are reading now was produced by running it against this repo.

Upstream OpenWiki ships as a standalone CLI carrying its own harness (DeepAgents/LangGraph, a
shell backend, a SQLite checkpointer, provider adapters, an Ink TUI). Coding agents already
provide all of that plumbing. This repo extracts **only the agent** — the documentation system
prompt, the git-evidence collection, the wiki structure rules, and the idempotence logic — and
re-expresses it natively for each host. There is no application server, no build, no runtime; the
deliverable is the agent definition itself, expressed as prompt files.

## What lives here

| Path | Role |
|---|---|
| [`commands/wiki.md`](../commands/wiki.md) | The Claude Code slash command → `/openwiki:wiki`. Contains the full routing, the four run steps, and the verbatim upstream system prompt. **This is the canonical agent definition.** |
| [`.agents/skills/openwiki/SKILL.md`](../.agents/skills/openwiki/SKILL.md) | The Codex port of the same agent → `$openwiki`. Same system prompt, adapted for Codex (no subagents; relies on Codex's native context compaction). |
| [`.claude-plugin/plugin.json`](../.claude-plugin/plugin.json), [`marketplace.json`](../.claude-plugin/marketplace.json) | Packaging so Claude Code can install the command as a plugin from a marketplace. |
| [`hooks/openwiki-gate.sh`](../hooks/openwiki-gate.sh) | Optional shell gate for auto-running the wiki from a Claude Code `Stop`/`SessionEnd` hook — spawns the frontier model only when source actually changed. |
| [`hooks/test_gate.sh`](../hooks/test_gate.sh) | Self-check for the gate's skip/run decisions. |
| [`README.md`](../README.md) | Human-facing install + usage guide. |

The single source of truth for the agent's behavior is `commands/wiki.md`; the Codex `SKILL.md`
tracks it with host-specific adaptations. When they disagree, `commands/wiki.md` is authoritative.

## Install & run

**Claude Code (plugin):**
```
/plugin marketplace add SoulKyu/openwiki-cc
/plugin install openwiki@openwiki-cc
```
Then from the root of a target repo: `/openwiki:wiki` (auto-routes), `/openwiki:wiki init`, or
`/openwiki:wiki update`. Plugin commands are always namespaced, so it is `/openwiki:wiki`, never a
bare `/openwiki`. Copying `commands/wiki.md` into `.claude/commands/` instead gives a bare `/wiki`.

**Codex (skill):** copy `.agents/skills/openwiki/` into `~/.agents/skills/` (or a repo's
`.agents/skills/`), restart Codex, and invoke `$openwiki` or ask to "update the openwiki docs".

Full install variants (per-project vs global, both hosts) are in the [README](../README.md).

## The three commands

| Invocation | Behavior |
|---|---|
| `/openwiki:wiki` | **Auto-route**: `openwiki/` exists → update, else → init. |
| `/openwiki:wiki init` | Build the wiki from scratch (≤ 8 pages; 1–2 for a small repo). |
| `/openwiki:wiki update` | Surgically refresh only pages affected by changes since the last run. |
| `/openwiki:wiki update <instruction>` | Same, plus an extra instruction appended to the run. |

`init` vs `update` is the core distinction: **init** builds structure from scratch; **update** is
deliberately conservative — it diffs against the last run and edits only what the changes touched,
and it can legitimately be a **no-op** when nothing relevant changed. See
[architecture.md](architecture.md) for how a run actually executes.

## Model tier — do not run this small

Upstream OpenWiki assumes a **frontier coding model** (default `z-ai/glm-5.2`; provider list
includes Claude Opus 4.8 / Sonnet 5 / GPT 5.5). Run the command **and its subagents on Opus 4.8**
(Sonnet 5 minimum) on Claude Code, or Codex's strongest tier with high reasoning effort.
Documentation quality depends directly on the model — a small/fast tier produces a shallow wiki.

## Where to go next

- **[architecture.md](architecture.md)** — how a run executes end-to-end: the two host ports, the
  routing + four-step lifecycle (git evidence → snapshot → system prompt → metadata), idempotence,
  parallel subagents, root-file wiring, and the hook-based auto-run with its shell gate.
- **[Fidelity to upstream](../README.md#fidelity-to-upstream)** — what is verbatim from OpenWiki's
  source vs adapted for these harnesses, and why.

## Changing this repo

- **Behavior of the agent** → edit [`commands/wiki.md`](../commands/wiki.md), then mirror any
  semantic change into [`SKILL.md`](../.agents/skills/openwiki/SKILL.md). Keep the reproduced
  system prompt faithful to upstream; mark harness adaptations explicitly (upstream marks them
  `[adapted]`).
- **Packaging / version** → [`plugin.json`](../.claude-plugin/plugin.json) (bump `version`) and
  [`marketplace.json`](../.claude-plugin/marketplace.json).
- **Auto-run gate** → [`hooks/openwiki-gate.sh`](../hooks/openwiki-gate.sh); run
  `sh hooks/test_gate.sh` after any change to it (it exercises every skip/run branch).
