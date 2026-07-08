# Architecture — how a run executes

openwiki-cc has no runtime of its own. Its "architecture" is the **execution contract** encoded
in the agent definition: what the host model must do, in what order, before and after it writes
documentation. This page describes that contract and how it is packaged for two hosts.

Start at [quickstart.md](quickstart.md) if you haven't; this page is the deep dive.

## Two host ports, one agent

The same OpenWiki agent is expressed twice, because the two hosts have different capabilities:

| | Claude Code — [`commands/wiki.md`](../commands/wiki.md) | Codex — [`.agents/skills/openwiki/SKILL.md`](../.agents/skills/openwiki/SKILL.md) |
|---|---|---|
| Trigger | `/openwiki:wiki [init\|update] [instruction]` | `$openwiki` (or natural-language "update the openwiki docs") |
| Mode input | explicit `init`/`update` token, else auto-route | phrasing, else `openwiki/` auto-detect (no slash args) |
| Big-repo strategy | fans out read-only **subagents** (Task tool), each own context window | relies on Codex's **native context compaction** (no subagent tool) |
| Filesystem | native Read/Write/Edit/Glob/Grep/Bash on real repo paths | same, Codex-native |

`commands/wiki.md` is authoritative. Both reproduce OpenWiki's system prompt **verbatim from
upstream source** (not from memory), with only two harness adaptations, marked `[adapted]` inline:
(a) DeepAgents' virtual filesystem → the host's native file tools on real paths; (b) DeepAgents'
"task tool" → Claude Code subagents (Codex drops this entirely).

## Mode routing

The mode is resolved before any work:

- `init` → build from scratch (assume `openwiki/` has nothing useful).
- `update` → maintenance pass (inspect existing docs, edit surgically).
- **empty** → auto-route: `test -d openwiki && echo update || echo init`.
- A leading `init`/`update` token selects the mode; any remaining text becomes an **additional
  user instruction** appended to the run. With no mode token, auto-route and treat the whole
  argument string as the instruction (upstream OpenWiki's `[message]`).

Auto-route replaces upstream's interactive-chat default — a plugin slash command is always
namespaced and can't be a bare conversational `/openwiki`.

## The run lifecycle (Claude Code, four steps)

`commands/wiki.md` drives the model through a fixed sequence. `SKILL.md` mirrors it for Codex.

**Step 0 — pre-run no-op check** *(update mode only, and only when no extra instruction was
given)*. Mirrors upstream `getUpdateNoopStatus` / `shouldCheckUpdateNoop`: skip the entire run
when nothing relevant changed. Read `openwiki/.last-update.json`; if it has a `gitHead`, the run
is skipped when **both** hold:
- `git status --short` is empty after ignoring any line for `openwiki/.last-update.json`; **and**
- `HEAD == gitHead`, **or** every path in `gitHead..HEAD` is under `openwiki/`.

With no recorded `gitHead`, this check is skipped and the run proceeds.

**Step 1 — collect git evidence** (before any write; all `git --no-pager`, all read-only). Always
runs `status --short`, `rev-parse HEAD`, `diff --name-status HEAD`. History depends on mode:
init (or update with no metadata) → `log --max-count=20`; update with a `gitHead` →
`log gitHead..HEAD`; update with only an `updatedAt` → `log --since <updatedAt>`. Not a git repo →
degrade to filesystem timestamps + source inspection. The assembled output is the "git evidence"
block fed to the system prompt.

**Step 2 — snapshot** the current wiki content for idempotence:
```bash
find openwiki -type f -not -name .last-update.json -print0 | sort -z | xargs -0 sha256sum | sha256sum
```

**Step 3 — act as the agent.** The model runs OpenWiki's system prompt against the evidence:
inventory the repo (tree, config, entrypoints, representative files per domain — never
`glob **/*` from root, never read every file), write a temporary `openwiki/_plan.md`, then write
`quickstart.md` + section pages. Discipline baked into the prompt: ≤ 8 pages on init (1–2 for a
small repo), no stub pages, no single-file directories unless the boundary is real, each concept
gets one canonical home. Update mode is surgical — a soft diff budget (< ~5 files changed →
≤ 1–2 pages edited), no formatting-only churn, and no-op allowed. `_plan.md` is deleted before the
run ends.

**Step 4 — persist metadata.** Recompute the Step 2 hash. If **unchanged** → no-op, do not write
metadata, report the wiki is already current. If **changed** → write `openwiki/.last-update.json`:
```json
{ "updatedAt": "<ISO 8601>", "command": "init|update", "gitHead": "<git rev-parse HEAD>", "model": "<model id>" }
```

## Idempotence & state

Two independent mechanisms keep re-runs cheap and honest:

- **Content hash (Steps 2/4)** — the SHA-256 of wiki content decides whether a run *changed*
  anything. Nothing changed → `.last-update.json` is not even rewritten.
- **`gitHead` in `.last-update.json`** — the commit the *next* `update` diffs against. This single
  file replaces upstream OpenWiki's SQLite checkpointer; durable crash-resume is intentionally
  dropped as unnecessary on these hosts.

## Root agent-file wiring

Unless told otherwise, a run ensures the repo's **top-level** `/AGENTS.md` and/or `/CLAUDE.md`
carries a short `## OpenWiki` reference section pointing at `openwiki/quickstart.md` (the wiki is
never inlined into them). If neither file exists, `/AGENTS.md` is created with only that section.
Only top-level files are touched — never nested ones — and only the OpenWiki section, never
unrelated formatting.

## Auto-run via a Stop/SessionEnd hook

To keep the wiki fresh automatically, a Claude Code hook can invoke the command headlessly
(`claude -p '/openwiki:wiki update' --permission-mode acceptEdits`). The naive version spawns a
full frontier-model run **every turn** — even when nothing changed, it pays the model cost just to
let Step 0 discover there's no work.

[`hooks/openwiki-gate.sh`](../hooks/openwiki-gate.sh) fixes that by reproducing the Step 0 no-op
check in **pure shell** (a few `git` commands, zero tokens) and spawning `claude` **only** when
source actually changed since the recorded `gitHead`. It also carries the `OPENWIKI_HOOK=1` guard
(the headless run fires its own `Stop` hook → the gate exits early to avoid infinite recursion)
and detaches with `setsid` so it never blocks the session. Wired in `.claude/settings.json`:
```json
{ "hooks": { "Stop": [ { "hooks": [ { "type": "command", "command": "sh .claude/hooks/openwiki-gate.sh" } ] } ] } }
```
With the gate, `Stop` (every turn) is cheap enough for continuously-live docs; the frontier model
starts only on a real change. [`hooks/test_gate.sh`](../hooks/test_gate.sh) stubs `claude` and
exercises every skip/run branch — run it after touching the gate.

## Headless / CI permissions

For non-interactive runs (`claude -p`) without permission prompts, grant a minimal allowlist in
`.claude/settings.json`: read-only git + `find`/`sha256sum`/`rg`/`date`, with `Write`/`Edit`
scoped to `openwiki/**`, `CLAUDE.md`, `AGENTS.md` only. The exact snippet lives at the bottom of
[`commands/wiki.md`](../commands/wiki.md) — it is the OpenWiki `ShellAllowList` expressed as
Claude Code permissions.
