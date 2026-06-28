# idea-polisher-cc

A Claude Code plugin that polishes a single idea via a multi-round **cross-model
debate** — Claude critiques and resolves natively, while Codex and Antigravity
(Gemini) join as peer critics and fix-proposers via their CLIs.

It is the native Claude Code form of the
[`idea_polisher`](https://github.com/michaelguan1992/idea_polisher) CLI, which
remains the deterministic **reference implementation** and fallback.

> **Status: scaffold.** The plugin skeleton and design plan are in place; the
> skill and subagent bodies are stubs to be implemented per
> [`docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md`](docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md).
> Run `/ce-work docs/plans/2026-06-28-002-feat-idea-polisher-cc-bundle-plan.md`
> from this repo to build it out (units U1–U5).

## What it will do

- `/idea-polish "<idea>"` runs a critic↔resolver loop until no critic has a
  constructive critique left, or after K rounds.
- Every available model critiques the idea each round; Claude (the host, and the
  idea **owner**) synthesizes one revised idea from the critiques + peer
  fix-proposals.
- Produces a `summary.md` with the polished idea, a per-round critique digest
  with dispositions, the participating models, and the stop reason.

## Requirements (when built)

- **Claude Code** (the host — provides the owner/resolver turn natively).
- Optional peers for genuine cross-model value: `codex` (`codex exec`) and `agy`
  (`agy --print`), installed and authenticated. With Claude only, the tool
  degrades to a single model reviewing its own idea.

## Layout

```
.claude-plugin/   plugin.json + marketplace.json (install/distribution)
skills/idea-polish/   coordinator skill (the loop)  [stub]
agents/           idea-critic, idea-resolver subagents  [stubs]
docs/plans/       the implementation plan
examples/         a sample seed idea
```

## License

MIT — see [LICENSE](LICENSE).
