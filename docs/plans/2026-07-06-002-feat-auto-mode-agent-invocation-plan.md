# `--auto` Mode — Agent & Routine Invocation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an explicit `--auto` flag that makes `/idea-polish` fully non-interactive, and document the headless calling contract for agents and scheduled routines.

**Architecture:** This repo is a prose-orchestrated Claude Code plugin — every change is a Markdown edit to `skills/idea-polish/SKILL.md` and `README.md`. No code, no build, no test framework; verification is grep checks plus the repo's acceptance check (a headless run).

**Tech Stack:** Markdown only.

**Spec:** `docs/plans/2026-07-06-001-feat-auto-mode-agent-invocation-design.md` (approved).

## Global Constraints

- All edits are Markdown; nothing to compile or lint.
- Do not touch the reliability-critical contracts in SKILL.md § Definitions (verdict, disposition, quorum, charter gate, output split) — this feature only concretizes the already-specified "non-interactive run" trigger.
- Do not add auto-detection of headless mode, JSON output, an MCP wrapper, or a shipped routine/command — spec's "Out of scope" list.
- The exact error string is: `error: --auto requires an idea argument or --file`.
- The exact charter marker for an unattended-derived Approach is: `unconfirmed (derived)`.

---

### Task 1: `--auto` flag and non-interactive behavior in SKILL.md

**Files:**
- Modify: `skills/idea-polish/SKILL.md` (§1 Intake ~line 58, §1a ~line 104, §5b ~line 301)

**Interfaces:**
- Produces: the flag name `--auto`, the error string, and the `unconfirmed (derived)` marker — Task 2's new section and Task 3's README bullets must use these exact strings.

- [ ] **Step 1: Add the `--auto` bullet to §1 Intake**

In `skills/idea-polish/SKILL.md`, after the `--resolve-first` bullet (the one reading "**`--resolve-first`:** if present, skip entry classification and resolve before the first critique."), insert:

```markdown
- **`--auto`:** non-interactive mode, for agent/routine callers (see § Agent &
  routine invocation). With `--auto` the run never asks the user anything: a
  missing idea (no argument and no `--file`) stops the run with
  `error: --auto requires an idea argument or --file` instead of asking, and
  charter capture derives without confirming (§1a). Wherever this skill says
  "non-interactive run", the trigger is `--auto`; without the flag the run is
  interactive.
```

Also update the Idea bullet at the top of §1 — change:

```markdown
- **Idea:** from the command argument; else from a `--file <path>` argument; else
  ask the user for it.
```

to:

```markdown
- **Idea:** from the command argument; else from a `--file <path>` argument; else
  ask the user for it (in an `--auto` run, stop instead with
  `error: --auto requires an idea argument or --file`).
```

- [ ] **Step 2: Wire `--auto` into §1a charter capture**

In §1a's numbered procedure, change step 1 from:

```markdown
1. Distill `{problem, approach}` from `idea-v0.md` in plain language, 1–2
   sentences each. If the seed has no discernible approach (a brain-dump of
   unresolved concerns), **ask the user to state it** rather than inventing one.
```

to:

```markdown
1. Distill `{problem, approach}` from `idea-v0.md` in plain language, 1–2
   sentences each. If the seed has no discernible approach (a brain-dump of
   unresolved concerns), **ask the user to state it** rather than inventing one —
   except in an `--auto` run: never ask; derive the most plausible Approach
   best-effort and mark the charter `unconfirmed (derived)`.
```

And change step 2's last sentence from:

```markdown
   In a
   non-interactive run, derive it, mark it `unconfirmed`, and log that
   confirmation was skipped.
```

to:

```markdown
   In a
   non-interactive run (`--auto`), derive it, mark it `unconfirmed` (or
   `unconfirmed (derived)` when the Approach was invented per step 1), and log
   that confirmation was skipped.
```

- [ ] **Step 3: Surface the marker in §5b**

In §5b's `## Charter & shifts` bullet, change:

```markdown
- `## Charter & shifts` — the (final) charter (problem + thesis, noting `unconfirmed`
  if it was never confirmed), then every charter shift event from §4b′ in order:
```

to:

```markdown
- `## Charter & shifts` — the (final) charter (problem + thesis, noting `unconfirmed`
  if it was never confirmed, and `unconfirmed (derived)` if the Approach was also
  derived unattended in an `--auto` run), then every charter shift event from §4b′
  in order:
```

- [ ] **Step 4: Verify the edits**

Run:

```bash
grep -c -- '--auto' skills/idea-polish/SKILL.md
grep -n 'unconfirmed (derived)' skills/idea-polish/SKILL.md
grep -n 'error: --auto requires' skills/idea-polish/SKILL.md
```

Expected: first count ≥ 5; the marker appears in §1a and §5b (two or more hits); the error string appears in §1 (two hits: Idea bullet + `--auto` bullet).

- [ ] **Step 5: Commit**

```bash
git add skills/idea-polish/SKILL.md
git commit -m "feat(idea-polish): --auto flag defines the non-interactive run"
```

---

### Task 2: "Agent & routine invocation" section in SKILL.md

**Files:**
- Modify: `skills/idea-polish/SKILL.md` (append after § Acceptance check, end of file)

**Interfaces:**
- Consumes: the `--auto` flag semantics from Task 1.
- Produces: the section heading `## Agent & routine invocation` — Task 1's `--auto` bullet cross-references it, and Task 3's README paragraph points to it.

- [ ] **Step 1: Append the section**

Add at the end of `skills/idea-polish/SKILL.md`:

````markdown
## Agent & routine invocation

Other agents and scheduled routines call this skill **headlessly**. The
coordinator must run as the main agent — its critic/resolver turns spawn Task
subagents, and subagents cannot spawn nested subagents — so callers shell out
rather than Task-spawning the skill:

```
claude -p "/idea-polish --file idea.md --auto [--rounds N] [--peers ...]"
```

Run it from the directory where `runs/` should land. With `--auto` the run never
prompts a human (§1); the charter is derived unconfirmed (§1a) and every gate is
autonomous (§4b′, §4c).

**Output contract for callers:** the final message names
`runs/<ts>/final-idea.md` and `runs/<ts>/summary.md` and prints the polished
idea — a calling agent consumes stdout or reads the two files.

**Routine wiring example:** a scheduled routine whose prompt is — for each file
in `ideas/inbox/`, run `claude -p "/idea-polish --file <that file> --auto"`,
then move the processed file to `ideas/done/`.
````

- [ ] **Step 2: Verify**

Run:

```bash
grep -n '## Agent & routine invocation' skills/idea-polish/SKILL.md
```

Expected: exactly one hit, after the Acceptance check section.

- [ ] **Step 3: Commit**

```bash
git add skills/idea-polish/SKILL.md
git commit -m "docs(idea-polish): agent & routine invocation contract"
```

---

### Task 3: README usage additions

**Files:**
- Modify: `README.md` (§ Usage, lines 37–61)

**Interfaces:**
- Consumes: the `--auto` flag and the section name `Agent & routine invocation` from Tasks 1–2.

- [ ] **Step 1: Add `--auto` to the usage block and flag list**

In `README.md` § Usage, add to the fenced command list (after the `--peers` line):

```
/idea-polish --file idea.md --auto
```

Add to the flag bullets (after the `--without` bullet):

```markdown
- `--auto` — non-interactive: never asks a human (errors if no idea is given;
  charter derived unconfirmed). For agents and scheduled routines.
```

- [ ] **Step 2: Add the headless-invocation paragraph**

After the flag bullets and the default-set paragraph (before `### Adding a model`), insert:

````markdown
### Headless / agent invocation

Agents and scheduled routines run the skill headlessly from the directory where
`runs/` should land:

```
claude -p "/idea-polish --file idea.md --auto"
```

The final message names `runs/<ts>/final-idea.md` and `runs/<ts>/summary.md` and
prints the polished idea. See `skills/idea-polish/SKILL.md` § Agent & routine
invocation for the full contract.
````

- [ ] **Step 3: Verify**

Run:

```bash
grep -n -- '--auto' README.md
grep -n 'Headless / agent invocation' README.md
```

Expected: `--auto` appears in the usage block, the flag list, and the new subsection (≥ 3 hits); the subsection heading appears once.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: README usage for --auto headless invocation"
```

---

### Task 4: Acceptance check (headless run)

**Files:**
- None modified — verification only.

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Run the spec's acceptance check**

From the repo root:

```bash
claude -p "/idea-polish --file examples/sample-idea.md --auto --rounds 2"
```

Expected: completes with no prompt to a human; a new `runs/<ts>/` folder contains `final-idea.md` and `summary.md`; `summary.md`'s `## Charter & shifts` marks the charter `unconfirmed`. (Peer CLIs unreachable is fine — the run degrades to single-model self-review per §2.)

- [ ] **Step 2: Confirm and clean up**

```bash
ls runs/ | tail -1
grep -n 'unconfirmed' "runs/$(ls runs/ | tail -1)/summary.md"
```

Expected: the marker is present. Leave the run folder in place (`runs/` is already the repo's scratch output area).
