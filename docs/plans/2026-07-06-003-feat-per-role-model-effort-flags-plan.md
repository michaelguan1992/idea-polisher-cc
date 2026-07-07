# Per-Role Model & Effort Flags Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `--critic-model` / `--critic-effort` / `--resolver-model` / `--resolver-effort` flags so Claude's own critic and resolver subagent turns no longer have to inherit the host session's model/effort.

**Architecture:** This repo is a prose-orchestrated Claude Code plugin — every change is a Markdown edit, no build, no tests. The flags are defined in `SKILL.md` §1 Intake; §4a and §4d wire them into the Task-spawn instructions (model → Task `model` parameter; effort → thinking directive prepended to the seeded prompt). Peers and the finalizer are untouched.

**Tech Stack:** Markdown only. Verification = grep checks + reading the edited sections; full acceptance is a manual `/idea-polish` run (spec § Acceptance).

**Spec:** `docs/superpowers/specs/2026-07-06-critic-resolver-model-effort-design.md`

## Global Constraints

- Effort values are exactly `low` | `medium` | `high`; anything else stops the run with `error: invalid effort '<value>'; expected low|medium|high` (verbatim from spec).
- Effort → directive mapping (verbatim): `low` → no line, `medium` → `Think hard.`, `high` → `Ultrathink.` — prepended as the first line of the seeded prompt, before the substituted role-prompt text.
- Model values pass through to the Task call's `model` parameter unvalidated (the Task call errors on unknown models). Unset flags ⇒ today's behavior exactly (inherit host, no directive).
- Prompt assets `skills/idea-polish/references/prompts/*.md` must NOT be edited — the directive is coordinator-prepended, not a prompt slot.
- The §4d drift-check corrective retry re-runs §4d synthesis and therefore uses the same resolver model/effort — make this explicit.
- Finalizer (§5a) stays on host inherit — no flag.
- This is deliberate divergence from the Python `idea_polisher` reference (spec § Divergence note); do not "fix" it to match.

---

### Task 1: SKILL.md — intake flags + §4a/§4d wiring

**Files:**
- Modify: `skills/idea-polish/SKILL.md` (§1 Intake bullet list ~line 87; §4a Claude bullet ~line 177; §4d Synthesis bullet ~line 252)

**Interfaces:**
- Produces: the flag names, effort values, error string, and directive mapping that Task 2's doc edits reference. Exact flag names: `--critic-model`, `--critic-effort`, `--resolver-model`, `--resolver-effort`.

- [ ] **Step 1: Add the intake bullet**

In `skills/idea-polish/SKILL.md` §1, after the Timeout bullet, Edit:

```
old_string:
- **Timeout:** per peer call, default **120s** (`--timeout`).

new_string:
- **Timeout:** per peer call, default **120s** (`--timeout`).
- **Claude turn model/effort:** `--critic-model M` / `--critic-effort E` for
  Claude's own critic turns (§4a), `--resolver-model M` / `--resolver-effort E`
  for its resolver turns (§4d) — all optional. `M` is passed through as the Task
  call's `model` parameter, unvalidated (an unknown model errors at the Task
  call); unset ⇒ inherit the host session, exactly the prior behavior. `E` is
  `low` | `medium` | `high`, mapped to a thinking directive prepended as the
  **first line** of the seeded prompt (before the substituted role-prompt text):
  `low` → no line, `medium` → `Think hard.`, `high` → `Ultrathink.` — a
  thinking-budget nudge, not a hard setting. Any other effort value stops the
  run with `error: invalid effort '<value>'; expected low|medium|high`. Peers
  are unaffected (they pin model/effort in the roster `Command` —
  `references/peers.md`), and the finalizer (§5a) always inherits the host.
```

- [ ] **Step 2: Wire §4a (Claude critic)**

Edit:

```
old_string:
- **Claude:** read `references/prompts/critic.md`, substitute its slots (`{idea}` with
  the current idea fenced in triple quotes, `{charter}` with the charter §1a fenced,
  `{context}` with the frozen context block §1a fenced), and seed a generic subagent
  (via Task) with the result. Its final message is the verdict block.

new_string:
- **Claude:** read `references/prompts/critic.md`, substitute its slots (`{idea}` with
  the current idea fenced in triple quotes, `{charter}` with the charter §1a fenced,
  `{context}` with the frozen context block §1a fenced), and seed a generic subagent
  (via Task, passing `--critic-model` as the `model` parameter when set) with the
  result — when `--critic-effort` is set, prepend its thinking directive (§1) as
  the first line of the seeded prompt. Its final message is the verdict block.
```

- [ ] **Step 3: Wire §4d (synthesis + corrective retry)**

Edit:

```
old_string:
  with the current idea, `{charter}` §1a fenced, `{context}` §1a fenced, `{critiques}` with the critique list
  from 4b, `{peer_proposals}` with the wrapped peer proposals, `{defense_directive}`
  with the round's directive from §4b′), and seed a generic subagent (via Task) with the
  result. It returns the full revised idea + `---DISPOSITION---` + per-critique
  disposition. Split on `---DISPOSITION---`.

new_string:
  with the current idea, `{charter}` §1a fenced, `{context}` §1a fenced, `{critiques}` with the critique list
  from 4b, `{peer_proposals}` with the wrapped peer proposals, `{defense_directive}`
  with the round's directive from §4b′), and seed a generic subagent (via Task,
  passing `--resolver-model` as the `model` parameter when set) with the result —
  when `--resolver-effort` is set, prepend its thinking directive (§1) as the
  first line of the seeded prompt. The drift-check corrective retry below re-runs
  this synthesis and uses the same model/effort. It returns the full revised idea
  + `---DISPOSITION---` + per-critique disposition. Split on `---DISPOSITION---`.
```

- [ ] **Step 4: Verify**

Run: `grep -c -- "--critic-model\|--critic-effort\|--resolver-model\|--resolver-effort" skills/idea-polish/SKILL.md`
Expected: `6` or more (intake bullet has all four; §4a has two; §4d has two — grep -c counts lines, so ≥6 matching lines). Also run `grep -n "Ultrathink" skills/idea-polish/SKILL.md` — expected: exactly 1 hit, in the §1 intake bullet. Confirm `git diff` touches nothing under `references/prompts/`.

- [ ] **Step 5: Commit**

```bash
git add skills/idea-polish/SKILL.md
git commit -m "feat(idea-polish): per-role model/effort flags for Claude critic & resolver turns"
```

---

### Task 2: Doc mirrors — peers.md reminder box + README flag list

**Files:**
- Modify: `skills/idea-polish/references/peers.md` (reminder blockquote, ~line 26)
- Modify: `README.md` (flag bullet list, ~line 50, and example block ~line 47)

**Interfaces:**
- Consumes: the four flag names and effort values from Task 1 (`--critic-model`, `--critic-effort`, `--resolver-model`, `--resolver-effort`; effort `low|medium|high`).

- [ ] **Step 1: Update the peers.md reminder box**

Edit `skills/idea-polish/references/peers.md`:

```
old_string:
> model_reasoning_effort=high`, `agy --model <model>`). Claude's own turns run at the
> host session's model/effort (the generic subagents inherit it). A weak or
> low-effort setting silently degrades the critique quality.

new_string:
> model_reasoning_effort=high`, `agy --model <model>`). Claude's own turns run at the
> host session's model/effort (the generic subagents inherit it) unless overridden
> per run with `--critic-model` / `--critic-effort` / `--resolver-model` /
> `--resolver-effort` (`SKILL.md` §1). A weak or
> low-effort setting silently degrades the critique quality.
```

- [ ] **Step 2: Add the README flag bullets and example**

Edit `README.md` — add an example line:

```
old_string:
/idea-polish --file idea.md --auto
```

new_string:
/idea-polish --file idea.md --auto
/idea-polish "<your idea>" --critic-model haiku --critic-effort high --resolver-model opus
```

Then add flag bullets:

```
old_string:
- `--auto` — non-interactive: never asks a human (errors if no idea is given;
  charter derived unconfirmed). For agents and scheduled routines.

new_string:
- `--auto` — non-interactive: never asks a human (errors if no idea is given;
  charter derived unconfirmed). For agents and scheduled routines.
- `--critic-model M` / `--critic-effort low|medium|high` — model and reasoning
  effort for Claude's **own** critic turns (default: inherit the session). Peers
  pin theirs in the roster instead (`references/peers.md`).
- `--resolver-model M` / `--resolver-effort low|medium|high` — same, for Claude's
  own resolver turns.
```

- [ ] **Step 3: Verify**

Run: `grep -n -- "--critic-model" skills/idea-polish/references/peers.md README.md`
Expected: ≥1 hit in each file.

- [ ] **Step 4: Commit**

```bash
git add skills/idea-polish/references/peers.md README.md
git commit -m "docs(idea-polish): document per-role model/effort flags"
```

---

### Task 3: Acceptance check (manual, optional now)

Per the spec § Acceptance — run when convenient, not a blocker for the merge of the prose edits:

- [ ] Run `/idea-polish --file examples/sample-idea.md --rounds 1 --critic-model haiku --critic-effort high --resolver-model opus` and observe: §4a Task calls use `model: haiku` with the seeded prompt starting `Ultrathink.`; §4d Task calls use `model: opus` with no directive line.
- [ ] Run `/idea-polish --file examples/sample-idea.md --rounds 1` (flagless) and confirm behavior is unchanged from before (no `model` parameter, no directive line).
- [ ] Run with `--critic-effort extreme` and confirm the run stops with `error: invalid effort 'extreme'; expected low|medium|high`.
