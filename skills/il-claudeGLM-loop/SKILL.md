---
name: il-claudeGLM-loop
description: Four-phase plan hardening with GLM-5.3 as the rival model. PHASE 0 RECON — Claude scouts first (codebase + docs on brownfield; prior art, stack, and pitfalls research on greenfield) and drafts an assumptions ledger. PHASE 1 INTERROGATE — confirm the ledger in one batch, then question only the load-bearing decisions one at a time (each with why-it-matters, a recommendation, and what-breaks-if-we-guess-wrong), cosmetic ones batched, with a visible decision map and an accept-all-recommendations escape hatch. PHASE 2 REVIEW — the locked plan goes to PLAN.md and GLM-5.3 adversarially reviews it read-only (VERDICT: APPROVED/REVISE, severity-gated); Claude revises and re-submits to the SAME GLM session until APPROVED or MAX_ROUNDS, then you sign off before any code. PHASE 3 BUILD (optional) — you pick the builder and the models swap jobs: GLM builds via il-glm-build and Claude reads the full diff + runs the proof itself; Claude builds and a fresh read-only GLM session cross-inspects the diff (on by default, logged opt-out only); either way you approve the final diff. Use when the user says "/il-claudeGLM-loop", "run the GLM loop", "glm-loop this", "grill me then have GLM review", "stress-test this plan before we build", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments, greenfield architecture) and wants alignment AND a cross-model sanity check first. Locked plan needing only the GLM loop → /il-glm-review. NOT for reviewing already-written code, NOT for trivial changes.
---

# il-claudeGLM-loop — Recon, Interrogate, Review, Build

_Rewrite of [claudex-loop](https://github.com/chaseai-yt/claudex-loop) by Chase AI (MIT) with OpenAI Codex replaced by GLM-5.3. See `THIRD-PARTY-NOTICES.md`._

Four phases, four failure modes killed:

- **Phase 0 — RECON** kills *interviewing blind*: Claude scouts the terrain (code or research) before asking you anything, so the interview starts informed instead of generic.
- **Phase 1 — INTERROGATE** kills *building the wrong thing*: Claude interrogates you until intent is locked — but only on decisions that are actually load-bearing.
- **Phase 2 — REVIEW** kills *a plan that sounds right but breaks*: a different model (GLM-5.3) attacks the locked plan. Cross-model = no echo chamber.
- **Phase 3 — BUILD** *(optional)* kills *grading your own work*: one model implements the locked plan, the rival model grades the diff — in both directions.

You enter at four points only: confirming the assumptions ledger, answering the fire, signing off the converged plan, and approving the final diff if you build. GLM is read-only throughout recon, interrogation, and review — **no code is written until you sign off the converged plan.**

---

## PHASE 0 — RECON (Claude alone)

Before asking the user a single question, determine the terrain and gather what can be gathered without them.

### Detect the terrain
- **Brownfield** — the working directory has real source code (not just scaffolding/config). Recon the codebase.
- **Greenfield** — empty dir, fresh scaffold, or the user is describing a brand-new project with no repo yet. There is nothing to recon; research replaces it.

### Brownfield recon
1. Explore the codebase: architecture, relevant modules, existing patterns the plan must fit, current schema/auth/infra as applicable.
2. Look for living docs: `CONTEXT.md` (or `CONTEXT-MAP.md` for multi-context repos) and `docs/adr/`. If they exist, load them — the project has a ubiquitous language and prior decisions the plan must respect, and Phase 1 runs **docs-aware** (see below).
3. If the task involves tech or an integration the repo can't answer, open the **research gate** (below) before proceeding.

### Greenfield recon
No code to read, so research carries the phase. Open the **research gate**, then cover:
1. **Prior art** — how do existing tools/products solve this? What's the standard shape?
2. **Stack** — reasonable default stack for this kind of project, with one alternative worth considering.
3. **Known pitfalls** — the 3-5 things people building this class of thing get wrong (search for postmortems, "lessons learned", common gotchas of the candidate stack).

### The research gate (one question, asked at kickoff when external research would help)
Don't silently pick a research depth — offer the tiers with a recommendation based on stakes, and let the user choose:

- **`none`** — Claude's knowledge + codebase only. Right for medium tasks on familiar ground.
- **`web`** — a handful of targeted WebSearch passes (docs, gotchas, prior art). Minutes, not a project. The default recommendation for most greenfield work.
- **`deep`** — a multi-agent research orchestration: parallel finder agents each searching a different way (prior art, stack landscape, pitfalls/postmortems, docs), then deep-read agents on the best sources, then one synthesis agent producing the brief. Heavy and token-expensive — recommend only for high-stakes greenfield, unfamiliar tech, or when the landscape itself is the question. The user choosing this tier IS the explicit opt-in.
  - **Preferred runner:** a deep-research dynamic workflow via the Workflow tool, if the session has it. **Model pin:** every `agent()` call MUST pass `model: 'opus'` — letting a dozen research agents inherit a heavy session model annihilates token usage for what is mostly search-and-summarize work. Leave effort at the default. **Args gotcha:** the workflow runtime may deliver `args` as a JSON-encoded STRING — always open the script with `const A = typeof args === 'string' ? JSON.parse(args) : args`.
  - **Fallback when the Workflow tool is unavailable** (common — do not treat this as a blocker): run the same shape with parallel `Explore`/general-purpose agents plus WebSearch, one agent per research angle, then synthesize yourself. Say which runner you used.

If invoked with `research=none|web|deep`, skip the question and use that tier.

**If `deep` is chosen: draft the research prompt and get sign-off before launching.** Show the user the topic framing + the 3-5 specific questions the assumptions ledger needs answered (not a generic "research X" — questions shaped like "what do teams building X get wrong about auth?" / "what's the current standard stack for Y and why?"). The user edits or approves, THEN launch. Save the synthesized brief to `docs/research/YYYY-MM-DD-<slug>-glm-loop-research.md` (or your notes location, with `## Key Takeaways`) — link it from the ledger entries it sourced and from `PLAN.md`.

### Skill inventory scan (both terrains, after terrain detection)
Enumerate installed skills and match against the task's domain: list `~/.claude/skills/` plus any plugin skill dirs (folder names + frontmatter `description` first lines are enough — don't read full SKILL.mds during recon).

**One bench, not two.** The Codex-based original scanned two install targets (`~/.claude/skills` for Claude, `~/.agents/skills` for Codex) because those are different runtimes. GLM-5.3 here runs *inside the Claude Code CLI*, so both agents read the same skill directory. There is no "installed on only one bench" case to report, and no separate Codex-side loading behaviour to smoke-test.

Record hits in the Assumptions Ledger as proposed toolchain entries, never auto-loads:

> "threejs-game-skills pack installed (9 skills incl. aaa-graphics-builder, gameplay-systems, 3d/image/audio generators) — proposing the build phase load graphics-builder + gameplay-systems, and the asset track use the generator skills. — source: skill inventory scan"

**Discovery informs the plan; nothing loads unless `PLAN.md`'s `## Toolchain` section names it and survives review.**

### Output: the Assumptions Ledger
End Phase 0 by presenting a single batch — NOT one-at-a-time — of everything Claude resolved on its own:

```markdown
## Assumptions Ledger
_Confirm or correct in one pass. Anything unmarked I treat as confirmed._
1. <assumption> — source: <code path / doc / research finding / convention>
2. ...
```

Each entry cites its source. The user confirms/corrects in one reply. Corrections that open real questions get promoted into the Phase 1 decision map. This is the single biggest time-save over a naive grill: the interview never wastes questions on things the repo or the research already answered.

---

## PHASE 1 — INTERROGATE (you ↔ Claude)

The interview. Built around one principle: **every question must justify its own existence.**

### Open with the Decision Map

```markdown
## Decision Map
### Load-bearing (asked one at a time)
- [ ] <decision> — irreversible / expensive-if-wrong (schema, auth, data model, concurrency, money, public API)
### Cosmetic (batched with defaults)
- [ ] <decision> — cheap to change later
```

Load-bearing = wrong answer costs a migration, a rewrite, a security hole, or user trust. Cosmetic = renameable, refactorable, swappable. Update the map as questions resolve (check items off, add branches corrections open) so the user can see convergence instead of wondering how many questions are left.

### Load-bearing questions — one at a time, structured

> **Q<n>: <the question>**
> **Why it matters:** <the dependency or constraint that makes this load-bearing>
> **Recommendation:** <Claude's answer, committed — not a menu>
> **If we guess wrong:** <the concrete failure — migration, rewrite, breach, churn>

Wait for the answer before the next question. If drafting a question and the "if we guess wrong" line comes out weak — the question is cosmetic; demote it to the batch. If mid-interrogation a question turns out answerable from the code or the research, answer it yourself and log it to the ledger instead of asking.

### Cosmetic decisions — one batch
Present the whole cosmetic tier as recommendations with a one-line rationale each. The user vetoes by exception; silence = accepted.

### Escape hatch
At any point the user can say **"accept all remaining recommendations"** — Claude locks every open decision at its recommended answer, logs them as such in the plan, and proceeds. Offer it explicitly if the load-bearing tier exceeds ~8 questions.

### Docs-aware mode (auto-on when Phase 0 found CONTEXT.md/ADRs; offer once on greenfield)
- **Enforce the glossary** — when the user's wording collides with a `CONTEXT.md` definition, stop and resolve it on the spot: quote the glossary's meaning, state the apparent meaning, make them pick.
- **Pin down loose words** — an overloaded or vague term gets a proposed canonical replacement before the conversation continues on top of it.
- **Probe boundaries with scenarios** — when two concepts blur, construct a concrete edge case that forces the line between them to be drawn.
- **Check claims against the code** — when the user asserts how something behaves, verify in the source; a mismatch is surfaced as a question, not silently trusted either way.
- **Maintain `CONTEXT.md` as terms settle** (format: [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md)). Glossary ONLY — never implementation details. Created lazily on the first settled term.
- **Offer ADRs only past the three-part test** — expensive to reverse AND puzzling without context AND a genuine trade-off. Format: [ADR-FORMAT.md](./ADR-FORMAT.md). `docs/adr/` created lazily.

### Lock the plan
When the decision map is fully checked and you're aligned, **write `PLAN.md`**:

```markdown
# Plan: <task>
_Locked via il-claudeGLM-loop — by Claude + <user>_

## Goal
<one paragraph — reflects what the interrogation actually settled>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices the interrogation resolved — name them so GLM has something to bite; link any ADRs; mark any locked via the escape hatch>

## Toolchain
<only when the skill inventory scan matched something — which installed skills the build MUST load and follow, plus any generator skills or MCP capabilities the build depends on. Omit entirely on no matches. Reviewable like everything else: GLM should attack unused relevant skills and unjustified inclusions alike>

## Assumptions
<the confirmed ledger — with sources>

## Risks / open questions
<anything still genuinely open>

## Out of scope
<bounds the interrogation established>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Phases 0-1 (recon + interrogation) complete — plan locked with the user. MAX_ROUNDS=<n>.
```

---

## PHASE 2 — REVIEW (Claude ↔ GLM)

Hand the locked plan to GLM-5.3 for adversarial review. GLM runs through the Claude Code CLI pointed at z.ai's Anthropic-compatible endpoint — same repo-reading, same read-only guarantee, same cross-round session memory.

### Prerequisites (verify once, fast)
- Claude Code CLI installed (`claude --version`) — it is the harness that runs the GLM reviewer.
- **POSIX shell required.** The invocations below use env-var prefixes, `/tmp`, and `< /dev/null`. Git Bash (Windows), macOS, or Linux. They do not run in cmd or PowerShell.
- A z.ai API key on a GLM Coding plan, exported as `ZAI_API_KEY` (never hardcoded). Endpoint `https://api.z.ai/api/anthropic`, model `glm-5.3`.
- **Preflight, every run:** `[ -n "$ZAI_API_KEY" ]` or hard stop, then the direct ping in the Round 1 block (~3 s): `200` = key and endpoint fine, anything else = stop and surface the code. Cheap, fails fast. (The `ANTHROPIC_MODEL="glm-5.3"` pin works as is and needs no alias mapping — keep it pinned, dropping it risks a silent fallthrough to Claude.)
- **Cross-provider note:** run your MAIN Claude Code session as normal Claude (Anthropic) so the reviewer is a genuinely different model. Launch the whole session under GLM and both sides are GLM — the cross-model check is gone.
- **Echo before Round 1** so the user can confirm who is reviewing: model (`glm-5.3`), endpoint, and `claude --version`.
- On auth/model/endpoint error, surface `$ERR` to the user — do NOT silently retry.
- **Objective identity proof (verified on the gate run):** the response JSON carries `modelUsage` keyed by the model that actually served the request — a real GLM round shows `{"glm-5.3": {...}}`. Assert that key rather than asking the session to describe itself; models misidentify themselves, and a read-only reviewer has no Bash to echo its own env.
- **The stderr line `[claude-code:unrecognized_model] {"model":"glm-5.3","query_source":"sdk"}` is a benign warning (verified).** It is printed on EVERY call, including fully successful ones (a successful ping returned "pong" with `modelUsage` `{"glm-5.3"}` and still printed it). It is NOT a bad-key signature — never diagnose from it, or from a hang. Bad key / wrong endpoint is diagnosed only by the direct ping (non-200).
- **Empty output after a timeout = "cap too short OR stalled", not "bad key".** A real plan review is slow: a re-review of a mid-size plan plus one 480-line script took 760 s wall / 43 turns / 42 read-only tool calls, and the old 600 s cap killed that same review with exit 124. Use the stream file (see Round 1) to tell "still working" from "stalled".
- **Hooks fire too:** Claude Code SessionStart hooks from user plugins also run inside the headless GLM session. Harmless — hook lines in the stream are not errors.
- Optional shortcut: if a `glm` launcher is on PATH that already exports `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL`, replace the `ANTHROPIC_*= ... claude` prefix with just `glm`.

### Tunables (read from args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan Phase 1 produced. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. The artifact. |
| `TIMEOUT_MS` | `1200000` | Ceiling per headless call (shell `timeout 1200`). Real reviews run 10+ min; raise further for large-repo or deep reviews rather than treating a slow review as a failure. |
| `research` | ask | `none` / `web` / `deep` — pre-answers the Phase 0 research gate. |
| `inspect` | `on` | Post-build cross-inspection of Claude-built code by a fresh read-only GLM session. `off` = skip (logged as an explicit opt-out, never silently). |
| `MAX_INSPECTION_ROUNDS` | `2` | Initial post-build review + one reinspection after accepted fixes. |
| `isolate` | `off` | `worktree` runs Phase 3 GLM builds in a throwaway git worktree. See `il-glm-build`. |
| `MAX_TURNS` | unset | Optional `--max-turns` cap on the Phase 3 builder. |

If invoked with e.g. `rounds=3`, use that for `MAX_ROUNDS`. Echo resolved values before starting.

### The review prompt (sent each round)
> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable. Read the plan at `PLAN.md` (and `CONTEXT.md`/ADRs for domain language, if present) and any repo files you need (you are read-only: you may Read/Grep/Glob but cannot edit or run anything). Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Tag every finding `[HIGH]` (would break in production, cause data loss or a security hole, or block the core goal), `[MED]` (a real problem worth fixing but not a blocker), or `[LOW]` (nice-to-have / polish). Do NOT modify any files. Approval is severity-gated, not perfection-gated. End your reply with EXACTLY one line: `VERDICT: REVISE` if any `[HIGH]` finding remains, otherwise `VERDICT: APPROVED`. When you approve, still list any remaining `[MED]`/`[LOW]` items as non-blocking follow-ups — but do NOT withhold approval for them. Scope: read only the files this plan needs (name them for GLM in the prompt); do NOT Grep/Glob over large data or store directories; do NOT read `.env` or any secret/credential file — everything you read is sent to z.ai.

(On greenfield there are no repo files — GLM reviews `PLAN.md` and its `## Assumptions` section on their own merits; the assumption sources give it something concrete to attack.)

### Round 1 — fresh session

Claude Code's Bash tool caps foreground calls at 600000 ms, so run every round with `run_in_background: true` — the harness notifies on exit; do not poll.

```bash
[ -n "$ZAI_API_KEY" ] || { echo "ZAI_API_KEY unset — refusing to run"; exit 1; }
# Preflight ping (~3 s): 200 = key + endpoint fine. Anything else -> STOP, surface the code.
NO_PROXY='*' curl -s -m 30 -o /dev/null -w '%{http_code}\n' https://api.z.ai/api/anthropic/v1/messages -H "x-api-key: $ZAI_API_KEY" -H "anthropic-version: 2023-06-01" -H "content-type: application/json" -d '{"model":"glm-5.3","max_tokens":5,"messages":[{"role":"user","content":"ping"}]}'
STAMP=$(date +%s)-$$
OUT=/tmp/glm-verdict.$STAMP.jsonl
ERR=/tmp/glm-verdict.$STAMP.err
rm -f "$OUT" "$ERR"
echo "OUT=$OUT ERR=$ERR"   # ECHO IT — a later Bash call is a new shell and $$ differs

ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
timeout 1200 claude -p "$REVIEW_PROMPT" \
  --output-format stream-json --verbose \
  --allowedTools "Read Grep Glob" \
  < /dev/null > "$OUT" 2>"$ERR"
```

Then read `$OUT` — the literal path echoed above, **not** a recomputed `$STAMP`. It is a `.jsonl` stream; the final line with `"type":"result"` carries `session_id` (→ `SESSION_ID`), `result` (the critique, ending with the VERDICT line), `is_error`, `num_turns`, `duration_ms`, `modelUsage`. Extract with a short python json loop, e.g. `python -c "import json,sys; r=[x for x in map(json.loads,filter(str.strip,open(sys.argv[1]))) if x.get('type')=='result'][-1]; print(r['session_id'],r['is_error'],r['num_turns'],list(r['modelUsage'])); print(r['result'])" "$(cygpath -w "$OUT")"`. If there is no `result` line or `is_error` is true, the run failed — print `$ERR` and the tail of `$OUT` and STOP; do not retry blind. Exit 124 = cap hit, not proof of a bad key.

**Mid-run progress (only if a stall is suspected, not as polling):** the stream is written live — count `"type":"tool_use"` blocks in the `type=="assistant"` lines; `system`/`thinking_tokens` lines show the session is alive.

`2>"$ERR"` (not `/dev/null`) keeps the error text; it always contains the benign `unrecognized_model` warning — ignore that line. `< /dev/null` gives immediate stdin EOF so nothing blocks under a non-interactive driver.

### Rounds 2..MAX — resume the SAME session (GLM remembers its prior critiques)

```bash
ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic" \
ANTHROPIC_AUTH_TOKEN="$ZAI_API_KEY" \
ANTHROPIC_MODEL="glm-5.3" \
NO_PROXY="*" \
timeout 1200 claude -p "I revised the plan. Re-review PLAN.md — check whether your prior findings are addressed and flag anything new. Same rules, still read-only, same scope limits (only files you need, no Grep/Glob over large data/store dirs, never .env/secret files). Apply the same severity gate: VERDICT: REVISE only if a [HIGH] issue remains, otherwise VERDICT: APPROVED with any [MED]/[LOW] items listed as non-blocking follow-ups." \
  --resume "$SESSION_ID" \
  --output-format stream-json --verbose \
  --allowedTools "Read Grep Glob" \
  < /dev/null > "$OUT2" 2>"$ERR2"
```

(`$OUT2`/`$ERR2` are new stamped `.jsonl`/`.err` paths, echoed the same way; run with `run_in_background: true`.)

**Re-read `session_id` from every round's final `"type":"result"` line and resume that id next round.** Do not pin round 1's id for the whole loop: `--resume` can fork and hand back a new id, and resuming a stale one silently drops every intermediate finding — exactly the memory the loop exists for.

**Read-only guarantee:** in headless `-p` mode there is no interactive approver, so any tool NOT in `--allowedTools` is auto-denied. Whitelisting only `Read Grep Glob` means GLM can read the plan and the repo but cannot Edit, Write, or run Bash — read-only every round, by construction. Unlike the Codex original there is NO separate resume gotcha: the same flags apply to round 1 and every resume.

**Timeout guard (both rounds):** run every `claude -p` call under shell `timeout 1200` (`TIMEOUT_MS`; `gtimeout` on macOS) so a stall fails loud. The Bash tool's foreground cap is 600000 ms, so launch with `run_in_background: true` and wait for the exit notification. If the cap trips, treat it as a failed run — stop and tell the user.

**Windows path gotcha:** Git Bash `/tmp` resolves to `C:\Users\<user>\AppData\Local\Temp`. Windows-native tools (python, node) cannot open it under the `/tmp` name — convert with `cygpath -w` or use the Windows path when reading these files outside bash.

### Each round, after GLM returns
1. Append `## Round <n> — GLM` + the full `.result` critique to `LOG_FILE`.
2. Grep the last line for the verdict:
   - `VERDICT: APPROVED` → break to Resolution (converged). Copy any `[MED]`/`[LOW]` follow-ups into `LOG_FILE` under `### Non-blocking follow-ups` so they're not lost.
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (Claude is final arbiter — GLM advises, doesn't command). Revise `PLAN_FILE`. Append `### Claude's response` to `LOG_FILE`: what changed, what was rejected, why. Increment round.
3. If round > `MAX_ROUNDS` → break to Resolution (deadlock).

### Resolution (you sign off — final gate)
- **APPROVED:** present the final `PLAN_FILE`, a 3-bullet summary of what the loop improved, and the round count. Ask: *"Interrogated + survived N rounds of GLM. Implement it now — GLM builds it (`/il-glm-build`), Claude builds it, or stop here?"* Code only on a yes. **No code is written during Phases 0-2.**
- **MAX_ROUNDS hit without APPROVED (deadlock):** do NOT fake convergence. List each unresolved point + Claude's counter-position; hand it to the user to break the tie. A flagged disagreement beats a false "approved."

---

## PHASE 3 (optional) — BUILD (GLM ↔ Claude, roles flipped)

If the user picks GLM: invoke the `il-glm-build` skill with `SPEC_FILE=PLAN.md` and the same `LOG_FILE` — it appends `## Phase 3 — Build` to the log, so one artifact tells the whole story (reconned → interrogated → reviewed → built → verified). Roles flip: GLM writes the code with full access, Claude reviews the diff and runs the proof. If the user picks Claude, implement directly — then run the **post-build cross-inspection** below.

### Post-build cross-inspection (default on every Claude-built path)

The doctrine is *whoever made the thing never checks the thing* — that applies to Claude's code too. After Claude implements and the proof gates pass:

1. Launch a **fresh read-only GLM session** (new session, NOT the Phase 2 one — the reviewer should see the code cold, not through its own plan critiques). Same read-only invocation as Round 1, with a new `$STAMP`. Give it: `PLAN.md`, the base commit, and the code diff. Ask for PR-style findings — correctness, spec fidelity, edge cases, nothing outside scope — no verdict line needed; this is advisory review, not a gate loop.
2. Claude arbitrates each finding: accept (fix it, rerun affected tests) or reject *with a logged reason*. Cap at `MAX_INSPECTION_ROUNDS=2`.
3. Append to `LOG_FILE` under `## Post-build inspection`: findings verbatim, Claude's dispositions, rounds used. Present the summary alongside the final diff at the human gate.

Opt-out: `inspect=off` at invocation or the user declining at Resolution. Skipping silently is not allowed — the log must show either the inspection or the explicit opt-out.

---

## Hard rules
- Phases run in order: 0 → 1 → 2. Don't write `PLAN.md` until the interrogation has actually resolved the decision map with the user (or they invoked the escape hatch).
- The assumptions ledger is presented ONCE as a batch — never drip assumptions as individual questions.
- GLM is read-only EVERY review round — whitelist only `Read Grep Glob`, round 1 and every resume. It never writes during Phases 0-2.
- Never run a round with an empty `ZAI_API_KEY`, and never skip the preflight ping — fail fast on anything but a 200.
- Scope the reviewer in the prompt: name the files it needs, forbid Grep/Glob over large data/store directories, forbid reading `.env`/secret files — anything GLM reads is sent to z.ai.
- Re-read `session_id` each round; never pin round 1's id for the whole loop.
- Capture stamped temp paths once and echo them; every orchestrator step is a new shell where `$$` differs.
- The loop ALWAYS terminates at `MAX_ROUNDS`.
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a logged reason*. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after the user's final sign-off.
- `LOG_FILE` is the deliverable — keep the whole argument.
- `CONTEXT.md` stays a glossary only — never implementation details.

## What NOT to do
- Don't invoke this skill just to review pre-existing code. (Code built BY this skill does get reviewed — that's the post-build cross-inspection, on by default.)
- Don't run the main session under GLM — then both sides are GLM and the whole premise collapses.
- Don't let GLM edit files during review. Read-only, always.
- Don't skip Phase 1 — the interrogation is half the value.
- Don't ask questions the recon already answered, and don't ask a load-bearing-format question whose "if we guess wrong" is weak — demote it to the cosmetic batch.
- Don't turn Phase 0 into a research project on a medium-stakes task — the research gate exists so the user picks the depth.
- Don't `2>/dev/null` the headless calls — you lose the only diagnostic you get when auth fails.
