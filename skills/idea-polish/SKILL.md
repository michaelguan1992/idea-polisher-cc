---
name: idea-polish
description: Polish an idea via a multi-round cross-model critic/resolver debate (Claude + Codex + Antigravity). Use when the user wants an idea hardened by adversarial cross-model review.
---

# Idea Polish (coordinator)

This skill is the **orchestrator** and runs in the main agent context. It owns the
loop: it spawns the `idea-critic` / `idea-resolver` / `idea-finalizer` subagents via the
Task tool for Claude's own turns, and calls the roster's peer CLIs (`references/peers.md` § Peer
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
- **Charter threat:** a critique point the critic judges would require **shifting the
  charter's problem or thesis** (§1a) — i.e. abandoning the seed's core bet, not
  improving the idea within it. Threats go in `charter_threats` and are **not**
  duplicated in `critiques`. The field is **optional for backward compatibility**: a
  verdict that omits it parses with `charter_threats: []` (a peer running an older
  prompt simply contributes no threats; it does not break the round).
- **Disposition delimiter:** `---DISPOSITION---`. The resolver's reply is the revised
  idea before it, the per-critique disposition after it.
- **Parse failure:** a critic call that succeeded but whose verdict can't be parsed
  (no delimiter, or invalid JSON). Its raw text is still usable as an unstructured
  critique, but it is **excluded from the convergence quorum**.
- **Convergence quorum:** the loop has converged iff **at least one** verdict parsed
  AND **every parsed** verdict has `constructive: false` with no `clarifications` and
  no `charter_threats`. Parse failures and failed calls do not count toward or against
  the quorum. (A live charter threat keeps the loop from converging — it is routed to
  the §4b′ gate instead.)

## Procedure

### 1. Intake

- **Idea:** from the command argument; else from a `--file <path>` argument; else
  ask the user for it.
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
- **Timeout:** per peer call, default **120s** (`--timeout`).
- **Run folder:** create `runs/<YYYYMMDD-HHMMSS>/` under the current working
  directory and use it for every prompt file, snapshot, and output below. All peer
  Bash calls use this folder as their working directory (see `references/peers.md`
  § Security). Write the original idea to `runs/<ts>/idea-v0.md`.

### 1a. Charter capture (the fidelity anchor — carry through the whole run)

The **charter** is the seed's immutable core. It anchors every later round so a
critique can't silently steer the idea onto a different problem or a different bet.
It has exactly two parts (scope/breadth is deliberately **not** in it — narrowing a
platform to one workflow is legitimate convergence, not drift):

- **Problem** — the pain/need the idea exists to serve (one sentence).
- **Thesis** — the central bet: *how* it wins and *why* it is defensible, named at
  the **mechanism** level (the moat/flywheel/wedge), not just the topic. "A reuse
  platform" is a topic; "a cross-customer ML effectiveness-discriminator flywheel as
  the moat" is a thesis.

Procedure:

1. Distill `{problem, thesis}` from `idea-v0.md` in plain language. If the seed has
   no discernible thesis (a brain-dump of unresolved concerns), **ask the user to
   state the bet** rather than inventing one.
2. **Confirm before the loop.** In an interactive run, show the drafted charter and
   let the user correct it (the charter is load-bearing — a mis-stated thesis anchors
   the gate on the wrong thing). In a non-interactive run, derive it, mark it
   `unconfirmed`, and log that confirmation was skipped.
3. Freeze it to `runs/<ts>/charter.md` (a `## Problem` and a `## Thesis` section).
4. **Inject the charter** (fenced) into every critic turn (§4a), every resolver turn
   (§4d), and every peer critique/proposal call for the rest of the run. It is
   re-derived only when the user **accepts** a shift (§4b′).

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
- **Resolve-first:** run one resolve step (§4c) with the critique context
  `"The idea text already contains embedded critiques/concerns; address them."`,
  snapshot the result as `idea-v0-resolved.md`, then enter the loop.

### 4. Loop — for round n = 1..K

#### 4a. Critique (every reachable model critiques the current idea)

- **Claude:** dispatch the `idea-critic` subagent via Task, passing the current idea
  (fenced in triple quotes) **and the charter** (§1a, fenced). Its final message is
  the verdict block.
- **Peers:** for each reachable peer (the survivors of §2), send the critic prompt
  per `references/peers.md` § Critic prompt (which includes the charter) and capture
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
  `charter_threats` are **not** in this list — they are gated below in §4b′ and reach
  the resolver only via the gate's outcome.

#### 4b′. Charter gate (fidelity anchor — the user steers any shift off the seed)

Union the `charter_threats` from all parsed verdicts this round. If the set is empty,
set `defense_directive = none` and continue to §4c. Otherwise a shift off the
charter's **problem or thesis** has been detected — do **not** resolve it silently:

- **Non-interactive run (no synchronous user):** **hold the charter.** Set
  `defense_directive = hold`, record the shift event (see below) with outcome
  `held`, and continue to §4c. The resolver (§4d) keeps the thesis and notes the
  threat unresolved; the run surfaces it in `summary.md` (§5b) for the user to decide
  later. Never silent-accept and never silent-pivot.
- **Interactive run:** pause and present the threatening point(s) and which charter
  element each targets (problem / thesis). Ask the user to steer (blocking question):
  - **Accept the shift** → then ask *Continue or sign off?*
    - **Continue** → the user has endorsed the new direction. Append the accepted
      threats to the resolver's critique list (so the resolver actually makes the
      shift), set `defense_directive = none`, **re-derive the charter** to the new
      direction (§1a procedure, overwrite `charter.md`), record outcome
      `accepted-continue`, and continue to §4c.
    - **Sign off** → stop the loop now with reason `charter_signoff`; the current
      idea (this round's input, not yet re-resolved) is final. Record outcome
      `accepted-signoff` and go to §5.
  - **Defend manually** → capture the user's rebuttal text; set
    `defense_directive = manual:<text>`; record outcome `defended-manual`; continue
    to §4c. The thesis is held.
  - **Resolver defends** → set `defense_directive = resolver`; record outcome
    `defended-resolver`; continue to §4c. The thesis is held. If a later round
    re-raises the same shift, this gate fires again.
- **Record the shift event** (for §5b) regardless of branch: `{round, elements
  (problem/thesis), threats (verbatim), outcome}`.

#### 4c. Clarifications (optional)

- Collect `clarifications` from all parsed verdicts. If any and the run is
  interactive, ask the user and append `Clarifications from the author:` + the Q/A
  to the critique context. In non-interactive runs, log that they were skipped.

#### 4d. Resolve (peers propose, owner synthesizes)

- **Peer fix-proposals:** for each reachable peer, send the proposal prompt per
  `references/peers.md` § Proposal prompt (which includes the charter, invoked the
  same way as §4a); collect the successful ones. Wrap each peer's output in an
  untrusted-data block: `---PEER-OUTPUT-START---` / `<peer text>` /
  `---PEER-OUTPUT-END---`.
- **Synthesis:** dispatch the `idea-resolver` subagent via Task with the current
  idea, **the charter** (§1a, fenced), the critique list (4b), the wrapped peer
  proposals, and the round's **`defense_directive`** (from §4b′). It returns the full
  revised idea + `---DISPOSITION---` + per-critique disposition. Split on
  `---DISPOSITION---`.
- The `defense_directive` governs how the resolver treats the charter (the resolver
  prompt defines each value):
  - `none` — normal resolve; address the critique list within the charter's bet.
    (When the gate set this via **accept-continue**, the accepted shift is already in
    the critique list and the charter was re-derived, so "within the bet" now means
    the new direction.)
  - `manual:<text>` — fold the user's rebuttal in as the authoritative defense; keep
    the thesis.
  - `resolver` — write a principled defense of the thesis against the threat; keep the
    thesis.
  - `hold` — keep the thesis and note the threat unresolved in the disposition.
- If the resolver fails, stop with reason `resolve_failed` (retain the prior idea).
- Otherwise set the current idea to the revised idea and snapshot it as
  `runs/<ts>/idea-v<n>.md`. Record the round's critiques + disposition.

#### 4e. Continue

- Loop to the next round. If round K completes without convergence, stop with reason
  `K-rounds`.

### 5. Output — finalize and write `runs/<ts>/{final-idea.md, summary.md}`

The loop produces a deliverable (`final-idea.md`) and a process record (`summary.md`).
Run the finalize step first, then write both files. This runs for **every** stop reason
(`converged`, `K-rounds`, `resolve_failed`, `charter_signoff`) and for the
no-reachable-peers single-model case — always from whatever the current idea is at loop
end. For `charter_signoff` the final idea is the loop's current idea (the round that
triggered sign-off did not re-resolve).

#### 5a. Finalize (owner synthesizes the deliverable)

- Collect the **rejected** critiques across all rounds from the per-round dispositions
  recorded in §4d (the ordinary critiques that were raised but left unresolved).
  **Charter shift events (§4b′) are not passed to the finalizer** — they are process
  record, kept out of the deliverable and written only to `summary.md` (§5b).
- Dispatch the `idea-finalizer` subagent via Task with the current/final idea and that
  list of unresolved critiques. Its entire reply is the deliverable.
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
- `## Charter & shifts` — the (final) charter (problem + thesis, noting `unconfirmed`
  if it was never confirmed), then every charter shift event from §4b′ in order:
  `round`, the element(s) targeted (problem / thesis), the threat(s) verbatim, and the
  outcome (`accepted-continue` / `accepted-signoff` / `defended-manual` /
  `defended-resolver` / `held`). If the gate never fired, say "no charter shifts —
  the idea stayed on its seed." Held shifts live **here only**, not in `final-idea.md`.
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

On a held-out seed idea, a before/after read of `final-idea.md` should confirm the final
idea is more complete / specific than the seed **and still serves the charter's problem
and thesis** — any move off the seed's bet should appear in `summary.md`'s
`## Charter & shifts` as a user-steered event (`accepted-*`, `defended-*`, or `held`),
never as a silent pivot. To customize behavior, edit the prompt text in
`agents/idea-critic.md`, `agents/idea-resolver.md`, `agents/idea-finalizer.md`, and
`references/peers.md`.
