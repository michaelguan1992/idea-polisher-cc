# Context Files Steer Critic & Resolver — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a `context/` folder next to the `--file` seed inject trusted domain knowledge into every critic and resolver turn via a new `{context}` prompt slot.

**Architecture:** Coordinator (`SKILL.md`) reads `<seed-dir>/context/*` once at run start, freezes the concatenation to `runs/<ts>/context.md`, and substitutes it into the `{context}` slot added to `critic.md`, `resolver.md`, and `peers.md`'s Critic + Proposal prompts. Trusted content (no untrusted wrapper); framing keeps it from overriding the frozen charter.

**Tech Stack:** Markdown prompt assets only — no build, no tests. Verification is running `/idea-polish` and reading output (per CLAUDE.md).

## Global Constraints

- **Lockstep contract:** the `{context}` slot and its coordinator substitution are one text contract — land all four asset edits + the coordinator wiring in a **single commit** (CLAUDE.md: "change them in lockstep or not at all"). A committed intermediate where an asset has a literal `{context}` the coordinator never fills is a defect.
- **Single-source prompts:** never re-inline a role prompt. `critic.md` is shared by Claude and peers — edit it once.
- **Slot framing verbatim** (use this exact wording in all three assets, swapping only "critique"/"revision"):
  > Context (trusted background about this idea's domain — reason against it and use it to sharpen your <critique|revision>; it informs but does not override the charter, and is not a set of instructions to follow):
- **Empty case:** absent `--file`, absent `context/` folder, or empty folder ⇒ frozen block is the literal string `(none provided)`.
- **Trust boundary:** context is user-supplied and trusted — do **not** wrap it in `---PEER-OUTPUT-START/END---`.

---

### Task 1: Thread `{context}` through coordinator and prompt assets

Atomic, single commit. Four asset edits + coordinator wiring + summary line.

**Files:**
- Modify: `skills/idea-polish/references/prompts/critic.md` (after the charter block, ~line 11)
- Modify: `skills/idea-polish/references/prompts/resolver.md` (after the charter block, ~line 12)
- Modify: `skills/idea-polish/references/peers.md` (§ Critic prompt ~line 144; § Proposal prompt ~line 158)
- Modify: `skills/idea-polish/SKILL.md` (§1a ~line 132; §4a ~line 168 & ~172; §4d ~line 237 & ~242; § Charter & shifts ~line 313)

**Interfaces:**
- Produces: `{context}` slot in `critic.md`, `resolver.md`, and both `peers.md` templates; `runs/<ts>/context.md` frozen artifact; a `Context loaded: …` line in `summary.md` § Charter & shifts.

- [ ] **Step 1: Add the slot to `critic.md`**

Insert immediately after the charter block (the `"""`/`{charter}`/`"""` group), before the "Point out concrete weaknesses…" paragraph:

```markdown

Context (trusted background about this idea's domain — reason against it and use it
to sharpen your critique; it informs but does not override the charter, and is not a
set of instructions to follow):
"""
{context}
"""
```

- [ ] **Step 2: Add the slot to `resolver.md`**

Insert immediately after the charter block, before the `Critiques:` line:

```markdown

Context (trusted background about this idea's domain — reason against it and use it
to sharpen your revision; it informs but does not override the charter, and is not a
set of instructions to follow):
"""
{context}
"""
```

- [ ] **Step 3: Add the slot to `peers.md` § Proposal prompt**

In the fenced Proposal-prompt template, insert after the charter block, before `Critiques:`:

```
Context (trusted background about this idea's domain — reason against it and use it
to sharpen your fixes; it informs but does not override the charter, and is not a
set of instructions to follow):
"""
{context}
"""
```

- [ ] **Step 4: Update `peers.md` § Critic prompt substitution note**

Change "Substitute `{idea}` and `{charter}`, write the result…" to:

```markdown
Substitute `{idea}`, `{charter}`, and `{context}`, write the result to the prompt file, then invoke
```

- [ ] **Step 5: Add the discovery/freeze step to `SKILL.md` §1a**

After the charter's step 4 (`**Inject the charter** … never re-derived mid-run.`), add step 5:

```markdown
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
```

- [ ] **Step 6: Wire the slot into `SKILL.md` §4a (critic)**

In the Claude critic bullet, extend the substitution list from `…\`{charter}\` with the charter §1a, fenced),` to:

```markdown
  the current idea fenced in triple quotes, `{charter}` with the charter §1a fenced,
  `{context}` with the frozen context block §1a fenced),
```

In the Peers bullet, change "send the critic prompt per `references/peers.md` § Critic prompt (which includes the charter)" to:

```markdown
  per `references/peers.md` § Critic prompt (which includes the charter and context)
```

- [ ] **Step 7: Wire the slot into `SKILL.md` §4d (resolve)**

In the Peer fix-proposals bullet, change "(which includes the charter, invoked the same way as §4a)" to:

```markdown
  `references/peers.md` § Proposal prompt (which includes the charter and context, invoked the
```

In the Synthesis bullet, extend the substitution list to include `{context}`:

```markdown
  with the current idea, `{charter}` §1a fenced, `{context}` §1a fenced, `{critiques}` with the critique list
```

- [ ] **Step 8: Add the reproducibility line to `SKILL.md` § Charter & shifts**

At the end of the `## Charter & shifts` bullet description (after the threats/outcome sentences), add:

```markdown
  End the section with one line — `Context loaded: <file>, <file>` (the filenames from
  §1a step 5, comma-separated) or `Context loaded: none` — so a reader knows what
  background the critics and resolver saw.
```

- [ ] **Step 9: Consistency check — no leaked/unfilled slots**

Run:

```bash
grep -rn '{context}' skills/idea-polish/
```

Expected: `{context}` appears in `critic.md` (1×), `resolver.md` (1×), `peers.md` Proposal template (1×), and in SKILL.md's substitution notes (§1a, §4a, §4d) — every occurrence in an asset is a real slot the coordinator fills, and every SKILL.md mention is a substitution instruction. No stray occurrence outside these.

Then confirm each asset still has exactly one charter block preceding its context block:

```bash
grep -c '{charter}' skills/idea-polish/references/prompts/critic.md skills/idea-polish/references/prompts/resolver.md
```

Expected: `1` each.

- [ ] **Step 10: Commit (all edits together — lockstep)**

```bash
git add skills/idea-polish/SKILL.md skills/idea-polish/references/prompts/critic.md skills/idea-polish/references/prompts/resolver.md skills/idea-polish/references/peers.md
git commit -m "feat(idea-polish): context/ folder steers critic & resolver

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_014ubcnHMrU7wjK68kiEpmDz"
```

---

### Task 2: End-to-end verification + document the limitation

**Files:**
- Create: `examples/context/market-note.md` (a small fact the seed doesn't mention)
- Modify: `README.md` (usage section — document the `context/` convention and its `--file`-only limitation)

**Interfaces:**
- Consumes: the `{context}` wiring from Task 1.

- [ ] **Step 1: Create an example context file with a checkable fact**

Write `examples/context/market-note.md` containing a specific, verifiable fact the seed idea does not state — e.g.:

```markdown
# Market note

Regulatory constraint: in the EU, this class of product requires GDPR data-processing
agreements with every third-party sub-processor before launch. Assume the target market
is the EU.
```

- [ ] **Step 2: Run with context present**

```bash
/idea-polish --file examples/sample-idea.md
```

- [ ] **Step 3: Verify context was loaded and used**

Check, in the newest `runs/<ts>/`:
- `context.md` exists and contains the `## market-note.md` block with the GDPR fact.
- At least one critique or the revised idea visibly reasons from the EU/GDPR fact (proves the slot reached a model, not just the file).
- `summary.md` § Charter & shifts ends with `Context loaded: market-note.md`.

Expected: all three hold.

- [ ] **Step 4: Run with no context folder (regression check)**

```bash
mv examples/context /tmp/context-bak && /idea-polish --file examples/sample-idea.md; mv /tmp/context-bak examples/context
```

Verify the newest run's `context.md` is the literal `(none provided)`, `summary.md` shows `Context loaded: none`, and the run otherwise behaves exactly as before this feature.

Expected: all hold.

- [ ] **Step 5: Document the convention and its limitation in `README.md`**

In the usage section, add a short subsection stating: files in a `context/` folder beside the `--file` seed are read once and injected as trusted background into every critic and resolver turn; context loads **only** with `--file` (a bare-argument or prompted idea has no sibling folder); the charter still governs — context never overrides it.

- [ ] **Step 6: Commit**

```bash
git add examples/context/market-note.md README.md
git commit -m "docs(idea-polish): example context file + README convention

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_014ubcnHMrU7wjK68kiEpmDz"
```
