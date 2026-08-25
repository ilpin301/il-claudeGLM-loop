# Plan: il-claudeGLM-loop — rewrite chaseai-yt/claudex-loop with GLM-5.3 replacing OpenAI Codex

_Locked via grill-me-glm Act 1 — by Claude + ilpin301. Revised after GLM review round 1._

## Goal
Produce a standalone, publishable Claude Code plugin repo at `C:\Users\<user>\il-claudeGLM-loop` that is a faithful rewrite of `chaseai-yt/claudex-loop` with every OpenAI Codex mechanic replaced by GLM-5.3 running headlessly through the Claude Code CLI against z.ai's Anthropic-compatible endpoint. It ships three skills — `il-claudeGLM-loop` (four-phase plan hardening), `il-glm-review` (standalone adversarial plan-review loop), `il-glm-build` (role-flipped build where GLM writes and Claude verifies) — plus the two existing legacy GLM grill skills preserved under `legacy/`. After both loops pass a throwaway-repo smoke test, the superseded loose skills in `ilpin301/claude-skills` are retired so the plugin is the single home for GLM loop tooling.

## Approach
1. **Scaffold repo** at `C:\Users\<user>\il-claudeGLM-loop`: `git init`, MIT `LICENSE`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` (plugin name `il-claudeGLM-loop`, author ilpin301, keywords planning/glm/cross-model/adversarial-review/skills). No `assets/`.
2. **Port `skills/il-claudeGLM-loop/SKILL.md`** from upstream `skills/claudex-loop/SKILL.md` — the four phases kept intact (0 RECON, 1 INTERROGATE, 2 REVIEW, 3 BUILD), with all Codex CLI mechanics swapped for the GLM invocation contract below. Copy `ADR-FORMAT.md` and `CONTEXT-FORMAT.md` alongside.
3. **Port `skills/il-glm-review/SKILL.md`** from upstream `codex-review`, reusing the already-verified GLM invocation from the existing `~/.claude/skills/il-glm-review`, hardened per the contract below. This is the plugin-housed successor to that loose skill.
4. **Write `skills/il-glm-build/SKILL.md`** from upstream `codex-build`: frozen spec (`SPEC_FILE=PLAN.md`), clean-tree gate, explicit blast-radius warning, optional worktree isolation, prompt-via-tempfile contract, bounded fix loop, Claude reads the full diff and runs the proof itself, human gate before commit.
5. **Move legacy skills** `grill-me-glm` and `grill-with-docs-glm` verbatim (with their `THIRD-PARTY-NOTICES.md`) into `legacy/` of the new repo.
6. **Write `README.md`**: what the four phases kill, prerequisites (`ZAI_API_KEY`, Claude Code CLI, **Git Bash / POSIX shell required**, main session must be Anthropic Claude), install, the three commands, and the machine-wide-write warning for the build phase.
7. **Write `THIRD-PARTY-NOTICES.md`** at repo root: Chase AI / claudex-loop (MIT) as the upstream this is derived from; Matt Pocock / `grill-me` (MIT) for the interrogation pattern carried through the legacy skills.
8. **Install the plugin from the local path and prove it loads** — the three skills appear and their frontmatter parses. This happens **before** any retirement, so a malformed manifest can never leave the machine with neither old nor new tooling.
9. **Retire-first migration:** one commit in `ilpin301/claude-skills` removing `grill-me-glm`, `grill-with-docs-glm`, `il-glm-review`. Bump the submodule pointer in `~/.claude`.
10. **Run the acceptance gate** (below) in a throwaway git repo.
11. **On pass:** keep the retirement. **On fail:** `git revert` the retirement commit, fix, retest.
12. **After the gate passes,** ask the user before creating/pushing `ilpin301/il-claudeGLM-loop` on GitHub (visibility is their call at that moment).

## Key decisions & tradeoffs
- **Full publishable repo, not loose skills.** Mirrors upstream structure so the lineage is legible and the plugin installs on other machines. Cost: a second repo to maintain beside `claude-skills`.
- **`il-` prefix on all three skill names** (`il-claudeGLM-loop`, `il-glm-review`, `il-glm-build`), matching the user's personal convention. This deliberately collides by name with the loose `il-glm-review` — resolved by the retire-first ordering, not by renaming.
- **Retire first, then test — but only after the plugin is proven to load.** The old skills are removed from `claude-skills` before the smoke test so no name shadowing can make the test lie about which definition ran; the load-check in step 8 removes the "neither loads" window. Rollback is one `git revert`.
- **GLM gets real write access in `il-glm-build` via `--dangerously-skip-permissions`.** This is the user's explicit choice over a worktree jail, and the honest cost is stated rather than minimised: **the flag is machine-wide, not repo-scoped.** GLM can write outside the repo, run arbitrary shell, install packages, and make network calls; content in the repo or in `PLAN.md` is untrusted input that a permission-free agent may act on (prompt injection). The clean-tree gate makes *in-repo* changes isolatable and revertible — that is all it does, and the skill text must say so in those words. Mitigations shipped: the gate, an `isolate=worktree` opt-in tunable that runs the build in a throwaway worktree for anyone who wants the jail, an optional `MAX_TURNS` cap, and a README warning. Residual machine-wide risk is documented, not denied.
- **Single skill bench.** Upstream scans two install targets (`~/.claude/skills` for Claude, `~/.agents/skills` for Codex) because they are different runtimes. GLM runs *inside the Claude Code CLI*, so both agents read the same directory. The dual-bench scan collapses to one, and the "skill exists on only one bench" branch is deleted rather than faked.
- **Cross-model integrity is verified per run, not merely documented.** The main session must be Anthropic Claude, and the reviewer must actually be GLM — a silently-empty `ZAI_API_KEY` would fall through to the logged-in Anthropic credentials and produce Claude reviewing Claude, which looks identical in the JSON. Every round therefore asserts the reviewer identity (non-empty `ZAI_API_KEY` before launch, plus the model/endpoint echoed back by the headless session) before its verdict counts.
- **Read-only is enforced by construction, not by a sandbox flag.** In headless `-p` mode any tool absent from `--allowedTools` is auto-denied, so whitelisting only `Read Grep Glob` makes GLM read-only every round — and unlike the Codex original there is no separate resume-flag gotcha, because the same flags apply to round 1 and every resume.
- **Session id is re-read every round, never pinned.** `claude -p --resume` can fork and return a new id, so resuming the round-1 id forever would silently drop intermediate findings — the exact memory the loop exists for. Each round resumes the id returned by the previous round.
- **Per-run temp files, stderr preserved.** Fixed shared filenames let a stale file from a crashed run be parsed as this round's verdict; stderr sent to `/dev/null` throws away the auth error text needed to diagnose a failure. Both fixed in the contract below.
- **POSIX shell is a hard prerequisite.** The invocation syntax (env prefixes, `< /dev/null`, `/tmp`) is Git Bash / macOS / Linux only; it does not run in cmd or PowerShell. Stated in the README rather than papered over with a portability layer nobody asked for.
- **Upstream defaults preserved unchanged** (MAX_ROUNDS=5, MAX_FIX_ROUNDS=2, MAX_INSPECTION_ROUNDS=2, inspect=on, `PLAN.md`, `PLAN-REVIEW-LOG.md`) so behaviour differences trace to the model swap, not to retuning. New tunables added only where round 1 proved a gap: `TIMEOUT_MS` (default 600000), `isolate` (default `off`), `MAX_TURNS` (default unset — revisit with a generous default such as 50 once the gate shows typical build turn counts).

## GLM invocation contract (the core substitution)

Preflight, every run: `[ -n "$ZAI_API_KEY" ]` or hard stop. Echo `ANTHROPIC_MODEL`, the endpoint, and `claude --version` before round 1 so the user can confirm who is reviewing.

Read-only reviewer (Phase 2 / `il-glm-review`), round 1 fresh session:

```bash
STAMP=$(date +%s)-$$; OUT=/tmp/glm-verdict.$STAMP.json; ERR=/tmp/glm-verdict.$STAMP.err
rm -f "$OUT" "$ERR"; echo "OUT=$OUT ERR=$ERR"   # echo it: a later Bash call is a NEW shell, $$ differs
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p "$REVIEW_PROMPT" --output-format json \
  --allowedTools "Read Grep Glob" < /dev/null > "$OUT" 2>"$ERR"
```

Parse `.session_id` and `.result` (critique, ends with the VERDICT line). Rounds 2..MAX add `--resume "$SESSION_ID"` with the same flags, where `SESSION_ID` is **the id returned by the immediately preceding round**, re-read each time. `.is_error` true, or a missing/empty output file, is a hard stop — surface `$ERR` to the user, never retry blind.

**Path gotcha, verified in this session:** Git Bash `/tmp` is `C:\Users\<user>\AppData\Local\Temp`, which Windows-native tools (python, node) cannot open as `/tmp`. Any non-bash reader of these files must go through `cygpath -w` or use the Windows path directly.

Builder (Phase 3 / `il-glm-build`), after the clean-tree gate:

```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p --dangerously-skip-permissions --output-format json \
  < "$P" > "$BOUT" 2>"$BERR"   # BOUT/BERR stamped and echoed the same way
```

`--max-turns` appended when `MAX_TURNS` is set. With `isolate=worktree`, the command runs inside a throwaway `git worktree` on a scratch branch; after Claude reviews the diff, the work comes back as `git cherry-pick` of the scratch commit onto the working branch (single commit, no merge commit), and the worktree is removed. On rejection the worktree is deleted and nothing touches the working branch. Fix rounds resume the previous round's session id, bounded by `MAX_FIX_ROUNDS=2`, after which Claude takes over directly and logs the takeover. Every call runs under a `TIMEOUT_MS` ceiling (600000 default; raise it for large-repo reviews rather than treating a slow review as a failure).

## Terminology mapping
| Upstream | This repo |
|---|---|
| `claudex-loop` (plugin + skill) | `il-claudeGLM-loop` |
| `codex-review` | `il-glm-review` |
| `codex-build` | `il-glm-build` |
| `codex exec -s read-only` | `claude -p --allowedTools "Read Grep Glob"` |
| `codex exec resume ... -c sandbox_mode="read-only"` | `claude -p --resume "$SESSION_ID" --allowedTools "Read Grep Glob"` |
| `codex exec --yolo` | `claude -p --dangerously-skip-permissions` (machine-wide — see warning) |
| `thread_id` from JSONL `thread.started` | `.session_id` from `--output-format json`, re-read each round |
| `-o /tmp/codex-verdict.txt` | `> "$OUT"` (stamped, echoed once), read `.result` |
| `~/.codex/config.toml` model echo | echo `ANTHROPIC_MODEL` + endpoint + `claude --version` |
| `~/.agents/skills` (Codex bench) | n/a — single bench, `~/.claude/skills` |

## Acceptance gate (must pass before the retirement stays)
Plugin load-check (step 8) precedes the retirement. The rest runs in a throwaway git repo:
1. **Review loop, round 1** — valid JSON, non-empty `.session_id` and `.result`, `.is_error` false, `.result` ends in exactly one `VERDICT:` line.
2. **Reviewer identity, from objective signals** — non-empty `ZAI_API_KEY` at preflight, plus gate item 7 (a deliberately bad key must hard-stop, proving the call really goes to z.ai and not to logged-in Anthropic credentials), plus `.model` in the response JSON when present. The session self-reporting which model it is counts as corroboration only — models misidentify themselves, and a read-only reviewer has no Bash to echo its own env.
3. **Session memory, rounds 2 and 3** — each resume uses the previous round's id and demonstrably references its own earlier findings. Round 3 is included specifically to catch id churn across a resume chain.
4. **Read-only proven, not assumed** — a round whose plan explicitly invites a file write leaves the tree unchanged (`git status` clean).
5. **Build loop** — a tiny spec makes GLM actually create the specified file and run the proof command; Claude reads the resulting diff and reruns the proof itself and it passes.
6. **Verdict parsing** — the orchestrator branches correctly on both `VERDICT: APPROVED` and `VERDICT: REVISE`.
7. **Failure path** — an intentionally bad key produces a hard stop with the stderr text surfaced, not a silent pass or a blind retry.

## Assumptions
1. `ZAI_API_KEY` is set and on an active GLM Coding plan — verified present in this session's bash env; re-checked at every run per the contract.
2. `glm-5.3` is the current latest GLM model id — matches the user's existing skills, last bumped in `claude-skills` commit 49f3ee3 "chore(glm): point GLM review skills at glm-5.3".
3. No `glm` launcher on PATH — verified; skills spell the `ANTHROPIC_*` env prefix explicitly and mention the launcher only as an optional shortcut.
4. `~/.claude/skills` is the submodule `ilpin301/claude-skills` — verified via `.gitmodules`; the retirement commit lands in that repo and the parent `~/.claude` needs a submodule-pointer bump.
5. Upstream claudex-loop is MIT licensed and derivation with attribution is permitted — verified `LICENSE` present in the clone.
6. The main Claude Code session runs on Anthropic Claude, making GLM a genuinely different model — asserted, and the reviewer side is now verified per run (gate item 2).
7. **Corrected (round 1 evidence):** `/tmp` works for bash-side redirection only. It resolves to `C:\Users\<user>\AppData\Local\Temp` and is not openable by Windows-native tools under that name — cross-tool reads must use `cygpath -w`. The original assumption that "`/tmp` paths work on this machine" was wrong and broke the first verdict read in this very session.

## Risks / open questions
- **`--dangerously-skip-permissions` is machine-wide.** Accepted by explicit user decision; mitigated by the clean-tree gate, opt-in worktree isolation, optional turn cap, and a loud README warning. Residual risk: a prompt-injected spec or repo file could drive shell commands outside the repo. Anyone running `il-glm-build` on untrusted repo content should set `isolate=worktree`.
- **`--resume` context preservation across a cross-provider headless chain is the least-proven mechanic.** Gate items 1–3 are the first hard proof. If it fails, fallback is stateless rounds with the prior critique re-injected into the prompt — worse but functional.
- **`--dangerously-skip-permissions` behaviour under a non-Anthropic endpoint is unverified.** Gate item 5 is the proof; a failure means `il-glm-build` ships marked experimental rather than blocking the whole repo.
- **The `deep` research tier depends on a Workflow tool** that may not be available in every session. Kept, with a documented fallback to parallel Explore/WebSearch agents.
- **Stamped temp paths must be echoed, not recomputed.** Each orchestrator Bash call is a fresh shell, so `$$` differs between the call that writes the verdict and the call that reads it. The skills capture the path once and reuse the literal string.
- **Two repos now hold GLM tooling** (`il-claudeGLM-loop` for the loop, `claude-skills` for everything else). Drift risk if the invocation contract changes and only one is updated.

## Out of scope
- Reviewing or refactoring already-written code — that is a code-review tool's job, not this plugin.
- Changing any GLM invocation semantics beyond what the Codex→GLM swap and the round-1 hardening require; upstream defaults and phase structure stay as-is.
- Porting the legacy skills' content — `grill-me-glm` and `grill-with-docs-glm` move verbatim, they are not rewritten.
- A cmd/PowerShell portability layer — POSIX shell is a stated prerequisite.
- Publishing to GitHub during this work; the push is a separate, explicitly approved step after the gate.
- Retiring any `claude-skills` skill other than the three named.
- Multi-machine install, CI, or plugin-marketplace listing beyond the local `marketplace.json`.
