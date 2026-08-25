---
name: il-glm-review
description: A standalone adversarial PLAN-review loop where Claude Code (builder) and GLM-5.3 (read-only critic) tag-team an implementation plan before any code is written. Use this when you ALREADY have a plan or a clear idea and just want the cross-model stress-test — no requirements interview first. Claude drafts/loads the plan into PLAN.md, GLM reviews it read-only (headless `claude -p` pointed at z.ai) and returns VERDICT:APPROVED or VERDICT:REVISE under a severity gate, Claude revises and re-submits to the SAME GLM session (context preserved) until APPROVED or a configurable MAX_ROUNDS cap is hit. Human approves the converged plan before code. Use when the user says "/il-glm-review", "glm review my plan", "have GLM review my plan", "argue this plan with GLM", "adversarial plan review", "make Claude and GLM argue over the plan", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants a second-model sanity check on the PLAN before implementation. For a guided requirements interview BEFORE the review, use /il-claudeGLM-loop instead. NOT for reviewing already-written CODE and NOT for trivial changes.
---

# il-glm-review — Adversarial Plan-Review Loop

_Rewrite of `codex-review` from [claudex-loop](https://github.com/chaseai-yt/claudex-loop) by Chase AI (MIT), with OpenAI Codex replaced by GLM-5.3. See `THIRD-PARTY-NOTICES.md`._

Two models, one plan, a bounded argument. **Claude is the builder and orchestrator. GLM-5.3 is a read-only critic** that can read the repo and the plan but cannot touch a single file. They communicate strictly through `PLAN.md` + a GLM session that persists across rounds. The human enters at exactly two points: kickoff and final sign-off.

This is a **deliberate, high-stakes tool** — reach for it on auth, data models, concurrency, migrations, payments, anything expensive to get wrong. Skip it for obvious/cheap work.

## Prerequisites (verify once, fast)

- Claude Code CLI installed (`claude --version`) — it is the harness that runs the GLM reviewer.
- **POSIX shell required.** Env-var prefixes, `/tmp`, `< /dev/null` — Git Bash (Windows), macOS, or Linux. Not cmd, not PowerShell.
- z.ai API key on a GLM Coding plan, exported as `ZAI_API_KEY` (never hardcoded). Endpoint `https://api.z.ai/api/anthropic`, model `glm-5.3`.
- **Preflight, every run:** `[ -n "$ZAI_API_KEY" ]` or hard stop. Measured behaviour with a missing or invalid key: the run does **not** quietly become a Claude review — `glm-5.3` is not an Anthropic model, so the call is rejected — but it fails by hanging until the timeout with an empty output file, burning ten minutes for nothing. Preflight turns that into a one-line failure. (The silent-fallthrough risk returns the moment someone drops the `ANTHROPIC_MODEL` pin — keep it pinned.)
- **Cross-provider note:** run your MAIN session as normal Claude (Anthropic). If the whole session runs under GLM, both sides are GLM and the cross-model check is gone.
- **Echo before Round 1:** model (`glm-5.3`), endpoint, `claude --version`. If the user objects, stop before burning a round.
- On auth/model/endpoint error, surface `$ERR` — do NOT silently retry.
- **Objective identity proof (verified on the gate run):** the response JSON carries `modelUsage` keyed by the model that actually served the request — a real GLM round shows `{"glm-5.3": {...}}`. Assert that key rather than asking the session to describe itself; models misidentify themselves, and a read-only reviewer has no Bash to echo its own env.
- **A bad key hangs, it does not error fast (verified).** With an invalid `ZAI_API_KEY` the call produces an empty output file, writes `[claude-code:unrecognized_model] {"model":"glm-5.3"}` to stderr, and then sits until the timeout kills it (exit 124). Two consequences: the empty/missing output file is your real failure detector, and the `glm-5.3` model id is itself a safety net — Anthropic does not recognise it, so a key mix-up cannot silently downgrade into Claude reviewing Claude. Never treat a timeout as "slow review" without checking `$ERR` first.
- Optional shortcut: a `glm` launcher on PATH that already exports `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL` can replace the `ANTHROPIC_*= ... claude` prefix.

## Tunable variables (read from skill args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | Where the evolving plan lives (repo root). |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only transcript of the argument. The artifact. |
| `TIMEOUT_MS` | `600000` | Ceiling per headless call. Raise for large repos rather than calling a slow review a failure. |

If invoked with e.g. `rounds=3`, use that for `MAX_ROUNDS`. Echo the resolved values before starting.

## Flow

### Step 0 — Kickoff (human gate #1)

The invocation is the kickoff. Confirm scope in one line: what is being planned. If the user gave no task, ask for it (one question). Then proceed — no round-by-round approvals; the human gate is at the end.

### Step 1 — Claude plans

Do real planning: read the relevant code, think through the approach, surface decisions and tradeoffs. Then write `PLAN_FILE`:

```markdown
# Plan: <task>
_Round 0 — initial draft by Claude_

## Goal
<one paragraph>

## Approach
<numbered steps, concrete>

## Key decisions & tradeoffs
<the contestable choices — name them explicitly so GLM has something to bite>

## Risks / open questions
<what you're unsure about>

## Out of scope
<bounds>
```

Initialize `LOG_FILE`:
```markdown
# Plan Review Log: <task>
Started <stamp the user's local time if known, else "session start">. MAX_ROUNDS=<n>.
```

Show the user the plan inline and say you're sending it to GLM for adversarial review.

### Step 2 — The loop

Maintain `ROUND` (start 1) and `SESSION_ID` (empty until round 1 returns).

**The review prompt** sent each round:

> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `PLAN.md` and any repo files you need (you are read-only: you may Read/Grep/Glob but cannot edit or run anything). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Tag every finding `[HIGH]` (would break in production, cause data loss or a security hole, or block the core goal), `[MED]` (a real problem worth fixing but not a blocker), or `[LOW]` (nice-to-have / polish). Do NOT modify any files. Approval is severity-gated, not perfection-gated. End your reply with EXACTLY one line: `VERDICT: REVISE` if any `[HIGH]` finding remains, otherwise `VERDICT: APPROVED`. When you approve, still list any remaining `[MED]`/`[LOW]` items as non-blocking follow-ups — but do NOT withhold approval for them.

**Round 1** (creates the session):

```bash
[ -n "$ZAI_API_KEY" ] || { echo "ZAI_API_KEY unset — refusing to run (would silently review as Claude)"; exit 1; }
STAMP=$(date +%s)-$$
OUT=/tmp/glm-verdict.$STAMP.json
ERR=/tmp/glm-verdict.$STAMP.err
rm -f "$OUT" "$ERR"
echo "OUT=$OUT ERR=$ERR"   # ECHO IT — a later Bash call is a new shell and $$ differs

ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p "$REVIEW_PROMPT" \
  --output-format json \
  --allowedTools "Read Grep Glob" \
  < /dev/null > "$OUT" 2>"$ERR"
```

Read `$OUT` — the literal path echoed above, never a recomputed `$STAMP`. Pull `.session_id` (→ `SESSION_ID`) and `.result` (the critique, ending with the VERDICT line). Missing/empty file, or `.is_error` true → print `$ERR`, STOP, tell the user. No blind retry.

> `2>"$ERR"` instead of `/dev/null` keeps the auth/endpoint error text — it is the only diagnostic you get. `< /dev/null` gives immediate stdin EOF so nothing blocks under a non-interactive driver.
>
> **Timeout guard:** run every call under `TIMEOUT_MS`. Via Claude Code's Bash tool pass `timeout: 600000` (the default 2-minute tool timeout kills real reviews). In a plain shell prefix `timeout 600` (Linux/Git Bash) or `gtimeout 600` (macOS). If it trips, treat it as a failed run — stop, don't retry blind.
>
> **Windows path gotcha:** Git Bash `/tmp` is `C:\Users\<user>\AppData\Local\Temp`. Windows-native tools (python, node) can't open it as `/tmp` — use `cygpath -w` or the Windows path when reading these files outside bash.

**Rounds 2..MAX** (resume the SAME session — GLM remembers its earlier critiques, won't re-litigate settled points):

```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p "I revised the plan. Re-review PLAN.md — check whether your prior findings are addressed and flag anything new. Same rules, still read-only. Apply the same severity gate: VERDICT: REVISE only if a [HIGH] issue remains, otherwise VERDICT: APPROVED with any [MED]/[LOW] items listed as non-blocking follow-ups." \
  --resume "$SESSION_ID" \
  --output-format json \
  --allowedTools "Read Grep Glob" \
  < /dev/null > "$OUT2" 2>"$ERR2"
```

**Re-read `.session_id` from every round and resume that id next round.** `--resume` can fork and return a new id; pinning round 1's id would silently drop all intermediate findings — the exact session memory this loop exists for.

**Read-only guarantee:** in headless `-p` mode any tool absent from `--allowedTools` is auto-denied — there is no interactive approver to say yes. Whitelisting only `Read Grep Glob` makes GLM read-only by construction, round 1 and every resume alike. (The Codex original needed a separate `-c sandbox_mode` on resume; this port does not.)

**Each round, after GLM returns:**
1. Append to `LOG_FILE`: `## Round <n> — GLM` + the full critique.
2. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → break, go to Step 3 (converged). Copy any `[MED]`/`[LOW]` items into `LOG_FILE` under `### Non-blocking follow-ups`.
   - `VERDICT: REVISE` → Claude reads the critique, decides **what's actually worth acting on** (Claude has final say — GLM advises, it does not command). Revise `PLAN_FILE`. Append `### Claude's response` + what changed, what was rejected, why. Increment `ROUND`.
3. If `ROUND > MAX_ROUNDS` → break to Step 3 (deadlock).

### Step 3 — Resolution (human gate #2)

**If APPROVED:** present the final `PLAN_FILE`, a 3-bullet summary of what the argument improved, and the round count. Ask: *"Plan survived N rounds of GLM. Implement it now — GLM builds it (`/il-glm-build`), Claude builds it, or stop here?"* Only on a yes is code written. **No code is written during the loop.** If the user picks GLM, invoke `il-glm-build` with `SPEC_FILE=PLAN.md` and the same `LOG_FILE` — roles flip and the build rounds append to the same log.

**If MAX_ROUNDS hit without APPROVED (deadlock):** do NOT pretend it converged. List each point GLM still flags and Claude's counter-position. Hand it to the human to break the tie. A flagged disagreement beats a false "approved."

## Hard rules

- GLM is read-only EVERY round — `--allowedTools "Read Grep Glob"`, round 1 and every resume. It never writes.
- Never run with an empty `ZAI_API_KEY` — an unset key does not silently review as Claude (the pinned model id blocks that), it hangs until the timeout — fail fast instead.
- Re-read `.session_id` each round; never pin round 1's id for the whole loop.
- Capture stamped temp paths once and echo them; each orchestrator step is a new shell where `$$` differs.
- The loop ALWAYS terminates at `MAX_ROUNDS`. No unbounded recursion.
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a logged reason*.
- Code only after human gate #2.
- `LOG_FILE` is the deliverable — keep the whole argument.

## What NOT to do

- Don't use this to review existing code — this is a plan-review loop.
- Don't run the main session under GLM — both sides GLM defeats the point.
- Don't skip the log — the argument transcript is the most valuable artifact.
- Don't let GLM edit files. Read-only, always.
- Don't `2>/dev/null` — you throw away the only failure diagnostic.
