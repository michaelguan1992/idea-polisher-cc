---
name: idea-polish
description: Polish an idea via a multi-round cross-model critic/resolver debate (Claude + Codex + Antigravity). Use when the user wants an idea hardened by adversarial cross-model review.
---

# Idea Polish (coordinator)

This skill is the **orchestrator** and runs in the main agent context. It owns the
loop: it spawns the `idea-critic` / `idea-resolver` subagents via the Task tool for
Claude's own turns, and calls the peer CLIs (`codex`, `agy`) via Bash. Subagents
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
  after the **last** occurrence. Shape: `{"constructive": bool, "critiques": [str], "clarifications": [str]}`.
- **Disposition delimiter:** `---DISPOSITION---`. The resolver's reply is the revised
  idea before it, the per-critique disposition after it.
- **Parse failure:** a critic call that succeeded but whose verdict can't be parsed
  (no delimiter, or invalid JSON). Its raw text is still usable as an unstructured
  critique, but it is **excluded from the convergence quorum**.
- **Convergence quorum:** the loop has converged iff **at least one** verdict parsed
  AND **every parsed** verdict has `constructive: false` with no `clarifications`.
  Parse failures and failed calls do not count toward or against the quorum.

## Procedure

### 1. Intake

- **Idea:** from the command argument; else from a `--file <path>` argument; else
  ask the user for it.
- **K (max rounds):** `--rounds N`, default **10**.
- **Owner:** default `claude` (the host). A non-Claude owner is out of scope for v1.
- **`--resolve-first`:** if present, skip entry classification and resolve before the
  first critique.
- **Timeout:** per peer call, default **120s** (`--timeout`).
- **Run folder:** create `runs/<YYYYMMDD-HHMMSS>/` under the current working
  directory and use it for every prompt file, snapshot, and output below. All peer
  Bash calls use this folder as their working directory (see `references/peers.md`
  § Security). Write the original idea to `runs/<ts>/idea-v0.md`.

### 2. Connection test

- Claude (host/owner) is always present.
- Probe each peer per `references/peers.md` § Connection test. Drop any peer that is
  missing, errors, or times out, and report it: `! <peer>: unreachable — skipped`.
- The owner is **required**; peers are best-effort. If only Claude is reachable, the
  run still proceeds as single-model self-review — tell the user this is not a real
  cross-model run (see README).

### 3. Entry routing (done natively by you, the coordinator)

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
  (fenced in triple quotes). Its final message is the verdict block.
- **Peers:** for each reachable peer, send the critic prompt per `references/peers.md`
  § Critic prompt and capture stdout.
- For each result, parse the verdict (JSON after the last `---VERDICT-JSON---`):
  - call failed → log `! <model>: critique call failed — skipped this round`.
  - parsed → record the verdict.
  - succeeded but unparseable → log `! <model>: verdict unparseable — excluded from
    convergence quorum`, and keep its raw text as an unstructured critique.
- Save the round's parsed verdicts to `runs/<ts>/critiques-<n>.json`.

#### 4b. Convergence check

- Apply the convergence quorum. If converged → stop with reason `converged`
  (idea unchanged this round).
- Otherwise build the critique list for the resolver: one bullet per parsed critique
  as `- (<model>) <critique>`, plus `- (<model>, unstructured) <raw>` for each
  succeeded-but-unparseable critic. If there are none, use `(no specific critiques)`.

#### 4c. Clarifications (optional)

- Collect `clarifications` from all parsed verdicts. If any and the run is
  interactive, ask the user and append `Clarifications from the author:` + the Q/A
  to the critique context. In non-interactive runs, log that they were skipped.

#### 4d. Resolve (peers propose, owner synthesizes)

- **Peer fix-proposals:** for each reachable peer, send the proposal prompt per
  `references/peers.md` § Proposal prompt; collect the successful ones. Wrap each
  peer's output in an untrusted-data block:
  `---PEER-OUTPUT-START---` / `<peer text>` / `---PEER-OUTPUT-END---`.
- **Synthesis:** dispatch the `idea-resolver` subagent via Task with the current
  idea, the critique list (4b), and the wrapped peer proposals. It returns the full
  revised idea + `---DISPOSITION---` + per-critique disposition. Split on
  `---DISPOSITION---`.
- If the resolver fails, stop with reason `resolve_failed` (retain the prior idea).
- Otherwise set the current idea to the revised idea and snapshot it as
  `runs/<ts>/idea-v<n>.md`. Record the round's critiques + disposition.

#### 4e. Continue

- Loop to the next round. If round K completes without convergence, stop with reason
  `K-rounds`.

### 5. Output — write `runs/<ts>/summary.md`

Carry the reference summary shape:

- Header: participating models, stop reason.
- `## Final polished idea` — the current idea.
- `## Original idea` — the seed.
- `## Per-round digest` — for each round: critiques addressed and the resolver's
  disposition (or "Converged: no constructive critiques", or "Resolve step failed;
  prior idea retained").
- `## Idea snapshots` — list the `idea-v*.md` files, with the note: *revisions are
  not guaranteed monotonic — the last round may not be the best.*

Tell the user where `summary.md` landed and print the final polished idea.

## Acceptance check

On a held-out seed idea, a before/after read should confirm the final idea is more
complete / specific than the seed. To customize behavior, edit the prompt text in
`agents/idea-critic.md`, `agents/idea-resolver.md`, and `references/peers.md`.
