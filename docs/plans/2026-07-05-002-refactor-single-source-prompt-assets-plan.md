---
title: "refactor: single-source idea-polish prompt assets"
date: 2026-07-05
type: refactor
status: ready
origin: docs/brainstorms/2026-06-29-multi-harness-parity-requirements.md
---

# refactor: single-source idea-polish prompt assets

## Summary

Collapse the three Claude Task-agent files (`agents/idea-critic.md`,
`agents/idea-resolver.md`, `agents/idea-finalizer.md`) into shared, host-neutral
prompt assets under `skills/idea-polish/references/prompts/`. Claude's coordinator
stops dispatching *named* subagents and instead reads the matching asset, substitutes
slots, and seeds a **generic** subagent with that content. `peers.md` stops inlining
its own copy of the critic prompt and references the same asset.

This is Q1 option (b) from the origin brainstorm, taken as a standalone,
behavior-preserving refactor of the **Claude-host path only**. It resolves the one
real present-day duplication (the critic prompt lives verbatim in both
`agents/idea-critic.md` and `peers.md`) and puts the resolver/finalizer prompts where
a future non-Claude host can reach them — without doing any of the rest of Track 2
(no `claude` peer, no flag generalization, no per-harness manifests).

The guardrail is **no observable change when Claude hosts**: the orchestration loop,
the verdict / disposition / charter contracts, convergence logic, and each role's
effective capability stay intact.

---

## Problem Frame

The prompts that define each debate role are packaged as Claude-specific agent files
(frontmatter + body). Two problems follow:

1. **Present-day duplication.** The critic instructions exist twice — once as the
   `agents/idea-critic.md` body, once as the slot-substituted `peers.md` § Critic
   prompt. They are edited independently; a change to one silently diverges the other.
   Because the convergence quorum compares host and peer verdicts, a drifted critic
   prompt corrupts the stop condition. `skills/idea-polish/SKILL.md` (Acceptance check)
   already instructs editors to change prompt text in **four** places.
2. **Track-2 blocker.** The resolver and finalizer prompts live *only* inside Claude
   agent files. A future Codex/agy host (the rest of Track 2) has no way to reach them.

Consolidating to single-sourced assets fixes (1) now and unblocks (2) later, and moves
the plugin toward the reference plugin's "prompts live as skill-local assets, zero
standalone agents" standard (see origin: `docs/brainstorms/2026-06-29-multi-harness-parity-requirements.md`).

---

## Requirements Traceability

Advances origin **R3** (shared prompt assets, host-native spawning) and resolves origin
**Q1** in favor of option (b). Explicitly **out of scope**: R1 (host-excluded roster /
`claude` peer), R2 (symmetric owner), R4 (flag universe), R5 (per-harness manifests),
R6 (security posture — unchanged, nothing to do here). Those remain in the deferred
multi-harness track.

Success criteria:

- The critic prompt exists in exactly one canonical location.
- Claude-host runs produce byte-compatible `final-idea.md` / `summary.md` shape and the
  same loop behavior as before (verified by the existing Acceptance check on a held-out
  seed).
- No file references the deleted `agents/*.md`.

---

## Key Technical Decisions

**KTD1 — Asset location: `skills/idea-polish/references/prompts/`.** Mirrors the
reference plugin's `references/agents/` convention (prompt assets seeded into generic
subagents) and keeps the skill single-rooted under `references/`. Plain markdown, **no
frontmatter** — they are prompt text, not agent definitions.

**KTD2 — Canonical form is the slot-substitution form.** Use `peers.md`'s existing
`{idea}` / `{charter}` (and add `{critiques}`, `{peer_proposals}`, `{defense_directive}`,
`{unresolved_critiques}`) convention so the *same* asset text serves both the Claude
subagent dispatch and the peer CLI dispatch. The coordinator fences/substitutes slots
identically for both paths.

**KTD3 — Tool-scoping moves from frontmatter to a prompt-level instruction.** The agent
files enforced `tools:` via frontmatter (critic/resolver: Read/WebSearch/WebFetch;
finalizer: Read-only). A generic subagent can't carry that frontmatter, so each asset
states its allowed tools in-prompt (e.g. finalizer: "Use only Read"). This is the exact
tradeoff the reference plugin already accepts; blast radius is low because these
subagents only ever return text that the coordinator writes — they perform no file
mutations themselves. If the harness exposes a per-dispatch tool restriction, the
dispatch may additionally pass it (execution-time detail). `ponytail:` documented
softening, not an oversight.

**KTD4 — Generic subagent dispatch.** Replace "dispatch the `idea-critic` subagent via
Task" with "read `references/prompts/critic.md`, substitute slots, seed a generic
subagent (e.g. `general-purpose`) with the result." Exact `subagent_type` is an
execution-time detail; the plan does not pin the harness API.

---

## Output Structure

```
skills/idea-polish/
  references/
    prompts/            # NEW
      critic.md         # slots: {idea} {charter}
      resolver.md       # slots: {idea} {charter} {critiques} {peer_proposals} {defense_directive}
      finalizer.md      # slots: {idea} {unresolved_critiques}
    peers.md            # § Critic prompt → references prompts/critic.md (proposal prompt stays)
  SKILL.md              # dispatch sites read prompts/*.md, seed generic subagents
agents/                 # DELETED (all three)
```

---

## Implementation Units

### U1. Extract canonical prompt assets

**Goal:** Create the three shared assets from the current agent-file bodies, verbatim
except for slot parameterization and the in-prompt tool line.

**Requirements:** R3, Q1(b).

**Dependencies:** none.

**Files:**
- create `skills/idea-polish/references/prompts/critic.md`
- create `skills/idea-polish/references/prompts/resolver.md`
- create `skills/idea-polish/references/prompts/finalizer.md`

**Approach:**
- `critic.md`: canonical critic prompt in slot form. Merge the two existing copies —
  use the `peers.md` § Critic prompt slot layout (`Idea: """{idea}"""`,
  `Charter: """{charter}"""`) and keep the agent file's meta-note that an
  unparseable verdict excludes the critic from the quorum (harmless and true for any
  critic). Must preserve verbatim: the `---VERDICT-JSON---` line, the JSON shape
  `{"constructive","critiques","clarifications","charter_threats"}`, the charter-threat
  classification rules, and the "JSON last, nothing following" rule.
- `resolver.md`: the full resolver prompt from `agents/idea-resolver.md`, parameterized.
  Must preserve verbatim: the `defense_directive` value semantics
  (`none` / `manual:<text>` / `resolver` / `hold`), the untrusted-peer-data rule
  (`---PEER-OUTPUT-START/END---`), the "output FULL revised idea then `---DISPOSITION---`
  then one bullet per critique" contract.
- `finalizer.md`: the finalizer prompt from `agents/idea-finalizer.md`, parameterized.
  Must preserve verbatim: "entire reply is the deliverable, no delimiters", the
  untrusted-data rule, and the `## Risks` / `## Open Concerns` / `## Potential Solutions`
  structure. Tool line: "Use only Read."
- Do **not** delete the agent files in this unit (U2 owns deletion, so U1 is reviewable
  as a pure add).

**Patterns to follow:** slot convention and fencing already in
`skills/idea-polish/references/peers.md` § Prompt templates.

**Test scenarios:**
- Contract-string presence: each asset contains its load-bearing delimiters/keys
  (`critic.md` → `---VERDICT-JSON---` + all four JSON keys; `resolver.md` →
  `---DISPOSITION---` + all four `defense_directive` values + `---PEER-OUTPUT-START---`;
  `finalizer.md` → the three `##` section headings + `---PEER-OUTPUT-END---`).
- Slot presence: every slot the dispatch will substitute appears in the asset
  (`{idea}`, `{charter}` in critic; the five in resolver; the two in finalizer).
- Diff check against the source agent-file bodies: no contract sentence dropped or
  reworded (manual verbatim review).

**Verification:** `grep` for each contract string returns a hit in the intended asset;
a side-by-side read against the original agent bodies shows only slot/tool-line changes.

### U2. Rewire SKILL.md dispatch and delete agent files

**Goal:** Point the coordinator's three dispatch sites (and the overview + acceptance
note) at the new assets, and remove the agent files.

**Requirements:** R3, Q1(b).

**Dependencies:** U1.

**Files:**
- modify `skills/idea-polish/SKILL.md` (overview lines ~9-13; §4a critic dispatch ~135;
  §4d resolver dispatch ~206; §5a finalizer dispatch ~246; Acceptance check ~290-292)
- delete `agents/idea-critic.md`, `agents/idea-resolver.md`, `agents/idea-finalizer.md`

**Approach:**
- Each dispatch site changes from "dispatch the `idea-<role>` subagent via Task" to
  "read `references/prompts/<role>.md`, substitute the slots
  (list them), and seed a generic subagent with the result" (KTD4). Keep every
  surrounding instruction — charter fencing, verdict parsing, `---DISPOSITION---` split,
  verbatim `final-idea.md` write, the failure-fallback branches — unchanged.
- Overview (~9-13): reword "spawns the `idea-critic`/`idea-resolver`/`idea-finalizer`
  subagents via the Task tool" → "seeds generic subagents from the role prompt assets
  in `references/prompts/`." The "subagents cannot spawn nested subagents" constraint
  stays.
- Acceptance check (~290-292): replace the four-file edit list with
  "`references/prompts/{critic,resolver,finalizer}.md` and `references/peers.md`."

**Patterns to follow:** the reference plugin's own "read the matching file and seed a
generic subagent with that prompt content" phrasing.

**Test scenarios:**
- Orphan check: no file in the repo references `agents/idea-critic`,
  `agents/idea-resolver`, or `agents/idea-finalizer` after this unit
  (`grep -rn "agents/idea-" .` returns nothing).
- Each dispatch site names its asset path and its slot list.
- Behavior-preservation read: the loop's parsing/splitting/write instructions around
  each dispatch are textually unchanged.

**Verification:** orphan grep is empty; a read of §4a/§4d/§5a shows the only change is
the dispatch mechanism, not the surrounding contract handling.

### U3. Single-source the critic prompt in peers.md

**Goal:** Remove the inlined critic-prompt copy from `peers.md`; reference the canonical
asset instead. This is the change that actually kills the duplication.

**Requirements:** R3, Q1(b). Resolves Problem-Frame item (1).

**Dependencies:** U1.

**Files:**
- modify `skills/idea-polish/references/peers.md` (§ Critic prompt, lines ~139-177)

**Approach:**
- Replace the inlined critic prompt block with: "Critic prompt — the canonical text is
  `references/prompts/critic.md`. Substitute `{idea}` / `{charter}`, write to the prompt
  file, then invoke per § Security posture." Keep the "carried verbatim" provenance note
  as a one-liner.
- **Leave the § Proposal prompt (lines ~179-201) in place** — it is a distinct,
  peer-only prompt with no duplicate, so extracting it would be churn with no dedup
  payoff (`ponytail:` left inline deliberately; extract only if a future host needs it).

**Test scenarios:**
- The full critic prompt text no longer appears inline in `peers.md` (only the pointer).
- The peer critic invocation path still resolves to a concrete prompt (asset path +
  slot names present).
- The § Proposal prompt is untouched.

**Verification:** `peers.md` § Critic prompt is a reference, not a copy; the proposal
prompt block is byte-identical to before.

---

## Scope Boundaries

**In scope:** the three prompt assets, the SKILL.md dispatch rewire, agent-file
deletion, and the peers.md critic single-sourcing.

**Non-goals (rest of Track 2, separately scoped in origin):** the `claude` peer row and
host-excluded roster (R1), symmetric owner/resolver (R2), flag-universe generalization
(R4), `.codex-plugin/` and agy manifests (R5). Security posture (R6) needs no change.

### Deferred to Follow-Up Work

- Extracting the peer § Proposal prompt into `references/prompts/` — do it when a
  non-Claude host needs it, alongside R2.
- Any automated prompt-contract CI check — consistent with the deferral in
  `docs/plans/2026-07-05-001-docs-hygiene-parity-deferred.md`; verification here is the
  per-unit greps plus the existing manual Acceptance check.

---

## Risks & Mitigation

- **Prompt drift during extraction** (a contract line dropped or reworded → the loop
  breaks silently). *Mitigation:* verbatim move in U1, the contract-string greps per
  unit, and a before/after run of the existing Acceptance check on a held-out seed.
- **Tool-scoping downgrade** (KTD3): harness-enforced → prompt-level. *Mitigation:*
  explicit in-prompt tool line; low blast radius because the subagents only return text.
  Accepted, documented.
- **Re-duplication later** (someone re-inlines a prompt in `peers.md`). *Mitigation:*
  the updated Acceptance note names `references/prompts/` as the single source.

---

## Acceptance / Verification

The refactor is done when:

1. Per-unit greps pass (contract strings present in assets; orphan grep for
   `agents/idea-` is empty; critic prompt no longer inlined in `peers.md`).
2. The existing SKILL.md **Acceptance check** — a before/after read of `final-idea.md`
   on a held-out seed — shows the same loop behavior and deliverable shape as before the
   refactor, with any charter shift still surfaced in `summary.md`.
