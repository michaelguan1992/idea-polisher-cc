# Multi-Harness Parity + Quality Hygiene — Requirements

**Date:** 2026-06-29
**Status:** Ready for planning (`/ce-plan`)
**Scope:** Deep — product (orchestration shape changes)

## Summary

Bring `idea-polisher-cc` to the *quality* standard of the
[everyinc compound-engineering-plugin](https://github.com/everyinc/compound-engineering-plugin),
without copying its scale. Two tracks:

1. **Hygiene** — the handful of standards files and a validation check the reference
   has and we don't.
2. **Multi-harness** — generalize the orchestration from "Claude is always host/owner"
   to a symmetric tri-host model where **Codex, Antigravity (agy), or Claude** can each
   host. The host owns and resolves the idea; the other two run as peer critics.

## Context

- The reference plugin's apparent "standard" is mostly **scale and multi-harness
  architecture** (26 skills, 7 harnesses, a TS build), not quality we lack. Most of its
  footprint is a different, larger product and is deliberately *not* a target here.
- Current state (verified): root has only `README.md`, `LICENSE`, `.gitignore`,
  `.claude-plugin/`. Single skill `idea-polish`. Three Claude Task-tool agents
  (`agents/idea-critic.md`, `idea-resolver.md`, `idea-finalizer.md`).
- `skills/idea-polish/references/peers.md` hard-codes Claude as host/owner excluded
  from the roster, with `codex` + `agy` as the only peers.

## Decisions made (in dialogue)

- **Targets:** Codex and Antigravity (agy) as additional hosts. (Cursor, Kimi,
  opencode, pi explicitly deferred.) Both CLIs are already invoked as peers, so the
  inversion is natural.
- **Owner role:** **host always owns/resolves.** Fully symmetric — no privileged
  model. When Claude hosts, behavior is unchanged from today.
- **No TS build / `src/`.** The plugin is pure markdown; a build pipeline would be
  pure ceremony.

## Goals

- Close the quality-hygiene gap with ~5 small files + one CI-run validation check.
- Make the idea-polish loop runnable with any of {Claude, Codex, agy} as host, the
  other two as peers, with no behavioral regression when Claude hosts.
- Keep the skill source single-sourced; per-harness directories stay thin.

## Non-goals

- Cursor / Kimi / opencode / pi harnesses.
- TypeScript `src/`, `package.json`, build tooling, `assets/`/favicon.
- Expanding the skill set beyond `idea-polish`.

## Requirements

### Track 1 — Hygiene

| File / item | Why it matters here |
|---|---|
| `CONCEPTS.md` | Highest-value cheap add. Glossary for the project's real vocabulary: *charter / charter threat, convergence quorum, verdict delimiter, disposition, owner vs peer, host*. Tooling reads it. |
| `PRIVACY.md` | The plugin sends the user's idea text to external models via `codex`/`agy` (and now `claude -p`). A real disclosure surface — short and specific, not boilerplate. |
| `SECURITY.md` | Plugin executes external subprocesses with opt-in `--dangerously-skip-permissions`. Reference `peers.md` already documents the posture; surface it at repo root. |
| `CHANGELOG.md` | At `0.1.0` with 5 plan docs already. Cheap, conventional. |
| Validation check | The loop is prose-orchestrated and reliability-critical. One check asserts: both manifests parse; the verdict-JSON contract example parses against its documented shape. No framework — a single script. |
| `.github/` CI | A workflow that runs the validation check on push/PR. |

### Track 2 — Multi-harness orchestration

- **R1 — Host-excluded roster.** `peers.md` changes from "Claude never appears here"
  to "**the host** never appears here; the other two reachable models do." Add a
  `claude` peer row (`claude -p` / `--print`, `Input: arg`) used when Claude is not
  host. Pin model/effort per the existing roster reminder.
- **R2 — Symmetric owner.** The host owns/resolves via its **own native turn**; the
  other two critique as peers. Claude-as-host path is byte-for-byte current behavior.
- **R3 — Shared prompt assets, host-native spawning.** The critic / resolver /
  proposal prompts already live verbatim in `peers.md`. The orchestrator uses the
  **host's** subagent/native primitive (Task in Claude, `spawn_agent` in Codex, agy's
  agent/turn) with those shared prompts — instead of three Claude-Task-specific agent
  files. (See Open Question Q1 on what becomes of `agents/*.md`.)
- **R4 — Flag universe generalizes.** `--peers/--with/--without` now range over
  {claude, codex, agy} **minus the host**. The "not a valid flag value" rule inverts:
  whoever is *host* is rejected as a flag value (not always `claude`). With no flags,
  the run is host + the other two default-on peers.
- **R5 — Thin per-harness manifests.** Add `.codex-plugin/` and an agy plugin dir
  (`.agy` per the reference) so the skill is installable/hostable there, pointing at
  the single shared skill source. Add a generic `AGENTS.md` context file if the
  Codex/agy hosts need it (reference uses `AGENTS.md` as the cross-harness one).
- **R6 — Security posture carries over unchanged.** File-mediated prompt passing
  (`"$(cat .peer-prompt.txt)"`), `---PEER-OUTPUT-START/END---` wrapping, run-folder
  cwd, connection test. These are already harness-neutral. The `claude -p` peer is
  subject to the same command-injection rule as any peer.

## Open questions

- **Q1 — Fate of `agents/*.md`.** On Codex/agy there is no Task-tool agent file.
  Options for planning: (a) keep them as the Claude-host implementation of the shared
  prompts and have non-Claude hosts inline the `peers.md` prompts; (b) collapse all
  three into skill-local prompt assets referenced by every host. Lean (b) for single-
  sourcing, but it's an implementation call for `/ce-plan`.
- **Q2 — agy plugin dir name.** Confirm the reference's exact agy manifest layout
  (`.agy` vs `.agents/plugins`) before creating it.
- **Q3 — Does Codex/agy expose a "native owner turn"** equivalent to Claude running
  resolution natively, or must the host model also be invoked via its own CLI for the
  owner step? Affects R2's "own native turn" wording per harness.

## Dependencies / Assumptions

- Assumes `claude`, `codex`, `agy` CLIs are each installable as a *peer* (prompt-in /
  text-out, exit 0) — already true for codex/agy; `claude -p` assumed to satisfy the
  same contract (verify in planning).
- Assumes no behavioral change is acceptable only on the Claude-host path; non-Claude
  hosts are new surface and may legitimately differ in orchestration mechanics.
- The verdict-JSON contract (`---VERDICT-JSON---` + shape) is the cross-harness
  reliability anchor and must remain identical across all three hosts.

## Handoff

Run `/ce-plan` against this doc. Suggest planning the two tracks separately —
**Hygiene is independent and shippable first** (no orchestration risk); multi-harness
is the larger, sequenced track (R1→R4 are the behavioral core, R5 the packaging,
Q1–Q3 to resolve during planning).
