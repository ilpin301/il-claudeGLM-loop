---
name: grill-with-docs-glm
description: Two-act plan hardening with living documentation. ACT 1 (you ↔ Claude) — Claude interviews you relentlessly about a plan, one question at a time, challenging it against your project's existing domain model and glossary (CONTEXT.md), sharpening fuzzy terms, stress-testing with concrete scenarios, cross-referencing code, and updating CONTEXT.md + ADRs inline as decisions crystallise. ACT 2 (Claude ↔ GLM) — Claude writes the locked plan to PLAN.md and GLM-5.3 adversarially reviews it read-only (VERDICT:APPROVED/REVISE), Claude revises and re-submits to the SAME GLM session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-with-docs-glm", "grill me against the docs then have GLM review", "stress-test this against our domain model then get a second model on it", or is about to build something high-stakes in a project with established terminology/ADRs and wants alignment, documentation, AND a cross-model sanity check. Builds on Matt Pocock's grill-with-docs (MIT). NOT for reviewing already-written code (that's a code-review tool's job, not this skill) and NOT for trivial changes.
---

# Grill-with-Docs-GLM — Grill Against Your Domain, Then Get Reviewed

Two acts. Act 1 aligns intent *and* keeps your living docs honest; Act 2 has a different model attack the result.

- **Act 1** is Matt Pocock's `grill-with-docs`, used under MIT (see `THIRD-PARTY-NOTICES.md`). It interrogates you, challenges your plan against `CONTEXT.md`/ADRs, and updates them inline.
- **Act 2** is the GLM-5.3 adversarial review loop — cross-model, read-only, bounded.

You enter at two points: answering the grill, and signing off the converged plan.

---

## ACT 1 — GRILL WITH DOCS (you ↔ Claude)

<what-to-do>

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing.

If a question can be answered by exploring the codebase, explore the codebase instead.

</what-to-do>

<supporting-info>

## Domain awareness

During codebase exploration, also look for existing documentation:

### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account' — do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible — which is right?"

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up — capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

`CONTEXT.md` should be totally devoid of implementation details. Do not treat `CONTEXT.md` as a spec, a scratch pad, or a repository for implementation decisions. It is a glossary and nothing else.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).

</supporting-info>

### Handoff to Act 2

When the decision tree is resolved, the glossary/ADRs are updated, and we're aligned, **write the agreed plan to `PLAN.md`** (use the canonical terms from `CONTEXT.md`), then run Act 2:

```markdown
# Plan: <task>
_Locked via grill-with-docs — by Claude + <user>. Terms per CONTEXT.md._

## Goal
<one paragraph, in the project's ubiquitous language>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices the grill resolved — link any ADRs created>

## Risks / open questions
<anything still open>

## Out of scope
<bounds>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Act 1 (grill-with-docs) complete — plan locked, CONTEXT.md/ADRs updated. MAX_ROUNDS=<n>.
```

---

## ACT 2 — REVIEW (Claude ↔ GLM)

Hand the locked plan to GLM for adversarial review. GLM-5.3 runs through the Claude Code CLI pointed at z.ai's Anthropic-compatible endpoint — same repo-reading, read-only sandbox, and cross-round session memory the Codex port gave.

### Prerequisites
- Claude Code CLI installed (`claude --version`) — it is the harness that runs the GLM reviewer.
- A z.ai API key on a GLM Coding plan. Export it once as `ZAI_API_KEY` (never hardcode it in the skill). The reviewer talks to endpoint `https://api.z.ai/api/anthropic` with model `glm-5.3`.
- **Cross-provider note:** run your MAIN Claude Code session as normal Claude (Anthropic) so the reviewer (GLM-5.3) is a genuinely different model. If you launch the whole session under GLM, both sides are GLM and you lose the cross-model check.
- On auth/model/endpoint error (missing key, wrong model, bad endpoint), surface it to the user — do NOT silently retry.
- Optional shortcut: if a `glm` launcher is on PATH that already exports `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL`, you can replace the `ANTHROPIC_*= ... claude` invocation below with just `glm`.

### Tunables (args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap. Loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan from Act 1. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. |

Invoked with e.g. `rounds=3` → use it. Echo resolved values first.

### Review prompt (each round)
> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `PLAN.md` (and `CONTEXT.md`/ADRs for the domain language) and any repo files you need (you are read-only: you may Read/Grep/Glob but cannot edit or run anything). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, domain-language mismatches, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Tag every finding `[HIGH]` (would break in production, cause data loss or a security hole, or block the core goal), `[MED]` (a real problem worth fixing but not a blocker), or `[LOW]` (nice-to-have / polish). Do NOT modify any files. Approval is severity-gated, not perfection-gated. End your reply with EXACTLY one line: `VERDICT: REVISE` if any `[HIGH]` finding remains, otherwise `VERDICT: APPROVED`. When you approve, still list any remaining `[MED]`/`[LOW]` items as non-blocking follow-ups — but do NOT withhold approval for them.

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

### Rounds 2..MAX — resume SAME session (GLM remembers its prior critiques)
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

### Each round
1. Read the `.result` from the JSON; append `## Round <n> — GLM` + critique to `LOG_FILE`.
2. Last line verdict: `APPROVED` → Resolution (converged) — on APPROVED, copy any `[MED]`/`[LOW]` follow-ups GLM listed into `LOG_FILE` under `### Non-blocking follow-ups` so they're not lost; `REVISE` → Claude decides what's worth acting on (final arbiter), revise `PLAN_FILE`, append `### Claude's response` (what changed/rejected + why), increment.
3. round > `MAX_ROUNDS` → Resolution (deadlock).

### Resolution (you sign off)
- **APPROVED:** present final plan + 3-bullet summary of what the two acts improved + round count. Ask: *"Grilled + survived N rounds of GLM. Implement it now?"* No code during either act.
- **Deadlock (cap hit, no APPROVED):** list unresolved points + Claude's counter-position; hand to user. Don't fake convergence.

---

## Hard rules
- Act 1 precedes Act 2. `CONTEXT.md` stays a glossary only — no implementation details.
- GLM read-only EVERY round — whitelist only Read/Grep/Glob so GLM is read-only every round. Never writes.
- Loop ALWAYS terminates at `MAX_ROUNDS`. Claude is final arbiter on REVISE (reject with logged reason). Code only after sign-off. `LOG_FILE` is the deliverable.

## What NOT to do
- Don't review already-written code (that's a code-review tool's job, not this skill). Don't hardcode the z.ai key — read it from `$ZAI_API_KEY`. Don't let GLM edit files. Don't skip Act 1.
