# Peer models (Codex, Antigravity) — invocation reference

> **Scaffold — implement per plan U4.** Carry the exact behavior from the
> `idea_polisher` reference implementation (`idea_polish.py`: `MODELS`,
> `run_model`, `connection_test`).

To fill in:

- **Commands** — Codex: `codex exec --skip-git-repo-check "<prompt>"`; Antigravity: `agy --print --dangerously-skip-permissions "<prompt>"`. Pass the prompt as a single argument (never a shell string).
- **Connection test** — probe each with a trivial prompt; treat non-zero exit / timeout / missing binary as unreachable; drop unreachable peers for the run.
- **Graceful degradation** — the owner (Claude host) is required; peers are best-effort. A failed peer call is skipped for that step.
- **Security posture** — run peer subprocesses with `cwd` scoped to the run folder so an auto-approved agent's writes stay contained; document the `--dangerously-skip-permissions` blast-radius warning in the README.
- **Verdict parsing** — peers must emit the `---VERDICT-JSON---` delimiter + JSON verdict; parse only after the last delimiter; parse-failures are excluded from the convergence quorum.
