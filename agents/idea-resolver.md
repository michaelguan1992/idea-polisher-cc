---
name: idea-resolver
description: Synthesize a revised idea from the aggregated critiques and peer fix-proposals, with a per-critique disposition. Dispatched by the idea-polish coordinator for the owner's resolution turn.
tools: Read
---

# Idea Resolver

> **Scaffold — implement per `docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md` (U3).**

Body = the `RESOLVER_PROMPT` carried verbatim from the `idea_polisher` reference
implementation. Given the current idea, the aggregated critiques, and peer
fix-proposals, the resolver outputs the **full revised idea**, then:

```
---DISPOSITION---
- <critique>: addressed / rejected (with a short reason)
```

The coordinator splits the reply on `---DISPOSITION---` (idea before, disposition
after) and snapshots the revised idea as `idea-vN`.
