# idea-polisher-cc

A Claude Code plugin that polishes a single idea via a multi-round **cross-model
debate** — Claude critiques and resolves natively, while Codex and Antigravity
(Gemini) join as peer critics and fix-proposers via their CLIs.

It is the native Claude Code form of the
[`idea_polisher`](https://github.com/michaelguan1992/idea_polisher) CLI, which
remains the deterministic **reference implementation** and fallback. The hardened
design is carried over from that project's reviewed plan
([`2026-06-28-001-…-plan.md`](https://github.com/michaelguan1992/idea_polisher/blob/main/docs/plans/2026-06-28-001-feat-idea-polisher-debate-cli-plan.md)).

## What it does

- `/idea-polish "<idea>"` runs a critic↔resolver loop until no critic has a
  constructive critique left, or after K rounds (default 3).
- Every reachable model critiques the idea each round; Claude (the host, and the
  idea **owner**) synthesizes one revised idea from the critiques + peer
  fix-proposals.
- Writes a `summary.md` with the polished idea, a per-round critique digest with
  dispositions, the participating models, and the stop reason, plus per-round idea
  snapshots.

## Install

```
/plugin marketplace add michaelguan1992/idea-polisher-cc
/plugin install idea-polisher-cc
```

(Or add this repo's local path as a marketplace during development — the
`marketplace.json` `source` is `./`.)

## Usage

```
/idea-polish "<your idea>"
/idea-polish --file path/to/idea.md
/idea-polish "<your idea>" --rounds 2
/idea-polish "<your idea>" --resolve-first
```

- `--rounds N` — max critique/resolve rounds (default 3).
- `--resolve-first` — skip entry classification; resolve before the first critique
  (useful when the idea text already contains its own critiques).

Output lands in `runs/<timestamp>/` under your current directory: `summary.md`, the
`idea-v*.md` snapshots, and per-round `critiques-*.json`.

## Requirements

- **Claude Code** — the host. Provides the owner/resolver turn natively; always
  available.
- **Optional peers** for genuine cross-model value: `codex` (`codex exec`) and
  `agy` (`agy --print`), installed and authenticated. Reachable peers are detected
  automatically; missing ones are skipped.
- **You need ≥2 reachable models for a real cross-model run.** With Claude only,
  the tool degrades to a single model reviewing its own idea — still useful, but
  not the cross-model debate the plugin is for.

## ⚠️ Security warning

Peer calls run `agy` with `--dangerously-skip-permissions`, i.e. an AI agent that
**auto-approves its own actions** on your machine while reasoning about your idea
text. The plugin contains the blast radius as much as it can — the idea is passed
without shell interpolation, peer output is treated as untrusted data, and peer
processes run with the run folder as their working directory — **but cwd is not a
sandbox**: a skip-permissions agent can still read and write absolute paths
(`~/.ssh/...`, etc.). Only run with peers you trust, on idea text you are willing
to expose to them. If that is not acceptable, run Claude-only (peers absent) or use
the deterministic `idea_polisher` CLI.

## Customizing

The prompts are the product. To tune behavior, edit the prompt text in:

- `agents/idea-critic.md` — the critique turn + verdict contract.
- `agents/idea-resolver.md` — the synthesis turn + disposition contract.
- `skills/idea-polish/SKILL.md` + `skills/idea-polish/references/peers.md` — the
  loop, the peer commands, and the peer prompt templates.

## Acceptance check

On a held-out seed idea (see [`examples/sample-idea.md`](examples/sample-idea.md)),
a before/after read of `summary.md` should confirm the final idea is more complete
and specific than the seed.

## Layout

```
.claude-plugin/        plugin.json + marketplace.json (install/distribution)
skills/idea-polish/    coordinator skill (the loop) + references/peers.md
agents/                idea-critic, idea-resolver subagents
docs/plans/            the implementation plan
examples/              a sample seed idea
```

## License

MIT — see [LICENSE](LICENSE).
