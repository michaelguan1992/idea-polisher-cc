---
name: idea-polish
description: Polish an idea via a multi-round cross-model critic/resolver debate (Claude + Codex + Antigravity). Use when the user wants an idea hardened by adversarial cross-model review.
---

# Idea Polish (coordinator)

This skill is the **orchestrator** and runs in the main agent context. It owns the
loop: for Claude's own turns it seeds a generic subagent (via the Task tool) from the
role prompt assets in `references/prompts/` (`critic.md` / `resolver.md` /
`finalizer.md`), and calls the roster's peer CLIs (`references/peers.md` § Peer
roster; `codex` + `agy` by default) via Bash. Subagents
cannot spawn nested subagents, so the loop, fan-out, and aggregation must live
here, not inside a subagent.

Claude is the **host**, so it is always available and is the default idea **owner /
resolver** (its turns run natively — no `claude -p` subprocess). Codex and
Antigravity are **peers**: they critique and propose fixes via Bash, best-effort.

The behavior below is carried from the `idea_polisher` CLI reference
implementation. Follow the numbered procedure exactly — because the loop is
prose-orchestrated, the convergence quorum and the verdict contract are the
reliability-critical parts. See `references/peers.md` for the exact peer commands,
prompt templates, and the security posture.

## Definitions (carry verbatim)

- **Verdict delimiter:** `---VERDICT-JSON---`. A critic's verdict is the JSON object
  after the **last** occurrence. Shape: `{"constructive": bool, "critiques": [str], "clarifications": [str], "charter_threats": [str]}`.
- **Approach threat:** a critique point whose fix would require **rewriting the
  charter's `## Approach` sentence(s)** (§1a) — a challenge to how the idea wins,
  judged against the charter's text, not a notion like "the core bet". Threats go
  in `charter_threats` (**the JSON key keeps its historical name — never rename
  it**) and are **not** duplicated in `critiques`. The field is **optional for
  backward compatibility**: a verdict that omits it parses with
  `charter_threats: []` (a peer running an older prompt simply contributes no
  threats; it does not break the round).
- **Off-problem:** a critique (or a revision) is off-problem when acting on it
  would not serve what the charter's `## Problem` sentence says — a tangent, or a
  remedy that solves a different problem (the Problem is frozen for the run).
  Off-problem items are **discarded in the round they appear** and logged in
  `summary.md`; they are never resolved and never gated. Enforced at three
  layers: the critic prompts (self-filter), the resolver's disposition
  (`discarded (off-problem)`), and the coordinator (§4b screen, §4d drift check).
- **Disposition delimiter:** `---DISPOSITION---`. The resolver's reply is the revised
  idea before it, the per-critique disposition after it.
- **Parse failure:** a critic call that succeeded but whose verdict can't be parsed
  (no delimiter, or invalid JSON). Its raw text is still usable as an unstructured
  critique, but it is **excluded from the convergence quorum**.
- **Convergence quorum:** the loop has converged iff **at least one** verdict parsed
  AND **every parsed** verdict has `constructive: false` with no `clarifications` and
  no `charter_threats`. Parse failures and failed calls do not count toward or against
  the quorum. (A live approach threat keeps the loop from converging — it is routed to
  the §4b′ gate, whose resolver defense is what talks later rounds out of re-raising it.)

## Procedure

### 1. Intake

- **Idea:** from the command argument; else from a `--file <path>` argument; else
  ask the user for it (in an `--auto` run, stop instead with
  `error: --auto requires an idea argument or --file`).
- **K (max rounds):** `--rounds N`, default **10**.
- **Owner:** default `claude` (the host). A non-Claude owner is out of scope for v1.
- **Peers:** resolve the effective peer set from `references/peers.md` § Peer roster
  and these flags:
  - `--peers <a,b,...>` — base set (overrides the defaults).
  - `--with <a,b,...>` — add to the base.
  - `--without <a,b,...>` — remove from the base.

  Resolution: `base = --peers if given, else the roster's default-on peers (codex,
  agy); selected = (base + --with) − --without`. Validate every name in any flag
  against the roster — an unknown name (including `claude`, which is the host, not a
  roster peer) stops the run with `error: unknown peer '<name>'; registered: <list>`.
  **Retain the explicitly-named set** (the union of `--peers` and `--with`) so §2 can
  tell a named-but-missing peer from a default-but-absent one. No flags ⇒ `selected`
  is exactly the reachable default-on peers — identical to prior behavior.
- **`--resolve-first`:** if present, skip entry classification and resolve before the
  first critique.
- **`--auto`:** non-interactive mode, for agent/routine callers (see § Agent &
  routine invocation). With `--auto` the run never asks the user anything: a
  missing idea (no argument and no `--file`) stops the run with
  `error: --auto requires an idea argument or --file` instead of asking, and
  charter capture derives without confirming (§1a). Wherever this skill says
  "non-interactive run", the trigger is `--auto`; without the flag the run is
  interactive.
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
- **Run folder:** create `runs/<YYYYMMDD-HHMMSS>/` under the current working
  directory and use it for every prompt file, snapshot, and output below. All peer
  Bash calls use this folder as their working directory (see `references/peers.md`
  § Security). Write the original idea to `runs/<ts>/idea-v0.md`.

### 1a. Charter capture (the fidelity anchor — frozen for the whole run)

The **charter** is the seed's immutable core. It anchors every later round so a
critique can't silently steer the idea onto a different problem or a different
approach. Both sections are **1–2 sentences each** — short, fixed text is what the
off-problem and approach-threat tests anchor to (the referent is the charter's
sentences, not a concept). Scope/breadth is deliberately **not** in it — narrowing
a platform to one workflow is legitimate convergence, not drift.

- **Problem** — the pain/need the idea exists to serve (1–2 sentences).
  **Frozen: never rewritten by anyone during the run.** Every revision reproduces
  it verbatim as its first section (§4d).
- **Approach** — *how* the idea wins and *why* it is defensible, named at the
  **mechanism** level (the moat/flywheel/wedge), not just the topic
  (1–2 sentences). "A reuse platform" is a topic; "a cross-customer ML
  effectiveness-discriminator flywheel as the moat" is an approach. Challenges to
  it are defended by the resolver (§4b′) and recorded in `summary.md` for the user
  to act on between runs.

Procedure:

1. Distill `{problem, approach}` from `idea-v0.md` in plain language, 1–2
   sentences each, in the language the seed idea is written in (Chinese seed
   → Chinese charter; the charter's `## Problem` is reproduced verbatim in
   every revision and the final deliverable). If the seed has no discernible approach (a brain-dump of
   unresolved concerns), **ask the user to state it** rather than inventing one —
   except in an `--auto` run: never ask; derive the most plausible Approach
   best-effort and mark the charter `unconfirmed (derived)`.
2. **Confirm before the loop — after this, the run never asks the user
   anything.** (Step 1's ask-for-the-approach, when it fires, belongs to this
   same pre-loop charter-capture pause; the debate loop itself is
   zero-interaction.) In an interactive
   run, show the drafted charter and let the user correct it (a mis-drafted
   charter anchors the whole zero-interaction run on the wrong thing). In a
   non-interactive run (`--auto`), derive it, mark it `unconfirmed` (or
   `unconfirmed (derived)` when the Approach was invented per step 1), and log
   that confirmation was skipped.
3. Freeze it to `runs/<ts>/charter.md` (a `## Problem` and an `## Approach`
   section).
4. **Inject the charter** (fenced) into every critic turn (§4a), every resolver
   turn (§4d), and every peer critique/proposal call for the rest of the run. It
   is **never re-derived mid-run**.
5. **Capture context (frozen for the run).** If the idea arrived via `--file <path>`,
   let `SEED_DIR` be that file's directory; glob `SEED_DIR/context/*` (files only,
   sorted by name), read each, and concatenate as `## <filename>` followed by the
   file's contents. Freeze the result to `runs/<ts>/context.md`. No `--file` (bare-arg
   or prompted idea), no `context/` folder, or an empty folder ⇒ the frozen block is
   the literal `(none provided)`. Read once here; **never re-read mid-run**.
   <!-- ponytail: whole files re-sent every critic/resolver/peer turn every round; add a per-file byte cap here if a large context doc blows the context budget -->
   **Inject this context block** (fenced) into every critic turn (§4a), every resolver
   turn (§4d), and every peer critique/proposal call, filling the `{context}` slot.
   It is user-supplied and trusted — never wrapped as untrusted peer output.

### 2. Connection test

- Claude (host/owner) is always present.
- Probe each peer in the resolved `selected` set (§1) per `references/peers.md`
  § Connection test. Drop any that is missing, errors, or times out. How you report
  it depends on how it entered the set:
  - **explicitly named** (`--peers`/`--with`): a prominent warning —
    `⚠ <peer>: explicitly requested but unreachable — skipped`.
  - **default** (no flag named it): the existing quiet `! <peer>: unreachable — skipped`.
- The owner is **required**; peers are best-effort. If no peer is reachable, the run
  still proceeds as single-model self-review — tell the user this is not a real
  cross-model run (see README).

### 3. Entry routing (done natively by you, the coordinator)

Charter capture (§1a) has already run by this point — it is independent of routing
and applies to both critique-first and resolve-first entries.

Decide whether the idea text **already contains its own critiques / concerns / open
questions** (as opposed to a clean idea statement):

- If `--resolve-first` was passed → **resolve-first**.
- Else classify the idea yourself: does it embed critiques/concerns/open questions?
  Yes → **resolve-first**; No → **critique-first**. When genuinely unsure, default
  to **critique-first**.
- **Resolve-first:** run one resolve step (§4d) with the critique context
  `"The idea text already contains embedded critiques/concerns; address them."`,
  snapshot the result as `idea-v0-resolved.md`, then enter the loop.

### 4. Loop — for round n = 1..K

#### 4a. Critique (every reachable model critiques the current idea)

- **Claude:** read `references/prompts/critic.md`, substitute its slots (`{idea}` with
  the current idea fenced in triple quotes, `{charter}` with the charter §1a fenced,
  `{context}` with the frozen context block §1a fenced), and seed a generic subagent
  (via Task, passing `--critic-model` as the `model` parameter when set) with the
  result — when `--critic-effort` is set, prepend its thinking directive (§1) as
  the first line of the seeded prompt. Its final message is the verdict block.
- **Peers:** for each reachable peer (the survivors of §2), send the critic prompt
  per `references/peers.md` § Critic prompt (which includes the charter and context)
  and capture
  stdout. Build the call from the peer's roster `Command` + `Input`, with the
  coordinator appending the prompt — never a roster string that already contains it
  (see `peers.md` § Security posture).
- For each result, parse the verdict (JSON after the last `---VERDICT-JSON---`),
  reading `charter_threats` as `[]` when the key is absent:
  - call failed → log `! <model>: critique call failed — skipped this round`.
  - parsed → record the verdict (including its `charter_threats`).
  - succeeded but unparseable → log `! <model>: verdict unparseable — excluded from
    convergence quorum`, and keep its raw text as an unstructured critique.
- Save the round's parsed verdicts (with `charter_threats`) to
  `runs/<ts>/critiques-<n>.json`.

#### 4b. Convergence check

- Apply the convergence quorum. If converged → stop with reason `converged`
  (idea unchanged this round).
- Otherwise build the critique list for the resolver: one bullet per parsed critique
  as `- (<model>) <critique>`, plus `- (<model>, unstructured) <raw>` for each
  succeeded-but-unparseable critic. If there are none, use `(no specific critiques)`.
  `charter_threats` are **not** in this list — they are gated below in §4b′ and, when
  the gate fires, reach the resolver as a labelled threats-to-defend block, never as
  ordinary critiques.
- **Problem-relevance screen (coordinator):** judge every item in the critique list
  (including unstructured ones) against `charter.md`'s `## Problem`. Remove each
  off-problem item (see § Definitions), log it as
  `! discarded off-problem (<model>): <critique>`, and record it for §5b. Never ask
  the user — the coordinator judges alone.
- If the screen empties the list **and** this round has no clarifications and no
  approach threats, skip §4c–§4d entirely — the idea carries forward unchanged into
  §4e. The convergence quorum is unaffected: a `constructive: true` verdict still
  blocks convergence even when all its critiques were discarded (the next round's
  critics re-judge).

#### 4b′. Approach gate (autonomous — the resolver defends, the user reads the record)

Union the `charter_threats` from all parsed verdicts this round. If the set is
empty, set `defense_directive = none` and continue to §4c. Otherwise a challenge to
the charter's **Approach** has been detected. Never pause — the loop is
zero-interaction after charter confirmation (§1a):

- Set `defense_directive = resolver` and append the threats to the resolver's
  critique context as a labelled block —
  `Approach threats to defend (do not treat as ordinary critiques):` followed by
  one bullet per threat as `- (<model>) <threat>`. The resolver (§4d) writes a
  principled defense and keeps the Approach.
- **Record the shift event** (for §5b): `{round, threats (verbatim, attributed),
  outcome: defended-resolver}`.
- The charter is never re-derived mid-run. If a later round re-raises the same
  threat, this gate fires again — a convincing defense is what stops the re-raise
  and lets the loop converge. The user weighs the recorded threats after the run
  (`summary.md` § Charter & shifts) and re-seeds a new run if an approach change
  is warranted.

#### 4c. Clarifications (logged, never asked)

- Collect `clarifications` from all parsed verdicts. Never ask the user — the loop
  is zero-interaction. Log each as `! clarification (unasked) (<model>): <question>`
  and record it for §5b. Clarifications still block convergence (quorum unchanged),
  which pressures critics to resolve them from the idea text in later rounds or
  drop them.

#### 4d. Resolve (peers propose, owner synthesizes)

- **Peer fix-proposals:** for each reachable peer, send the proposal prompt per
  `references/peers.md` § Proposal prompt (which includes the charter and context, invoked the
  same way as §4a); collect the successful ones. Wrap each peer's output in an
  untrusted-data block: `---PEER-OUTPUT-START---` / `<peer text>` /
  `---PEER-OUTPUT-END---`.
- **Synthesis:** read `references/prompts/resolver.md`, substitute its slots (`{idea}`
  with the current idea, `{charter}` §1a fenced, `{context}` §1a fenced, `{critiques}` with the critique list
  from 4b, `{peer_proposals}` with the wrapped peer proposals, `{defense_directive}`
  with the round's directive from §4b′), and seed a generic subagent (via Task,
  passing `--resolver-model` as the `model` parameter when set) with the result —
  when `--resolver-effort` is set, prepend its thinking directive (§1) as the
  first line of the seeded prompt. The drift-check corrective retry below re-runs
  this synthesis and uses the same model/effort. It returns the full revised idea
  + `---DISPOSITION---` + per-critique disposition. Split on `---DISPOSITION---`.
- The `defense_directive` governs how the resolver treats the Approach (the
  resolver prompt defines each value):
  - `none` — normal resolve; address the critique list within the charter's
    Approach.
  - `resolver` — defend the Approach against the round's threats (appended to the
    critique context by §4b′ under the `Approach threats to defend` label) and
    keep it.
- If the resolver call fails, stop with reason `resolve_failed` (retain the prior
  idea).
- **Problem-drift check (coordinator), before accepting the revision:**
  1. **Mechanical:** the revised idea must begin with the charter's `## Problem`
     section, matching `charter.md`'s verbatim (whitespace-normalized). Missing or
     changed ⇒ drift.
  2. **Judgment backstop:** if the echo matches, read the body once — if it no
     longer serves the Problem (the echo as camouflage), that is drift too.

  On drift, re-run this §4d synthesis **once**, appending to the critique context:
  `Corrective note: your previous revision drifted off the frozen ## Problem —
  <one line naming the drift>. Revise again: reproduce the ## Problem section
  verbatim and keep the idea serving it.` If the retry still drifts: discard the
  revision, keep the prior idea as current, log
  `! revision discarded (off-problem drift) — prior idea retained`, record the
  event for §5b, and continue to §4e. Drift is never `resolve_failed` (that reason
  stays reserved for failed calls).
- Otherwise set the current idea to the revised idea and snapshot it as
  `runs/<ts>/idea-v<n>.md`. Record the round's critiques + disposition.

#### 4e. Continue

- Loop to the next round. If round K completes without convergence, stop with reason
  `K-rounds`.

### 5. Output — finalize and write `runs/<ts>/{final-idea.md, summary.md}`

The loop produces a deliverable (`final-idea.md`) and a process record (`summary.md`).
Run the finalize step first, then write both files. This runs for **every** stop reason
(`converged`, `K-rounds`, `resolve_failed`) and for the no-reachable-peers
single-model case — always from whatever the current idea is at loop end.

#### 5a. Finalize (owner synthesizes the deliverable)

- Collect the **rejected** critiques across all rounds from the per-round dispositions
  recorded in §4d (the ordinary critiques that were raised but left unresolved).
  **Approach-threat events (§4b′) and off-problem discards are not passed to the finalizer** — they are process
  record, kept out of the deliverable and written only to `summary.md` (§5b).
- Read `references/prompts/finalizer.md`, substitute its slots (`{idea}` with the
  current/final idea, `{unresolved_critiques}` with that list of unresolved critiques),
  and seed a generic subagent (via Task) with the result. Its entire reply is the
  deliverable.
- Write the finalizer's returned text **verbatim** to `runs/<ts>/final-idea.md` (no
  parsing, no delimiter to split on).
- If the finalize dispatch fails, fall back to writing the bare final idea text to
  `final-idea.md` and tell the user the structured finalize step was skipped — never
  leave the deliverable unwritten.

#### 5b. Evolution record — write `runs/<ts>/summary.md`

`summary.md` is the debate timeline, not the deliverable — it does **not** reproduce the
full idea. **Hard rule: `summary.md` must never contain the deliverable. Do not paste the
final idea, and do not inline the body of any `idea-v*.md` snapshot. Refer to the idea by
filename (`final-idea.md`, `idea-v<n>.md`) only.** The sections below are the *entire*
permitted contents:

- Header: participating models, stop reason, and a pointer to `final-idea.md` as the
  deliverable.
- `## Original idea` — the seed.
- `## Charter & shifts` — the charter (Problem + Approach, noting `unconfirmed`
  if it was never confirmed, and `unconfirmed (derived)` if the Approach was also
  derived unattended in an `--auto` run), then every charter shift event from §4b′
  in order: `round`, the threat(s) verbatim (attributed by model),
  and the outcome (always `defended-resolver`). If the gate never fired, say "no
  approach threats — the idea stayed on its seed." This section is where the user
  decides, **after** the run, whether a threat deserves a re-seeded Approach.
  Threats live **here only**, not in `final-idea.md`.
  End the section with one line — `Context loaded: <file>, <file>` (the filenames from
  §1a step 5, comma-separated) or `Context loaded: none` — so a reader knows what
  background the critics and resolver saw.
- `## Off-problem discards` — one line per discarded critique
  (`round, model, critique, layer that caught it: resolver / coordinator` — the
  critic layer self-filters silently before the verdict, so its drops never
  reach the coordinator and are not line items),
  one line per discarded revision or corrective retry from the §4d drift check, and
  one line per unasked clarification (`round, model, question`); or "none".
- `## Round-by-round evolution` — for each round: the critiques raised (attributed by
  model) and the resolver's disposition for each (addressed / rejected), or "Converged:
  no constructive critiques", or "Resolve step failed; prior idea retained".
- `## Idea snapshots` — list the `idea-v*.md` files, with the note: *revisions are not
  guaranteed monotonic — the last round may not be the best.*

**Output-contract self-check** (before finishing): confirm `final-idea.md` exists as a
standalone file holding the full deliverable, and that `summary.md` contains no copy of
the idea body — only pointers by filename. If either fails, fix it before reporting done.

Tell the user where both `final-idea.md` and `summary.md` landed, and print the final
polished idea (from `final-idea.md`).

## Acceptance check

On a held-out seed idea, a before/after read of `final-idea.md` should confirm the
final idea is more complete / specific than the seed, **begins with the charter's
`## Problem` verbatim**, and still serves that Problem within its `## Approach`.
After charter confirmation the run must complete with **zero further prompts to the
user**: approach threats appear in `summary.md` § Charter & shifts as
`defended-resolver` events, tangents in § Off-problem discards, clarifications as
unasked log lines — never as silent pivots or mid-run questions. To customize
behavior, edit the single-source role prompts in
`references/prompts/{critic,resolver,finalizer}.md` (the critic prompt is
referenced, not re-inlined, by `references/peers.md`).

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

**Prerequisite:** the plugin must be installed in the calling environment
(README § Install) — an uninstalled plugin fails immediately with
`Unknown command: /idea-polish`.

**Output contract for callers:** the final message names
`runs/<ts>/final-idea.md` and `runs/<ts>/summary.md` and prints the polished
idea — a calling agent consumes stdout or reads the two files.

**Routine wiring example:** a scheduled routine whose prompt is — for each file
in `ideas/inbox/`, run `claude -p "/idea-polish --file <that file> --auto"`,
then move the processed file to `ideas/done/`.
