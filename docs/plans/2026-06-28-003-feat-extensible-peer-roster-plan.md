---
title: "feat: extensible peer roster — add other AI models, keep Claude+codex+agy default"
type: feat
date: 2026-06-28
artifact_contract: ce-unified-plan/v1
artifact_readiness: implementation-ready
execution: code
product_contract_source: ce-plan-bootstrap
---

# feat: extensible peer roster

**Target repo:** `idea-polisher-cc` — **this** repository. All paths are repo-relative. Builds directly on the bundle plan (`docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md`), which left "configurable peers" out of v1 scope.

## Summary

Today `skills/idea-polish/references/peers.md` carries a fixed two-row command table (`codex`, `agy`) and there is no documented way to add a peer or to run a subset. The data table is technically already editable and the loop already iterates "each reachable peer" — so the real gaps are an **undocumented add-path** and **all-or-nothing peer selection**, not literal hard-coding. This change closes both: an editable **peer roster** plus a documented **peer contract** so any conforming CLI joins by editing one table, and opt-in flags (`--peers`, `--with`, `--without`) to select which registered peers run. With no flags, the run is byte-for-byte the current behavior (Claude host + reachable `codex`/`agy`). The plugin ships no new CLIs registered — it ships the contract and lets users add their own.

The coordinator loop already iterates over "each reachable peer" (`SKILL.md` §4a/§4d), so the loop structure, convergence quorum, and prompt templates are unchanged. The per-peer invocation is hardened so the safe prompt-passing the two fixed rows had is enforced for every peer (the `command` is args-only; the coordinator appends the prompt). This plan adds **peer-set resolution** at intake, the **args-only / coordinator-appended invocation**, and **roster/contract docs**.

**Product Contract preservation:** N/A — direct planning (`ce-plan-bootstrap`). Loop/convergence/security behavior inherited unchanged from the bundle plan.

---

## Requirements

| ID | Requirement |
|----|-------------|
| R1 | `peers.md` carries an editable peer **roster** table; `codex` and `agy` are marked default-on. The roster `command` column is **args-only** (everything before the prompt) plus an `input` column (`arg` \| `stdin`); the coordinator appends the prompt itself (see R6). Claude (host/owner) is unchanged, is never a roster row, and is never a valid flag value. |
| R2 | `peers.md` documents a peer **contract** (the interface a CLI must satisfy) so a new model is added by editing the roster alone — no `SKILL.md` loop changes. The contract makes the prompt-passing safety **enforced by construction**, not advisory (R6). |
| R3 | Selection flags resolve the effective peer set: base = `--peers <list>` if given, else the default-on roster peers; then add `--with <names>`, remove `--without <names>`. No flags ⇒ current behavior exactly. |
| R4 | A peer name in any flag that is not in the roster (including `claude`) ⇒ a clear error that lists the registered peer names; the run does not silently ignore typos. |
| R5 | A peer the user **explicitly named** (`--peers`/`--with`) that is unreachable ⇒ a prominent warning; a default peer that is merely absent ⇒ the existing quiet skip. Either way the run degrades gracefully. Resolution retains the explicitly-named set so §2 can tell the two cases apart. |
| R6 | The coordinator never executes a user-authored full command line and never interpolates the idea/prompt into the roster string: for every selected peer it runs `<args> "$(cat .peer-prompt.txt)"` (input `arg`) or `<args> < .peer-prompt.txt` (input `stdin`). Critic/proposal prompt templates, the `---VERDICT-JSON---` contract + convergence quorum, and untrusted-output wrapping + run-folder `cwd` apply unchanged to every selected peer; the per-peer connection-test *mechanism* is unchanged (only the set it iterates changes — see U2). |
| R7 | A peer counts toward the convergence quorum only if it reliably emits a parseable verdict. A peer whose verdict always parses as `constructive: true` will prevent convergence (every parsed verdict must be non-constructive); the contract documents this and recommends verifying format compliance before relying on a peer, keeping weak peers default-off. |
| R8 | README documents the three flags and points to the "Adding a peer" contract section. |

---

## Key Technical Decisions

- **KTD1 — Extensibility via data + docs, not code.** A roster table plus a contract section is the whole feature. The loop already fans out over the reachable set, so adding a model never touches orchestration. This is the laziest correct shape and the one the user chose ("contract only").
- **KTD2 — One resolution pipeline.** `base → +with → −without`, where `base` is `--peers` (if given) else the default-on peers. A single mental model that composes; `--peers` combined with `--with`/`--without` is well-defined (adjustments apply on top of the explicit base) rather than an error case.
- **KTD3 — Zero behavior change at the default.** `codex`+`agy` stay default-on; Claude stays the host/owner. A no-flags run is identical to today's — backward compatible by construction.
- **KTD4 — Contract-only, no bundled extras (user choice).** The plugin documents the contract and ships only the two existing peers registered. It does not pre-register CLIs it hasn't tested, so it makes no quality claims about command shapes it can't vouch for.
- **KTD5 — Explicit vs. default unreachability are different events.** A peer the user named and the CLI isn't there is likely a mistake worth a loud warning; a default peer simply not installed is the normal degradation path and stays quiet. Both still let the run proceed.
- **KTD6 — Prompt-passing safety is enforced, not advisory.** Because the roster `command` is now user-authored data executed via the Bash tool (the old table was two fixed, audited commands), the safe invocation cannot be left to "the user's CLI must be safe." The coordinator owns the prompt position: it appends `"$(cat .peer-prompt.txt)"` or pipes `< .peer-prompt.txt` to an **args-only** command, and never runs a user-written full command line. A CLI that can take the prompt as neither a trailing arg nor stdin is unsupported rather than hand-wired. This keeps the command-injection guarantee that the two hard-coded rows had.

---

## Implementation Units

### U1. Peer roster + "Adding a peer" contract

- **Goal:** Replace the fixed 2-row command table in `peers.md` with an editable roster and a documented, security-enforced contract.
- **Requirements:** R1, R2, R6, R7.
- **Dependencies:** none.
- **Files:** `skills/idea-polish/references/peers.md`.
- **Approach:** Reshape the existing `## Commands` table into a `## Peer roster` table with columns: `name`, `command` (**args-only** — everything before the prompt; this is where per-peer model/effort pinning lives), `input` (`arg` for a trailing-arg CLI, `stdin` for a pipe-fed one), `default` (✓ for `codex` and `agy`). The coordinator — not the row — supplies the prompt position (R6/KTD6), so a row never contains the idea text or its substitution. Add a `## Adding a peer` section stating the contract: (1) the CLI takes the prompt as a trailing positional arg (`input: arg`) **or** from stdin (`input: stdin`) and prints its reply to stdout — a CLI that can do neither is unsupported; (2) exits 0 on success — non-zero/timeout/missing ⇒ skipped; (3) **safety is enforced by the coordinator**: it appends `"$(cat .peer-prompt.txt)"` or pipes `< .peer-prompt.txt` to your args-only command, so you never write the prompt into the command and the idea is never shell-interpolated; (4) for critique, the reply ends with the `---VERDICT-JSON---` line + `{constructive, critiques, clarifications}` object — and only a parseable verdict counts toward convergence (R7): an unparseable one is kept as unstructured critique but excluded from the quorum, and a peer that always returns `constructive: true` will keep the loop from converging, so verify a new peer with the connection test and leave weak peers default-off. Carry the existing `--dangerously-skip-permissions` blast-radius warning **into this section** (not just the README): each added peer is another agent running on the idea text at that trust level and `cwd` is not a sandbox. Include one worked example row. Leave `## Security posture`, `## Connection test`, `## Graceful degradation`, `## Verdict parsing`, and `## Prompt templates` unchanged — they are already peer-agnostic.
- **Patterns to follow:** the current `peers.md` table + section structure; the existing model/effort-pinning reminder; the existing `## Security posture` file/stdin and blast-radius wording.
- **Test scenarios:** `Test expectation: doc structural check` — the roster table is valid Markdown with `codex`/`agy` marked default and an `input` value each; the contract section enumerates the four rules, the convergence caveat, and the blast-radius warning; the worked example row is args-only with an `input` value (no prompt token in the command).
- **Verification:** a reader can add a new peer by following only `## Adding a peer` without reading `SKILL.md`, and the documented invocation never places the prompt inside the user-authored command string.

### U2. Peer-set selection in the coordinator

- **Goal:** Resolve the effective peer set from flags at intake, then probe and run only that set.
- **Requirements:** R3, R4, R5, R6.
- **Dependencies:** U1.
- **Files:** `skills/idea-polish/SKILL.md`.
- **Approach:** Extend §1 Intake to parse `--peers <comma-list>`, `--with <comma-list>`, `--without <comma-list>`. Resolve per KTD2: `base = --peers if present else roster default-on peers; selected = (base + --with) − --without`. **Retain the explicitly-named set** (the union of `--peers` and `--with` names) alongside `selected`, because §2 needs it to tell a named-but-missing peer from a default-but-absent one (R5) — the flat `selected` set alone cannot. Validate every name in any flag against the roster (R4), including `claude` (the host is not a roster peer) — unknown name ⇒ stop with an error listing registered names. Update §2 Connection test to probe **the selected set** (not "each peer"): drop+report unreachable peers, but distinguish explicitly-named-unreachable (prominent warning, R5) from default-absent (existing quiet `! <peer>: unreachable — skipped`). §4a/§4d ("for each reachable peer") keep their iteration, but the per-peer invocation now reads the roster row's `command` + `input` and the coordinator appends the prompt itself per R6/KTD6 (this is documented in `peers.md` by U1, which §4 already points to). Note in §2 that with no reachable peers the run still proceeds as single-model self-review (existing behavior).
- **Technical design (directional, not spec):**
  ```
  base      = parse(--peers) or roster.default_on        # ["codex","agy"]
  named     = parse(--peers) + parse(--with)             # explicit subset, kept for §2
  selected  = (base + parse(--with)) - parse(--without)
  for name in (named + parse(--without)):
      if name not in roster: error("unknown peer '<name>'; registered: ...")
  # §2: probe `selected`. unreachable & in `named` -> loud warn; else quiet skip.
  # §4: invoke each reachable peer as  <command> "$(cat .peer-prompt.txt)"   (input=arg)
  #                                 or <command> < .peer-prompt.txt          (input=stdin)
  ```
- **Execution note:** the no-flags path must reduce to exactly today's behavior — verify that first.
- **Patterns to follow:** the existing §1 flag parsing (`--rounds`, `--resolve-first`, `--timeout`) and §2 degradation wording.
- **Test scenarios:**
  - No flags ⇒ selected = reachable {codex, agy}; run is identical to current behavior.
  - `--without agy` ⇒ selected = {codex}; agy never probed.
  - `--with gemini` (a roster row the user added) ⇒ selected = {codex, agy, gemini}.
  - `--peers codex,gemini` ⇒ base override = {codex, gemini}; agy excluded.
  - `--peers codex --with gemini --without codex` ⇒ selected = {gemini} (pipeline composes).
  - Unknown name (`--with foo`, not in roster) ⇒ clear error listing registered peers; loop does not start.
  - Explicitly named but unreachable (`--with gemini`, CLI absent) ⇒ prominent warning, run proceeds with the rest.
  - Default peer absent (no flags, `agy` not installed) ⇒ quiet skip, run proceeds.
- **Verification:** each scenario above produces the stated selected set and message; a no-flags run matches the pre-change run.

### U3. README — flags + contract pointer

- **Goal:** Document selection and point users at the contract for adding models.
- **Requirements:** R8.
- **Dependencies:** U2.
- **Files:** `README.md`.
- **Approach:** In Usage, add the three flags with one-line semantics and the resolution rule (base = `--peers` or defaults, then `+--with`/`−--without`). State that the default set is the **Claude host (always on, not a selectable peer)** plus the default-on peers `codex`+`agy`, that flags select among **peers only** (so `--without claude` / `--peers claude` are not valid — `claude` is not a roster name), and that no flags = current behavior. Add a short "Adding a model" note linking `skills/idea-polish/references/peers.md` § Adding a peer. Reinforce that any added peer inherits the same `--dangerously-skip-permissions`/untrusted-data security posture (cross-link the existing security warning, which the contract section also now carries).
- **Test scenarios:** `Test expectation: docs match behavior` — the flags and resolution rule in the README match U2; the contract link resolves; the security note is present.
- **Verification:** a user can go from the README to adding a working peer using only the linked contract.

---

## Scope Boundaries

**In scope:** an editable peer roster + documented contract, opt-in selection flags, README/docs, all reusing the existing loop and security posture.

### Deferred to Follow-Up Work
- **Bundling curated extra CLIs** (gemini-cli, `llm`, `ollama`) registered out of the box — the user chose contract-only for v1.
- **Config-file / persistent per-project roster** — flags are the v1 selection mechanism.
- **Non-Claude owner / configurable resolver** — already deferred by the bundle plan; unchanged here.
- **Parallel peer calls** — sequential Bash remains fine.

### Out of scope
- Re-deciding loop, convergence, prompt, or owner behavior — inherited unchanged.
- Building per-peer output adapters for CLIs that don't meet the contract — non-conforming CLIs degrade to unstructured critique via the existing fallback, no special-casing.

---

## Risks & Dependencies

- **User-authored peer command is the new injection surface (medium→high).** Unlike the two fixed, audited rows, a roster `command` is now user-authored data executed via the Bash tool; a hand-written full command line could re-introduce shell interpolation of the idea. This *is* a new surface — the earlier "no new code path" framing was wrong. *Mitigation:* enforce safety by construction (R6/KTD6) — the `command` column is args-only, the coordinator owns the prompt position (`"$(cat .peer-prompt.txt)"` or stdin), it never runs a user-written full command line, and CLIs that fit neither invocation are unsupported. Untrusted-output wrapping and run-folder `cwd` apply to every peer; the blast-radius warning now lives in the contract (U1) and README (U3).
- **Latency scales with peer count (low→medium).** Each reachable peer adds two sequential Bash calls per round (critic + proposal), each up to the per-call timeout, across up to K rounds; making peers easy to add invites large sets. *Mitigation:* note in the contract/README that wall-clock ≈ peers × rounds × 2 × timeout, suggest keeping the active set small or lowering `--rounds`, and flag parallel peer calls (already deferred) as the upgrade path.
- **Flag-resolution ambiguity (low).** Combining `--peers` with `--with`/`--without` could confuse. *Mitigation:* one documented pipeline (KTD2) with a worked precedence example in tests; unknown names error loudly (R4).
- **Prose-orchestrated selection (low).** The resolution lives in skill instructions, not code. *Mitigation:* explicit pipeline pseudocode in U2 + enumerated selection test scenarios; the `idea_polisher` CLI remains the deterministic fallback.
- **Silent default drift (low).** A future edit could accidentally change which peers are default-on. *Mitigation:* the no-flags-equals-today scenario (U2) is the guard.

---

## Open Questions

- **Is the full three-flag set-algebra justified now?** (Raised in review.) At today's 2-peer default, a not-installed peer is already silently skipped, and toggling the roster `default` column gives coarse selection with no new flag surface. A leaner v1 could ship `--without` only (the one flag meaningful at 2 peers) and add `--peers`/`--with` once a third peer is actually registered. The user chose opt-in flags deliberately — keeping all three is defensible since the feature exists to grow the peer set, but this is the one scope decision worth a second look before U2.
- **Quality bar for a registered peer** — beyond format compliance (R7), is there a desired check (e.g. the connection test confirming a parseable verdict) before a peer is allowed to count toward the convergence quorum, or is "leave weak peers default-off" guidance enough?
- **`--peers`/`--with`/`--without` naming** — these read clearly, but confirm at implementation they don't collide with any existing or planned flag.
- **Worked example peer in the contract** — pick a concrete CLI for the example row (e.g. a generic `llm`-style command) without implying it's bundled/tested.

---

## Sources & Research

- Local: `skills/idea-polish/references/peers.md`, `skills/idea-polish/SKILL.md`, `README.md`, and the bundle plan `docs/plans/2026-06-28-002-…-plan.md` — the loop already fans out over the reachable peer set, which is what makes this a data+docs change. No external research run (strong local patterns; not requested).
