---
name: idea-polish
description: Polish an idea via a multi-round cross-model critic/resolver debate (Claude + Codex + Antigravity). Use when the user wants an idea hardened by adversarial cross-model review.
---

# Idea Polish (coordinator)

> **Scaffold — implement per `docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md` (U4).**
> This skill is the orchestrator and must run in the main agent context (it spawns
> the `idea-critic` / `idea-resolver` subagents via Task and calls peer CLIs via
> Bash — subagents cannot nest, so the loop lives here, not in a subagent).

Carry the loop verbatim from the `idea_polisher` reference implementation
(`idea_polish.py`: `run_polish`, `critique_phase`, `resolve_phase`, `is_converged`,
`classify_entry`) and the prompt constants. Numbered phases to implement:

1. **Intake** — idea (arg / file / ask) and K rounds; owner defaults to Claude; honor a resolve-first override.
2. **Connection test** — Claude is the host (always present); probe `codex` / `agy` per `references/peers.md`; degrade gracefully (owner required, peers best-effort).
3. **Entry routing** — classify whether the idea already contains critiques → resolve-first vs critique-first (default critique-first on failure).
4. **Loop** (≤ K rounds): spawn `idea-critic` for Claude + Bash `codex`/`agy` critiques → aggregate verdicts → apply the convergence quorum (≥1 parsed critic, all parsed non-constructive; parse-failures logged and excluded) → handle clarifications → gather peer fix-proposals via Bash → spawn `idea-resolver` to synthesize → snapshot `idea-vN` → stop on convergence / K / resolve-failure.
5. **Output** — write `summary.md` (polished idea, per-round digest with dispositions, models, stop reason).

See `references/peers.md` for the exact peer commands and security posture.
