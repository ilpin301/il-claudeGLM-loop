---
name: il-glm-build
disable-model-invocation: true
description: DISABLED (user rule 2026-09-29: GLM never writes code; Claude implements). Hand a frozen spec (PLAN.md or any locked plan) to GLM-5.3 to IMPLEMENT with full write access, while Claude stays the spec-writer and reviewer — the exact role-flip of /il-glm-review. GLM builds from the spec in a headless permission-free session, Claude reads the full diff like a contributor PR, runs the proof test itself, and iterates fixes via the SAME GLM session up to MAX_FIX_ROUNDS before taking over. Clean git tree required before launch; human approves the diff before any commit. Use when the user says "/il-glm-build", "have GLM build this", "glm implement the plan", "hand the plan to GLM", "delegate the build to GLM", or right after a plan survives /il-claudeGLM-loop or /il-glm-review and they choose GLM for implementation (Phase 3). Also for standalone delegation: refactors, mechanical migrations, bug fixes with a known repro, test/coverage writing — anything that reads as a work order. NOT for tiny edits (~<20 lines — delegation overhead loses), NOT for design work (if writing the spec forces decisions, that's /il-claudeGLM-loop first), NOT for reviewing existing code, and NOT for anything needing Claude-session tools (MCP, secrets, browser).
---

**DISABLED BY USER RULE (2026-09-29): GLM never writes, edits or builds code. This skill must not be invoked; Claude implements all changes itself. GLM's only role is read-only critic of Claude's plans (il-glm-review / il-claudeGLM-loop). The body below is kept as historical reference only.**

# il-glm-build — GLM Types, Claude Verifies

_Rewrite of `codex-build` from [claudex-loop](https://github.com/chaseai-yt/claudex-loop) by Chase AI (MIT), with OpenAI Codex replaced by GLM-5.3. See `THIRD-PARTY-NOTICES.md`._

The role-flip of `/il-glm-review`: there, Claude builds and GLM critiques read-only. Here, **GLM is the builder with write access; Claude is the spec-writer and reviewer.** GLM implements a frozen spec end-to-end; Claude judges the diff like a contributor PR, demands proof, and iterates fixes in the same GLM session. The human enters at exactly two points: kickoff and diff sign-off.

**Spec quality decides success.** GLM starts with zero session context — everything it needs must be in the prompt. A plan that survived `/il-claudeGLM-loop` or `/il-glm-review` already is a frozen spec; that's the ideal input.

---

## ⚠️ Blast radius — read before first use

`--dangerously-skip-permissions` is **machine-wide, not repo-scoped.** In that mode GLM can:

- write files **outside** the repo,
- run arbitrary shell commands,
- install packages,
- make network calls.

Anything it reads — repo files, the spec itself, a dependency's README — is **untrusted input a permission-free agent may act on** (prompt injection). Never point this skill at a repo you don't trust.

The clean-tree gate does exactly one thing: it makes *in-repo* changes isolatable and revertible. **It does not bound the blast radius.** Mitigations available, in order of strength:

1. `isolate=worktree` — run the build in a throwaway git worktree so your working tree is untouchable (see below). Use this for any repo whose contents you did not write.
2. `MAX_TURNS` — cap the number of agent turns.
3. The clean-tree gate — always on, non-negotiable, but limited as described.

## Prerequisites (verify once, fast)

- Claude Code CLI installed (`claude --version`) — the harness that runs GLM.
- **POSIX shell required** (Git Bash / macOS / Linux). Not cmd, not PowerShell.
- z.ai API key exported as `ZAI_API_KEY`. Endpoint `https://api.z.ai/api/anthropic`, model `glm-5.3`.
- **Preflight, every run:** `[ -n "$ZAI_API_KEY" ]` or hard stop, then the direct ping in the Step 2 block (~3 s): `200` = key and endpoint fine, anything else = stop and surface the code. Cheap, fails fast. (The `ANTHROPIC_MODEL="glm-5.3"` pin works as is and needs no alias mapping — keep it pinned, dropping it risks a silent fallthrough to Claude.)
- **Echo at kickoff:** model, endpoint, `claude --version`, resolved tunables. If the user objects, stop before launching.
- **Objective identity proof (verified on the gate run):** the response JSON carries `modelUsage` keyed by the model that actually served the request — a real GLM round shows `{"glm-5.3": {...}}`. Assert that key rather than asking the session to describe itself; models misidentify themselves, and a read-only reviewer has no Bash to echo its own env.
- **The stderr line `[claude-code:unrecognized_model] {"model":"glm-5.3","query_source":"sdk"}` is a benign warning (verified).** It is printed on EVERY call, including fully successful ones (a successful ping returned "pong" with `modelUsage` `{"glm-5.3"}` and still printed it). It is NOT a bad-key signature — never diagnose from it, or from a hang. Bad key / wrong endpoint is diagnosed only by the direct ping (non-200).
- **Empty output after a timeout = "cap too short OR stalled", not "bad key".** Real GLM runs are slow (a plan re-review measured 760 s wall / 43 turns). Use the stream file (see Step 2) to tell "still working" from "stalled".
- **Hooks fire too:** Claude Code SessionStart hooks from user plugins also run inside the headless GLM session. Harmless — hook lines in the stream are not errors.
- Run from the target repo's root.

## Tunables (read from args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `SPEC_FILE` | `PLAN.md` | The frozen spec GLM implements. |
| `MAX_FIX_ROUNDS` | `2` | Fix iterations via resume before Claude takes over and finishes directly. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only build transcript. If it exists (earlier phases ran), append `## Phase 3 — Build`; else create it. |
| `PROOF_CMD` | from spec | Exact test/verify command GLM must run as proof. If the spec lacks one, ask the user ONE question to get it before launching. |
| `isolate` | `off` | `worktree` = build inside a throwaway git worktree on a scratch branch. |
| `MAX_TURNS` | unset | Appends `--max-turns <n>`. Revisit a default (~50) once real build turn counts are known. |
| `TIMEOUT_MS` | `1200000` | Ceiling per headless call (shell `timeout 1200`). |

Echo resolved values before starting.

## Step 0 — Gates (before any GLM launch)

1. **Spec gate.** `SPEC_FILE` must exist and read as a work order (goal, concrete steps, bounds). No spec → offer `/il-claudeGLM-loop` (interview first) or `/il-glm-review` (have a plan, want it stress-tested). If the user insists on building from a rough idea, write the spec WITH them first — that's design, and design stays with Claude.
2. **Clean-tree gate.** `git status -sb`. Dirty working tree → STOP and ask the user to commit or stash first. Non-negotiable: GLM writes with full access, and a dirty tree means its diff can't be isolated or cleanly reverted.
3. **Trust check.** If the repo contains content the user didn't author, recommend `isolate=worktree` before proceeding.
4. Confirm scope in one line, then go. No round-by-round approvals; the human gate is at the end.

## Step 1 — The build prompt (contract, via temp file)

Never inline-quote the prompt — write it to a temp file. Fill this contract completely; when chained from a loop skill, derive it from the plan's sections:

```bash
P=$(mktemp)
cat >"$P" <<'EOF'
GOAL: <one paragraph — what done looks like>
SPEC: Read <SPEC_FILE> at the repo root. It is a frozen, already-reviewed spec.
  Implement it exactly. If a step is impossible as written, implement the
  closest faithful version and report the deviation — do not redesign.
KEY PATHS: <files/dirs GLM will touch or must read first>
CONSTRAINTS: <"don't touch X", style rules, deps that must not change>
NON-GOALS: <explicitly out of scope — from the plan's Out of scope section>
PROOF: Run `<PROOF_CMD>` and include its full output in your report.
OUTPUT: End with a report — files changed (one line each: path + what/why),
  proof output, and any deviations from the spec with reasons.
EOF
```

## Step 2 — Launch GLM (fresh session)

```bash
[ -n "$ZAI_API_KEY" ] || { echo "ZAI_API_KEY unset — refusing to run"; exit 1; }
# Preflight ping (~3 s): 200 = key + endpoint fine. Anything else -> STOP, surface the code.
NO_PROXY='*' curl -s -m 30 -o /dev/null -w '%{http_code}\n' https://api.z.ai/api/anthropic/v1/messages -H "x-api-key: $ZAI_API_KEY" -H "anthropic-version: 2023-06-01" -H "content-type: application/json" -d '{"model":"glm-5.3","max_tokens":5,"messages":[{"role":"user","content":"ping"}]}'
git status -sb   # must be clean; abort otherwise

STAMP=$(date +%s)-$$
BOUT=/tmp/glm-build.$STAMP.jsonl
BERR=/tmp/glm-build.$STAMP.err
rm -f "$BOUT" "$BERR"
echo "BOUT=$BOUT BERR=$BERR"   # ECHO IT — the later read runs in a new shell where $$ differs

ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
timeout 1200 claude -p --dangerously-skip-permissions \
  --output-format stream-json --verbose \
  < "$P" > "$BOUT" 2>"$BERR"
```

- Prompt goes via stdin (`< "$P"`) — avoids quoting bugs AND gives immediate stdin EOF under a non-interactive driver.
- Append `--max-turns "$MAX_TURNS"` when set.
- Read `$BOUT` (the echoed literal path). It is a `.jsonl` stream; the final line with `"type":"result"` carries `result` (GLM's report), `session_id` (what fix rounds resume), `is_error`, `num_turns`, `duration_ms`, `modelUsage`. Extract with a short python json loop, e.g. `python -c "import json,sys; r=[x for x in map(json.loads,filter(str.strip,open(sys.argv[1]))) if x.get('type')=='result'][-1]; print(r['session_id'],r['is_error'],r['num_turns'],list(r['modelUsage'])); print(r['result'])" "$(cygpath -w "$BOUT")"`. No `result` line or `is_error` true → print `$BERR` and the tail of `$BOUT`, STOP. Exit 124 = cap hit, not proof of a bad key.
- **Mid-run progress (only if a stall is suspected, not as polling):** the stream is written live — count `"type":"tool_use"` blocks in the `type=="assistant"` lines; `system`/`thinking_tokens` lines show the session is alive.
- **Timing:** Claude Code's Bash tool caps foreground calls at 600000 ms, so launch every GLM call with `run_in_background: true` (the harness notifies on exit; do not poll) and read `$BOUT` when it exits. Don't kill a quiet background run early — real builds are legitimately slow.
- **Scope rule (prompt contract):** anything GLM reads is sent to z.ai. The build prompt's CONSTRAINTS must forbid reading `.env`/secret files and forbid Grep/Glob over large data/store directories.
- **Heads-up on completion (required):** when a background GLM run finishes, the FIRST line of your next message to the user must be a loud standalone banner — `🔔 GLM FINISHED — <what> (exit ok/fail) — verifying now` — BEFORE any verification output. The user is not watching tool calls; never let a completed build slide silently into the verify phase.

### `isolate=worktree`

```bash
git worktree add -b glm-build-$STAMP ../glm-build-$STAMP HEAD
# run Step 2 from ../glm-build-$STAMP, then verify there
```

After Claude reviews the diff and the proof passes: commit inside the worktree, `git cherry-pick` that commit onto the working branch (single commit, no merge commit), then `git worktree remove ../glm-build-$STAMP` and delete the scratch branch. On rejection: remove the worktree and delete the branch — the working branch is never touched.

## Step 3 — Verify (Claude, always, never delegated)

GLM's report is advisory. Verify yourself:

1. `git status -sb` + read the FULL diff (`git diff`). Judge it like a contributor PR: correctness, spec fidelity, style match with surrounding code, nothing touched outside scope.
2. **Check for out-of-scope writes.** Skip-permissions is machine-wide — confirm nothing landed outside the repo that the spec didn't call for. Expect interpreter artifacts the build produced as a side effect (`__pycache__/`, `.pytest_cache/`, build dirs) — those are the proof run's doing, not a spec deviation; gitignore or clean them rather than counting them against the diff.
3. Run `PROOF_CMD` yourself (or the focused tests for the changed area). GLM's pasted output doesn't count as proof.
4. Append to `LOG_FILE` under `## Phase 3 — Build`: `### Round <n> — GLM build` + its report summary + `### Claude's verdict` + what passed/failed review.

## Step 4 — Fix loop (same session, bounded)

Problems found → resume the SAME session (GLM keeps its context; cheaper and better than a fresh run). Write the fix list to a temp file (`$P2`), same contract discipline: exact problem, exact file, proof expected.

```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
timeout 1200 claude -p --resume "$SESSION_ID" --dangerously-skip-permissions \
  --output-format stream-json --verbose \
  < "$P2" > "$BOUT2" 2>"$BERR2"
```

(Run with `run_in_background: true`; `$BOUT2`/`$BERR2` are new stamped `.jsonl`/`.err` paths, echoed the same way.)

Re-read `session_id` from each round's final `"type":"result"` line and resume *that* id next time — resume can fork, and a stale id silently loses the fix context. Re-verify (Step 3) after each round. After `MAX_FIX_ROUNDS` failed rounds: STOP delegating — Claude takes over and finishes the remaining fixes directly. Log the takeover. Ping-ponging trivia through delegation burns more than it saves.

## Step 5 — Human gate (diff sign-off)

Present: 3-bullet summary of what was built, files-changed list, proof-test output (pass/fail, verbatim tail), rounds used, any spec deviations. Ask: *"GLM built it, proof passes, diff reviewed. Commit?"*

- Commit ONLY on yes — and Claude writes the commit, never GLM.
- Rejected → ask what's wrong, route back to Step 4 (or take over directly if fix rounds are spent).

## Hard rules

- Clean tree before launch. Always. No exceptions.
- Never launch with an empty `ZAI_API_KEY`, and never skip the preflight ping (200 or stop) — skip-permissions plus a silent provider fallback is the worst combination in this repo.
- Claude never skips the diff read. GLM's claims are advisory until Claude has read the diff and run the proof.
- Fix loop terminates at `MAX_FIX_ROUNDS` — then Claude takes over. No unbounded delegation ping-pong.
- Commits, pushes, releases, GitHub mutations: Claude-side only, after the human gate. GLM never commits.
- Capture stamped temp paths once and echo them; each orchestrator step is a new shell where `$$` differs.
- `LOG_FILE` is the deliverable — with the earlier phases it tells the whole story: reconned → interrogated → reviewed → built → verified.

## What NOT to do

- Don't build without a spec — that's designing by delegation, and it fails. Route to `/il-claudeGLM-loop` or `/il-glm-review` first.
- Don't use for ~<20-line single-obvious-change edits — just make the edit.
- Don't run this against untrusted repo content without `isolate=worktree`.
- Don't let GLM commit, and don't auto-commit yourself — human gate first.
- Don't `2>/dev/null` — you lose the only diagnostic when the run fails.
