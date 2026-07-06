# Problem-relevance filter & Problem/Approach charter — design

Date: 2026-07-06
Status: approved design, pre-implementation
Scope: `skills/idea-polish/SKILL.md`, `references/prompts/critic.md`,
`references/prompts/resolver.md`, `references/peers.md`. Prose-only changes — no
manifests, no code.

## Goal

During the debate, every critique and every refined idea must serve the charter's
stated **Problem**. Off-problem tangents are discarded in that round (and logged),
never acted on. The Problem is **immutable for the whole run**; the **Approach**
(renamed from "Thesis") can evolve, but only through the user via the existing gate.

Motivating gap: today the critic is *asked* to classify against the charter and the
resolver is *asked* to stay anchored, but nothing verifies either, and the
"thesis / core bet" test is conceptual — different models draw the line
differently. This design makes the anchor textual and the checks enforced at three
layers (critic, resolver, coordinator).

## Terminology: Thesis → Approach

"Thesis" is too technical for many ideas (e.g. business ideas). Rename it
**Approach** everywhere in prose: SKILL.md, both role prompts, peers.md templates,
and the charter file section header (`## Thesis` → `## Approach`).

**Frozen-contract exception:** the verdict JSON key stays `charter_threats`
(reliability-critical; peers may run older prompts). Only its *meaning* narrows —
see § Gate below. Prose may say "approach threat"; the JSON key never changes.

## Charter capture (§1a) changes

- `## Problem` and `## Approach` are each constrained to **1–2 sentences** at
  capture time (short fixed text is what makes text-anchored classification
  workable). User confirmation flow is unchanged.
- The Problem is **frozen for the run**. It is never re-derived — including at the
  gate. Only the Approach can be re-derived on an accepted shift.

## New definitions (SKILL.md § Definitions, mirrored in prompts)

Both tests are **text-anchored** — the referent is the charter's sentences, not a
concept like "the core bet":

- **Off-problem:** a critique (or a revision) is off-problem when acting on it
  would not serve what the `## Problem` sentence says. Off-problem items are
  discarded in that round and logged; they are never resolved and never gated.
  A critique whose remedy is "solve a different problem" is off-problem.
- **Approach threat** (carried in the `charter_threats` JSON key): a critique whose
  fix would require **rewriting the `## Approach` sentence(s)**. These are
  on-problem challenges to how the idea wins; they route to the §4b′ gate exactly
  as charter threats do today. Problem shifts are no longer gate material — they
  are off-problem discards.

## Enforcement — three layers

### Layer 1: critic prompts (`prompts/critic.md` + peers.md critic template)

Add one rule: only emit critiques that bear on the charter's `## Problem`; drop
tangents before writing the verdict. Replace the conceptual threat test
("shifting the problem or thesis / abandoning the core bet") with the text-anchored
one above. Verdict JSON shape unchanged.

### Layer 2: resolver prompt (`prompts/resolver.md` + peers.md proposal template)

- Treat any received critique that does not serve the `## Problem` as out of
  scope: mark it `discarded (off-problem)` in the disposition instead of acting
  on it. (Independent backstop for anything layer 3 lets through.)
- **Verbatim problem echo:** every revised idea must begin with the charter's
  `## Problem` section reproduced **verbatim, unedited**. The resolver may never
  modify it. This makes `final-idea.md` self-anchoring and gives the coordinator a
  mechanical drift check.
- "Stay within the bet" language is re-anchored to the Approach; the
  defense-directive semantics (`none` / `manual:<text>` / `resolver` / `hold`)
  are unchanged apart from wording.

### Layer 3: coordinator (SKILL.md)

**Critique screen — §4b, after building the critique list.** Read each critique
(including unstructured ones from unparseable verdicts) against `charter.md`
`## Problem`. Remove off-problem items; log each as
`! discarded off-problem (<model>): <critique>` and record it for `summary.md`.
If the list empties (and no clarifications and no approach threats), use
`(no specific critiques)` and **skip §4d** — the idea carries forward unchanged
into the next round. The convergence quorum definition is untouched: a
`constructive: true` verdict still blocks convergence even if all its critiques
were discarded (conservatively correct — the next round's critics re-judge).

**Revision check — after §4d synthesis, two stages:**

1. **Mechanical:** string-compare the revised idea's leading `## Problem` section
   against `charter.md`'s (whitespace-normalized). Mismatch or missing ⇒ drift.
2. **Judgment backstop:** if the echo matches, read the revision body once — does
   it still target the Problem, or is the echo camouflage? Off-problem body ⇒
   drift.

On drift: re-run the resolver **once** with a corrective note naming the drift
appended to the critique context. If the retry still drifts: discard the revision,
keep the prior idea as current, record the event, and continue to the next round.
Drift is never `resolve_failed` (that reason stays reserved for call failures).

### Gate (§4b′) — approach-only

The gate now handles **approach threats only**. Changes:

- The "which charter element each targets (problem / thesis)" presentation drops
  the problem case; threats target the Approach.
- **Accept-continue** re-derives **only the `## Approach` section** of
  `charter.md`; the `## Problem` is never rewritten.
- All other branches (accept-signoff, defend-manual, defend-resolver,
  non-interactive hold) unchanged apart from terminology.

## Output changes (§5)

- `summary.md` gains `## Off-problem discards`: one line per discarded critique
  (`round, model, critique, layer that caught it`) and one line per discarded
  revision/retry event; or `none`. Process record only — never idea content.
- `## Charter & shifts` renders the Problem/Approach terms and notes the Problem
  is frozen by design.
- `final-idea.md` now begins with the verbatim `## Problem` (a consequence of the
  echo, not a new finalizer rule); the finalizer prompt's "still serves the
  charter's problem" self-check language is re-anchored to the frozen sentence.

## Contracts preserved (unchanged, verbatim)

- Verdict delimiter, JSON shape, and the `charter_threats` key (optional ⇒ `[]`).
- Disposition delimiter and split.
- Convergence quorum definition.
- Output split (`final-idea.md` deliverable / `summary.md` pointers-only).
- Security posture in peers.md (args-only roster, prompt via file/stdin,
  untrusted-output wrapping).

## Deliberate divergence from the Python reference

The `idea_polisher` CLI reference gates problem *and* thesis shifts and has no
relevance filter, no verbatim echo, and no Problem/Approach terminology. This
design diverges on purpose (user decision, this spec): frozen Problem,
approach-only gate, three-layer relevance filter. Record this divergence in the
repo docs when implementing; do not port it back silently.

## Error handling summary

| Failure | Behavior |
| --- | --- |
| All critiques in a round discarded | Skip resolve; idea unchanged; round still counts toward K |
| Revised idea fails echo or judgment check | One corrective retry → else keep prior idea, log, continue |
| Peer on older prompt emits tangents | Caught by coordinator screen (layer 3); logged discard |
| Resolver call itself fails | Existing `resolve_failed` stop, unchanged |

## Acceptance check

Run `/idea-polish` on `examples/sample-idea.md` before and after. Verify:

1. `charter.md` has 1–2-sentence `## Problem` / `## Approach` sections.
2. Every `idea-v<n>.md` and `final-idea.md` begins with the verbatim `## Problem`.
3. `summary.md` contains `## Off-problem discards` (populated or `none`) and no
   idea content.
4. Seeding an idea with an obvious tangent critique embedded (resolve-first path)
   produces a logged discard, not a pivot.
