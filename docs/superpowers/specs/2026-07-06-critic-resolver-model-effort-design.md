# Per-role model & effort flags for Claude's critic/resolver turns

**Date:** 2026-07-06
**Status:** approved (design)

## Problem

Claude's own critic and resolver turns are generic Task subagents that inherit the
host session's model and reasoning effort. Peers already pin model/effort in their
roster `Command` (peers.md § Peer roster reminder box); the host's own turns have no
equivalent knob.

## Scope

- **In:** Claude's critic (§4a) and resolver/synthesis (§4d) subagent turns.
- **Out:** peers (keep pinning in roster commands), the finalizer (§5a — inherits
  host; add a `--finalizer-model` later if ever needed), interactive/config-file
  interfaces.

## Interface

Four new optional flags on `/idea-polish`, matching the existing flag style:

| Flag | Values | Unset behavior |
|------|--------|----------------|
| `--critic-model` | any Task-tool model name (`sonnet`, `opus`, `haiku`, `fable`) | inherit host (today's behavior) |
| `--critic-effort` | `low` \| `medium` \| `high` | no effort directive (today's behavior) |
| `--resolver-model` | same as `--critic-model` | inherit host |
| `--resolver-effort` | `low` \| `medium` \| `high` | no effort directive |

No model-name validation in prose — the value is passed through to the Task call,
which errors on an unknown model. Effort values other than `low|medium|high` stop
the run with `error: invalid effort '<value>'; expected low|medium|high`.

## Mechanism

The Task tool accepts a per-spawn `model` parameter but has **no per-spawn effort
parameter**, so:

- **Model:** pass the flag value as the Task call's `model` parameter when set.
- **Effort:** map the flag to a thinking-budget keyword line **prepended** to the
  seeded prompt (before the substituted role-prompt text):
  - `low` → no line
  - `medium` → `Think hard.`
  - `high` → `Ultrathink.`

  Caveat (stated in SKILL.md): this is a nudge to the model's thinking budget, not
  a hard setting — effort control is best-effort by construction.

## Edits

1. **`skills/idea-polish/SKILL.md`**
   - §1 Intake: add the four flags (defaults = inherit/none) and the effort-value
     validation error.
   - §4a Claude critic bullet: "…seed a generic subagent (via Task, with
     `model: <--critic-model>` when set; when `--critic-effort` is set, prepend the
     corresponding thinking directive to the prompt)."
   - §4d Synthesis bullet: same wording with the resolver flags. Applies to the
     drift-check corrective retry too (it re-runs §4d synthesis, so it uses the
     same model/effort).
2. **`skills/idea-polish/references/peers.md`** — reminder box: update "Claude's own
   turns run at the host session's model/effort (the generic subagents inherit it)"
   to note the new flags override that.
3. **README** — add the flags to its flag documentation if it has one (check at
   implementation time).

Prompt assets (`references/prompts/*.md`) are untouched; the effort directive is a
coordinator-prepended line, not a prompt-file slot.

## Divergence note

These flags are not in the Python `idea_polisher` reference (which has no
subagent-model concept). Deliberate divergence, same class as the ones listed in
CLAUDE.md — do not port back.

## Acceptance

Run `/idea-polish` on `examples/sample-idea.md` with
`--critic-model haiku --critic-effort high --resolver-model opus`: the §4a Task
calls show `model: haiku` and the seeded critic prompt starts with `Ultrathink.`;
the §4d Task calls show `model: opus` with no effort line; a flagless run is
byte-for-byte identical in behavior to today.
