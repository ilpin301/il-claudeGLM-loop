---
name: grill-me-glm
description: Two-act plan hardening. ACT 1 (you ↔ Claude) — Claude interviews you relentlessly about a plan or design, one question at a time, recommending an answer for each and exploring the codebase when it can answer itself, until every branch of the decision tree is resolved. ACT 2 (Claude ↔ GLM) — Claude writes the locked plan to PLAN.md and GLM-5.3 adversarially reviews it read-only (VERDICT:APPROVED/REVISE), Claude revises and re-submits to the SAME GLM session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-me-glm", "grill me then have GLM review", "grill me and stress-test the plan", "interview me about this plan then get a second model on it", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants both alignment AND a cross-model sanity check before implementation. Builds on Matt Pocock's grill-me (MIT). For the docs-aware variant use /grill-with-docs-glm; if you already have a plan and want only the GLM review use /glm-review. NOT for reviewing already-written code (that's a code-review tool's job, not this skill) and NOT for trivial changes.
---

# Grill-Me-GLM — Get Grilled, Then Get Reviewed

Two acts, two different jobs:

- **Act 1 fixes the #1 failure mode: building the wrong thing.** Claude interrogates *you* until intent is locked — no guessing at ambiguity. (This act is Matt Pocock's `grill-me`, used under MIT — see `THIRD-PARTY-NOTICES.md`.)
- **Act 2 fixes the #2 failure mode: a plan that sounds right but breaks.** A *different model* (GLM-5.3) adversarially attacks the locked plan. Cross-model = no echo chamber.

You enter at two points only: answering the grill, and signing off the converged plan. GLM is read-only the whole time and never touches a file.

---

## ACT 1 — GRILL (you ↔ Claude)

> Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.
>
> Ask the questions one at a time, waiting for my answer before continuing.
>
> If a question can be answered by exploring the codebase, explore the codebase instead.

When the decision tree is resolved and we're aligned, **write the agreed plan to `PLAN.md`** in this structure, then move to Act 2:

```markdown
# Plan: <task>
_Locked via grill — by Claude + <user>_

## Goal
<one paragraph — reflects what the grilling actually settled>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices the grill resolved — name them so GLM has something to bite>

## Risks / open questions
<anything still genuinely open>

## Out of scope
<bounds the grill established>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Act 1 (grill) complete — plan locked with the user. MAX_ROUNDS=<n>.
```

---

## ACT 2 — REVIEW (Claude ↔ GLM)

Now hand the locked plan to GLM for adversarial review. GLM-5.3 runs through the Claude Code CLI pointed at z.ai's Anthropic-compatible endpoint — same repo-reading, read-only sandbox, and cross-round session memory the Codex port gave.

### Prerequisites (verify once, fast)
- Claude Code CLI installed (`claude --version`) — it is the harness that runs the GLM reviewer.
- A z.ai API key on a GLM Coding plan. Export it once as `ZAI_API_KEY` (never hardcode it in the skill). The reviewer talks to endpoint `https://api.z.ai/api/anthropic` with model `glm-5.3`.
- **Cross-provider note:** run your MAIN Claude Code session as normal Claude (Anthropic) so the reviewer (GLM-5.3) is a genuinely different model. If you launch the whole session under GLM, both sides are GLM and you lose the cross-model check.
- On auth/model/endpoint error (missing key, wrong model, bad endpoint), surface it to the user — do NOT silently retry.
- Optional shortcut: if a `glm` launcher is on PATH that already exports `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL`, you can replace the `ANTHROPIC_*= ... claude` invocation below with just `glm`.

### Tunables (read from args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan Act 1 produced. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. The artifact. |

If invoked with e.g. `rounds=3`, use that for `MAX_ROUNDS`. Echo resolved values before starting.

### The review prompt (sent each round)
> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `PLAN.md` and any repo files you need (you are read-only: you may Read/Grep/Glob but cannot edit or run anything). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Tag every finding `[HIGH]` (would break in production, cause data loss or a security hole, or block the core goal), `[MED]` (a real problem worth fixing but not a blocker), or `[LOW]` (nice-to-have / polish). Do NOT modify any files. Approval is severity-gated, not perfection-gated. End your reply with EXACTLY one line: `VERDICT: REVISE` if any `[HIGH]` finding remains, otherwise `VERDICT: APPROVED`. When you approve, still list any remaining `[MED]`/`[LOW]` items as non-blocking follow-ups — but do NOT withhold approval for them.

### Round 1 — fresh session
```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p "$REVIEW_PROMPT" \
  --output-format json \
  --allowedTools "Read Grep Glob" \
  < /dev/null > /tmp/glm-verdict.json 2>/dev/null
```
Then: Read `/tmp/glm-verdict.json`. It is a JSON object — pull `.session_id` (save as `SESSION_ID`) and `.result` (the critique text, which ends with the VERDICT line). If the file is missing/empty, or `.is_error` is true, the run failed (missing/invalid `ZAI_API_KEY`, wrong model, or bad endpoint) — STOP and tell the user; do not retry blind. `2>/dev/null` hides cosmetic proxy/telemetry stderr; `< /dev/null` gives immediate stdin EOF so nothing can block under a non-interactive driver.

### Rounds 2..MAX — resume the SAME session (GLM remembers its prior critiques)
```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
claude -p "I revised the plan. Re-review PLAN.md — check whether your prior findings are addressed and flag anything new. Same rules, still read-only. Apply the same severity gate: VERDICT: REVISE only if a [HIGH] issue remains, otherwise VERDICT: APPROVED with any [MED]/[LOW] items listed as non-blocking follow-ups." \
  --resume "$SESSION_ID" \
  --output-format json \
  --allowedTools "Read Grep Glob" \
  < /dev/null > /tmp/glm-verdict.json 2>/dev/null
```
`--resume "$SESSION_ID"` replays the prior conversation so GLM won't re-litigate settled points. Read `/tmp/glm-verdict.json` the same way (`.result`, and `.session_id` stays stable).

**Read-only guarantee:** In headless `-p` mode there is no interactive approver, so any tool NOT in `--allowedTools` is auto-denied. Whitelisting only `Read Grep Glob` means GLM can read the plan and the repo but cannot Edit, Write, or run Bash — it is read-only every round, by construction. Unlike the Codex port there is NO separate resume gotcha: the same flags apply to round 1 and every resume.

**Timeout guard (both rounds):** run every `claude -p` call with a 10-minute ceiling so a stall fails loud. Via Claude Code's Bash tool pass `timeout: 600000` on the tool call (the default 2-minute timeout is too short for a real review). In a plain shell prefix with `timeout 600` (Linux/Git Bash) or `gtimeout 600` (macOS). If it trips, treat as a failed run — stop and tell the user.

### Each round, after GLM returns
1. Append `## Round <n> — GLM` + the full `.result` critique to `LOG_FILE`.
2. Grep the last line for the verdict:
   - `VERDICT: APPROVED` → break to Resolution (converged). On APPROVED, copy any `[MED]`/`[LOW]` follow-ups GLM listed into `LOG_FILE` under `### Non-blocking follow-ups` so they're not lost.
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (Claude is final arbiter — GLM advises, doesn't command). Revise `PLAN_FILE`. Append `### Claude's response` to `LOG_FILE`: what changed, what was rejected, why. Increment round.
3. If round > `MAX_ROUNDS` → break to Resolution (deadlock).

### Resolution (you sign off — final gate)
- **APPROVED:** present the final `PLAN_FILE`, a 3-bullet summary of what the two acts improved, and the round count. Ask: *"Grilled + survived N rounds of GLM. Implement it now?"* Code only on yes. **No code is written during either act.**
- **MAX_ROUNDS hit without APPROVED (deadlock):** do NOT fake convergence. List each unresolved point + Claude's counter-position; hand it to the user to break the tie. A flagged disagreement beats a false "approved."

---

## Hard rules
- Act 1 always precedes Act 2 — don't write `PLAN.md` until the grill has actually resolved the decision tree with the user.
- GLM is read-only EVERY round — whitelist only Read/Grep/Glob so GLM is read-only every round. It never writes.
- The loop ALWAYS terminates at `MAX_ROUNDS`.
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a logged reason*. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after the user's final sign-off.
- `LOG_FILE` is the deliverable — keep the whole argument.

## What NOT to do
- Don't review already-written code — that's a code-review tool's job, not this skill.
- Don't hardcode the z.ai key in the skill — read it from `$ZAI_API_KEY`.
- Don't let GLM edit files. Read-only, always.
- Don't skip Act 1 — the grill is half the value.
