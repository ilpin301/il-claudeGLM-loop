# il-claudeGLM-loop

Plan hardening for Claude Code, with **GLM-5.3** as the read-only rival critic.

Two different models harden your plan before a line of code exists. GLM only critiques plans, across several rounds; Claude writes all code, always.

This is a rewrite of [claudex-loop](https://github.com/chaseai-yt/claudex-loop) by Chase AI (MIT), with OpenAI Codex replaced throughout by GLM-5.3 running headlessly through the Claude Code CLI against z.ai's Anthropic-compatible endpoint. See [THIRD-PARTY-NOTICES.md](./THIRD-PARTY-NOTICES.md).

## The four failure modes it kills

| Phase | Kills |
|-------|-------|
| **0 — RECON** | *Interviewing blind.* Claude scouts the codebase (or researches prior art, stack, and pitfalls on greenfield) and drafts an assumptions ledger before asking you anything. |
| **1 — INTERROGATE** | *Building the wrong thing.* Load-bearing decisions get questioned one at a time — each with why-it-matters, a committed recommendation, and what-breaks-if-we-guess-wrong. Cosmetic ones get batched. |
| **2 — REVIEW** | *A plan that sounds right but breaks.* GLM-5.3 attacks the locked plan read-only, severity-gated, across a persistent session that remembers its own prior critiques. |
| **3 — BUILD** *(optional)* | *Unverified work.* Claude implements the approved plan and proves it with its own tests/checks. GLM never builds or reviews code. |

You enter at four points only: confirming the ledger, answering the fire, signing off the converged plan, approving the final diff.

## Skills

| Command | What it does |
|---------|--------------|
| `/il-claudeGLM-loop:il-claudeGLM-loop` | The full four-phase loop. Start here for high-stakes work. |
| `/il-claudeGLM-loop:il-glm-review` | Standalone adversarial plan-review loop. Use when you already have a plan and just want the cross-model stress-test. |
| `/il-claudeGLM-loop:il-glm-build` | **DISABLED** (user rule 2026-09-29: GLM never writes code; Claude implements). Kept as historical reference only. |

`legacy/` keeps the two earlier GLM grill skills (`grill-me-glm`, `grill-with-docs-glm`) that this plugin supersedes.

## Prerequisites

- **Claude Code CLI** (`claude --version`) — it is the harness that runs the GLM side too.
- **A POSIX shell.** Git Bash on Windows, or macOS/Linux. The invocations use env-var prefixes, `/tmp`, and `< /dev/null`; they do **not** run in cmd or PowerShell.
- **A z.ai API key on a GLM Coding plan**, exported as `ZAI_API_KEY`:
  ```bash
  export ZAI_API_KEY="…"
  ```
  Endpoint `https://api.z.ai/api/anthropic`, model `glm-5.3`.
- **Run your main session as normal Claude (Anthropic).** If the whole session runs under GLM, both sides are GLM and the cross-model check is gone — that is the entire point of the tool.

### Why the key check is not optional

Every skill here preflights `[ -n "$ZAI_API_KEY" ]` and refuses to run without it.

Two things measured on the acceptance run make this checkable rather than hopeful:

- The response JSON carries `modelUsage` keyed by the model that actually served the request — a real GLM round shows `{"glm-5.3": {...}}`. That is objective proof of who reviewed, unlike asking the session to describe itself.
- A *wrong* key does not fall through to Claude: `glm-5.3` is not an Anthropic model, so the call fails with `unrecognized_model`. It fails by **hanging until the timeout** with an empty output file rather than erroring fast, so the skills treat an empty output as failure and print stderr instead of retrying.

## GLM is read-only

GLM gets `--allowedTools "Read Grep Glob"` and headless mode auto-denies everything else, so it is read-only by construction, round one and every resume alike. It never writes, edits or builds code and never reviews code diffs; Claude implements every change.

`il-glm-build` (which ran GLM with `--dangerously-skip-permissions`, machine-wide) is disabled and must not be invoked.

## Install

```bash
git clone https://github.com/ilpin301/il-claudeGLM-loop
```

Then add it as a plugin marketplace in Claude Code (`/plugin marketplace add <path-or-url>`) and install `il-claudeGLM-loop`.

## Tunables

Pass as skill args, e.g. `rounds=3`:

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop always terminates here. |
| `PLAN_FILE` | `PLAN.md` | The locked plan. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript — the real artifact. |
| `TIMEOUT_MS` | `600000` | Ceiling per headless call. |
| `research` | ask | `none` / `web` / `deep` — pre-answers the Phase 0 research gate. |

## What changed from the Codex original

- **Plan reviewer (read-only critic) is GLM-5.3** via `claude -p` against z.ai, not `codex exec`.
- **Read-only by whitelist, not by sandbox flag.** `--allowedTools "Read Grep Glob"` — and unlike Codex, there is no separate resume gotcha: the same flags apply to round 1 and every resume.
- **`.session_id` re-read every round**, never pinned — `--resume` can fork, and a stale id silently drops intermediate findings.
- **One skill bench, not two.** Codex read `~/.agents/skills` while Claude read `~/.claude/skills`. GLM runs inside the same CLI, so there is one directory and no cross-bench mismatch to report.
- **Severity-gated verdicts.** `VERDICT: REVISE` only for `[HIGH]`; `[MED]`/`[LOW]` ship as non-blocking follow-ups instead of stalling convergence.
- **Hardened invocation:** preflight key check, per-run stamped temp files (echoed once, because each orchestrator step is a new shell where `$$` differs), stderr kept in a sibling `.err` instead of `/dev/null`, `NO_PROXY="*"`, and a documented Git Bash `/tmp` ↔ Windows path gotcha.

## License

MIT — see [LICENSE](./LICENSE) and [THIRD-PARTY-NOTICES.md](./THIRD-PARTY-NOTICES.md).
