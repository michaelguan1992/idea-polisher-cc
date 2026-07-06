# Design: `--auto` mode — agent & routine invocation for /idea-polish

**Date:** 2026-07-06
**Status:** approved design, pre-implementation

## Goal

Make `/idea-polish` invocable by other agents and scheduled Claude routines with
zero human interaction. Today the loop is already zero-interaction after charter
confirmation; what's missing is a defined trigger for "non-interactive run" and a
documented calling contract.

## Decisions (user-confirmed)

- **Invocation shape:** headless `claude -p "/idea-polish ... --auto"`. The
  coordinator must run as the main agent (subagents cannot spawn the nested Task
  subagents it needs), so callers shell out — they do not Task-spawn the skill.
- **No discernible Approach in the seed (auto mode):** derive best-effort, mark
  the charter `unconfirmed (derived)`, log prominently in `summary.md`. Never
  fail, never ask.

## Changes (all Markdown; SKILL.md + README)

### 1. `--auto` flag (SKILL.md §1 Intake)

- New flag defining non-interactive mode. No heuristic/headless detection —
  explicit flag only.
- Under `--auto`, a missing idea (no argument, no `--file`) stops the run with
  `error: --auto requires an idea argument or --file` instead of asking.
- Everywhere SKILL.md already says "non-interactive run" (§1a charter
  confirmation skip, §4b′ autonomous gate, §4c unasked clarifications), the
  trigger is now concrete: `--auto` present ⇒ non-interactive.

### 2. Charter capture in auto mode (SKILL.md §1a)

- Seed with no discernible Approach: derive the most plausible one, mark the
  charter `unconfirmed (derived)`, and record in `summary.md` § Charter & shifts
  that both derivation and confirmation were unattended.

### 3. New section: "Agent & routine invocation" (SKILL.md + README note)

- Command shape:
  `claude -p "/idea-polish --file idea.md --auto [--rounds N] [--peers ...]"`,
  run from the directory where `runs/` should land.
- Rationale: main-agent requirement (nested subagents) ⇒ shell out, don't
  Task-spawn.
- Output contract for callers: the final message names
  `runs/<ts>/final-idea.md` and `runs/<ts>/summary.md` and prints the polished
  idea — consume stdout or read the files.
- Routine wiring example (documentation only): a scheduled routine whose prompt
  runs `claude -p '/idea-polish --file <f> --auto'` for each file in
  `ideas/inbox/`, moving processed files to `ideas/done/`.

## Out of scope (deliberate)

- Auto-detection of headless mode; JSON output format; MCP wrapper; a shipped
  inbox-watcher routine/command. Revisit the shipped-routine idea once a real
  routine consumes this.

## Acceptance check

`claude -p "/idea-polish --file examples/sample-idea.md --auto --rounds 2"`
completes with no prompt to a human, writes `runs/<ts>/{final-idea.md,summary.md}`,
and the charter in `summary.md` is marked `unconfirmed`.
