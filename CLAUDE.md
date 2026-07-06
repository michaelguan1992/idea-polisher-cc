# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **plugin** (no build, no tests, no application code) that ships the
`/idea-polish` skill: a multi-round cross-model critic/resolver debate. Claude is
the host/owner; Codex (`codex exec`) and Antigravity (`agy --print`) join as peer
CLIs. It is the native-Claude-Code port of the `idea_polisher` Python CLI, which
remains the deterministic reference implementation — behavior changes here should
stay consistent with that reference.

Deliberate divergence (see
`docs/superpowers/specs/2026-07-06-problem-relevance-filter-design.md`): frozen
Problem, Problem/Approach terminology, autonomous approach-only gate, and the
off-problem relevance filter are **not** in the Python reference — do not port
them back silently, and do not "fix" this repo to match the reference on these
points.

**The prompts are the product.** Every change is a Markdown edit; there is nothing
to compile or lint. Verification is the acceptance check: run `/idea-polish` on
`examples/sample-idea.md` and read `final-idea.md` before/after.

## Architecture

- `skills/idea-polish/SKILL.md` — the **coordinator**. Runs in the main agent
  context and owns the loop (intake → charter capture → connection test → entry
  routing → critique/resolve rounds → finalize/summary). The loop must live here
  because subagents cannot spawn nested subagents.
- `skills/idea-polish/references/prompts/{critic,resolver,finalizer}.md` — the
  **single-source role prompts** with `{slot}` placeholders. Claude's own turns are
  generic subagents (Task tool) seeded from these; peers get the same critic prompt
  via `peers.md`. Never re-inline a role prompt elsewhere — reference these files.
- `skills/idea-polish/references/peers.md` — the **peer roster** (a table: args-only
  command + input mode), the connection test, peer prompt templates, and the
  security posture. Adding a model = adding a roster row that satisfies the contract
  in its § Adding a peer.
- `.claude-plugin/plugin.json` + `marketplace.json` — install/distribution manifests
  (marketplace `source` is `./` for local development).
- `docs/plans/` — dated implementation plans; new design work gets a plan file here
  (`YYYY-MM-DD-NNN-<slug>-plan.md`).

## Reliability-critical contracts (don't break these)

The loop is prose-orchestrated, so these text contracts are what makes it converge
and stay safe. They are defined verbatim in `SKILL.md` § Definitions and mirrored
in the prompt assets — change them in lockstep or not at all:

- **Verdict**: JSON after the last `---VERDICT-JSON---` line;
  `charter_threats` is optional (absent ⇒ `[]`) for backward compatibility with
  older peer prompts.
- **Disposition**: resolver output splits on `---DISPOSITION---`.
- **Convergence quorum**: ≥1 parsed verdict AND every parsed verdict is
  `constructive: false` with no clarifications and no charter threats; parse
  failures are excluded from the quorum.
- **Approach gate (autonomous)**: the debate loop is zero-interaction after the
  pre-loop charter confirmation. The charter's `## Problem` is frozen (every
  revision echoes it verbatim; the coordinator string-compares); approach threats
  are always resolver-defended (`defended-resolver`) and recorded in `summary.md`
  § Charter & shifts for the user to act on between runs.
- **Off-problem filter**: critiques/revisions that don't serve the frozen
  `## Problem` are discarded in-round at three layers (critic, resolver,
  coordinator) and logged in `summary.md` § Off-problem discards.
- **Output split**: `final-idea.md` is the standalone deliverable; `summary.md` is
  the process record and must **never** contain the idea body — pointers by
  filename only. SKILL.md ends with a self-check enforcing this.

## Security posture (preserve when editing peers.md or SKILL.md)

- Roster commands are **args only**; the coordinator appends the prompt via a file
  (`"$(cat .peer-prompt.txt)"` or stdin) — the prompt/idea text is never
  shell-interpolated and never lives in a roster row.
- Peer output is untrusted data, wrapped in `---PEER-OUTPUT-START/END---` blocks.
- Peers run with the run folder (`runs/<ts>/`) as cwd, but cwd is not a sandbox;
  auto-approval flags like `--dangerously-skip-permissions` are opt-in only and
  the shipped roster must not include them by default.
