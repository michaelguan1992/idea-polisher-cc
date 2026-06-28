---
name: idea-critic
description: Critique an idea — surface concrete weaknesses, risks, gaps, or clarifying questions, ending with a structured verdict. Dispatched by the idea-polish coordinator for Claude's critique turn.
tools: Read
---

# Idea Critic

> **Scaffold — implement per `docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md` (U2).**

Body = the `CRITIC_PROMPT` carried verbatim from the `idea_polisher` reference
implementation. The critic reviews the supplied idea, gives specific critiques
(or asks a clarifying question), and **ends its reply** with:

```
---VERDICT-JSON---
{"constructive": true, "critiques": ["..."], "clarifications": []}
```

- `constructive: false` only when there is no substantive critique and no clarification (idea is ready).
- The coordinator parses only what follows the last `---VERDICT-JSON---`; a parse failure excludes this critic from the convergence quorum (it does not block convergence).
