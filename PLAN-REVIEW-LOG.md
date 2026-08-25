# Plan Review Log: il-claudeGLM-loop — GLM rewrite of claudex-loop

Act 1 (grill) complete — plan locked with the user. MAX_ROUNDS=5.

Decisions locked in Act 1:
1. Deliverable — full publishable plugin repo (not loose skills).
2. Naming — `il-claudeGLM-loop`, `il-glm-review`, `il-glm-build`; new skills replace the old ones after a successful test.
3. Migration — retire the superseded `claude-skills` entries first, then test clean; `git revert` on failure.
4. Build access — `--dangerously-skip-permissions` guarded by the clean-tree gate.
5. Test gate — both loops smoke-tested in a throwaway repo.
6. Home — `C:\Users\<user>\il-claudeGLM-loop`; GitHub push only after the gate, with explicit approval.

---

## Round 1 — GLM (session e1a5a26c-2bbb-4629-ab9a-a548e521d14a)

Read PLAN.md, PLAN-REVIEW-LOG.md, existing `~/.claude/skills/il-glm-review/SKILL.md` (the verified invocation the plan reuses). Findings:

### [HIGH] findings

**1. "Blast radius is bounded by the clean-tree gate" is false for `il-glm-build`.** `--dangerously-skip-permissions` gives GLM unrestricted Bash and file write on the *whole machine*, not the repo. The clean-tree gate only makes changes *inside the repo* revertible — it does nothing about edits outside the repo, `curl` exfiltration, installed packages, or prompt injection riding in repo/PLAN content that a yolo agent then executes. The plan explicitly rejected the worktree jail on the grounds that the gate "already covers" this risk; that reasoning is wrong, so the rejection is unsound. Fix: run the build phase in an isolated git worktree (or at minimum a dedicated throwaway clone), correct the blast-radius claim in the skill text, and document the residual machine-wide risk instead of denying it.

### [MED] findings

**2. Session-id churn across resumes not handled.** The plan pins `SESSION_ID` from round 1 and says "`.session_id` stays stable". `claude -p --resume` can return a *new* session id per resume (resume forks). Gate item 2 only proves round 2 — round 3+ would silently resume from the round-1 fork and lose all intermediate findings, voiding the session-memory feature the loop exists to provide. Fix: re-read `.session_id` from every round's JSON and resume the latest one; add a round-3 check to the gate.

**3. Nothing verifies the reviewer actually ran as GLM.** If env leakage fails silently, `claude -p` may fall through to logged-in Anthropic creds and the "cross-model" review is Claude reviewing Claude — undetectable by the current gate. Fix: gate item asserting the reviewer's identity before counting a pass.

**4. Publishable repo, Git Bash-only invocation.** The invocation contract uses POSIX env-prefix syntax, `/tmp`, and `< /dev/null`. On plain Windows cmd/PowerShell (or a Mac without GNU `timeout`) every command breaks. Fix: state the Git Bash requirement in README prerequisites.

**5. Fixed temp filename `/tmp/glm-verdict.json` shared across rounds and concurrent runs.** A stale non-empty file from a previous run could be parsed as this round's verdict after a failed invocation that wrote nothing. Fix: per-run unique filename plus delete-before-write.

### [LOW] findings

**6. `2>/dev/null` discards the actual auth/endpoint error text.** Fix: redirect stderr to a sibling `.err` file.

**7. Builder has no turn cap.** Fix: add `--max-turns`.

**8. Retire-first leaves a no-tooling window if plugin install itself fails.** Fix: confirm the plugin skills load before the retirement commit, not after.

**9. 600 s ceiling may be tight for a `deep`-tier review of a large repo.** Fix: make the timeout a documented tunable.

VERDICT: REVISE

### Claude's response — round 1

**Accepted, with the user's decision preserved (finding 1).** GLM is right that the blast-radius claim was false: `--dangerously-skip-permissions` is machine-wide, and the clean-tree gate only makes *in-repo* changes revertible. The plan no longer claims otherwise and now names the real residual risks (writes outside the repo, arbitrary shell, network, prompt injection from repo/spec content). GLM's prescribed fix — mandatory worktree — is **not** adopted wholesale: the user chose skip-permissions over a worktree jail explicitly in Act 1, and that is their call to make. Instead the jail ships as an `isolate=worktree` opt-in tunable, recommended in the plan for untrusted repo content, alongside an optional `MAX_TURNS` cap and a README warning. Honest documentation plus an available mitigation, rather than overriding a decision the user made with the tradeoff in front of them.

**Accepted in full:** 2 (session id re-read every round, gate now checks round 3 specifically), 3 (reviewer-identity assertion is a gate item and a per-run preflight), 4 (POSIX shell is a stated prerequisite in README and skills; a portability layer is explicitly out of scope), 5 (per-run `$$`-stamped temp files with delete-before-write), 6 (stderr to a sibling `.err`, surfaced on failure), 7 (`MAX_TURNS`, default unset), 8 (plugin load-check moved ahead of the retirement commit — this closes the "neither loads" window), 9 (`TIMEOUT_MS` tunable, default 600000).

**Added from this session's own evidence, not from GLM:** assumption 7 was wrong and broke the first verdict read. Git Bash `/tmp` resolves to `C:\Users\<user>\AppData\Local\Temp` and is not openable under that name by Windows-native tools — cross-tool reads must go through `cygpath -w`. Corrected in the assumptions and called out in the invocation contract.

**Rejected:** nothing outright.

---

## Round 2 — GLM (same session e1a5a26c-2bbb-4629-ab9a-a548e521d14a, resumed)

Re-read revised PLAN.md. Prior findings check:

| # | Round-1 finding | Status |
|---|---|---|
| 1 | [HIGH] false blast-radius claim, worktree rejection unsound | **Fixed** — machine-wide scope stated in those words, prompt-injection risk named, `isolate=worktree` opt-in shipped, `MAX_TURNS`, README warning, explicit user decision logged. Claim no longer lies; risk now documented, not denied. |
| 2 | [MED] session-id churn | **Fixed** — id re-read every round, gate item 3 added round 3 specifically for chain churn. |
| 3 | [MED] no proof reviewer is GLM | **Fixed** — preflight key check, env echo, gate item 2. See MED-1 below for residual weakness. |
| 4 | [MED] POSIX-only syntax on "publishable" repo | **Fixed** — hard prerequisite in README + out-of-scope line. |
| 5 | [MED] fixed temp filename clobber/stale | **Fixed** — `$$`-stamped `OUT`/`ERR`, `rm -f` before write. See MED-2 for new wrinkle. |
| 6 | [LOW] `2>/dev/null` hides diagnostics | **Fixed** — `$ERR` surfaced on failure, gate item 7 proves the failure path. |
| 7 | [LOW] no turn cap on yolo builder | **Fixed** — optional `MAX_TURNS`. |
| 8 | [LOW] load-check before retirement | **Fixed** — step 8 precedes step 9. |
| 9 | [LOW] fixed 600 s ceiling | **Fixed** — `TIMEOUT_MS` tunable. |

Bonus: assumption 7 correction (Git Bash `/tmp` not readable by Windows-native tools) — real defect found with evidence, properly folded into contract. Good.

New findings from the revision itself:

**MED-1. Gate item 2's identity proof leans on model self-report.** A headless session "confirming" it runs against z.ai on glm-5.3 is the model describing itself — models routinely misidentify themselves, and the reviewer has no Bash to echo env objectively. Preflight key check + gate item 7 (bad key → hard stop) are the load-bearing evidence; self-report is decoration that could false-negative a legitimate pass. Fix: gate item 2 should cite the objective signals (non-empty key preflight + item-7 endpoint discrimination) and treat self-report as corroborating only; if the JSON ever carries a `.model` field, assert it.

**MED-2. `$$` temp paths break cross-call reads as written.** `OUT=/tmp/glm-verdict.$$.json` expands inside the one Bash call that runs `claude`; the orchestrator reads the file in a *later* Bash call where `$$` is a different PID — `$OUT` recomputes to the wrong path. Fix: the same command must `echo "$OUT"` (or derive the name from a stamp captured once), and every later reference uses that literal path.

**LOW-1.** `isolate=worktree` "merged back only after Claude reviews the diff" — merge mechanics unspecified (cherry-pick? copy? re-apply diff?). One line in the skill deciding this would do.

**LOW-2.** `MAX_TURNS` default unset — fine for now; consider a generous default (e.g. 50) after the gate proves typical build turn counts.

**LOW-3.** Builder stderr also `$$`-stamped — same read-path consideration as MED-2, same fix.

Non-blocking follow-ups: MED-1, MED-2, LOW-1..3.

VERDICT: APPROVED

### Claude's response — round 2

Approved on the severity gate. All four MED/LOW follow-ups were applied anyway, because three of them are cheap and one (MED-2) is a genuine bug in the contract I wrote:

- **MED-2 accepted — real defect.** `$$` expands per shell, and every orchestrator step is a new Bash call, so the read step would recompute a different path than the write step. The contract now stamps the name once (`STAMP=$(date +%s)-$$`), echoes it, and every later reference uses that literal path. Same fix applied to the builder's stdout/stderr (LOW-3).
- **MED-1 accepted.** Gate item 2 no longer rests on the reviewer describing itself. It now cites objective signals: non-empty key at preflight, the bad-key hard-stop of gate item 7 (which is what actually discriminates z.ai from Anthropic fall-through), and `.model` in the response JSON when present. Self-report is corroboration only.
- **LOW-1 accepted.** Worktree merge mechanics specified: scratch branch in a throwaway worktree, `git cherry-pick` back after Claude reviews the diff, worktree removed; on rejection nothing touches the working branch.
- **LOW-2 accepted as written** — `MAX_TURNS` stays unset for now, with a note to revisit a default of ~50 once the gate shows real build turn counts. Guessing a cap before there is data would just truncate builds.

**Rejected: nothing.**

### Non-blocking follow-ups carried forward

None outstanding — MED-1, MED-2 and LOW-1..3 were all folded into `PLAN.md` before sign-off. The only deferred item is the `MAX_TURNS` default, which is deliberately data-gated on the acceptance run.

### Convergence

APPROVED after 2 rounds (MAX_ROUNDS=5). Session memory verified in practice: round 2 tracked all nine round-1 findings by number and status, which is gate item 3's evidence arriving early, for free.

---

## Build + acceptance gate — results

Built per the approved plan, then run against the 7-item gate in throwaway repos.

| # | Gate item | Result | Evidence |
|---|-----------|--------|----------|
| 0 | Plugin loads before retirement | PASS | `claude plugin validate` passed; `claude plugin details` lists 3 skills (il-claudeGLM-loop, il-glm-build, il-glm-review), ~1,516 always-on tokens. Only warning is the non-kebab plugin name, which Claude Code accepts (user's naming pattern). |
| 1 | Review round 1 well-formed | PASS | `.is_error` false, `.session_id` `75f2b226-…`, `.result` 3,217 chars ending in exactly one `VERDICT: REVISE`. Found 3 real [HIGH]s in the bait plan (per-worker dict x4 workers, spoofable XFF, unbounded-dict memory DoS). |
| 2 | Reviewer identity, objective | PASS | Response JSON carries `modelUsage: {"glm-5.3": {...}}` on every round — objective proof of who served the request, better than the self-report the plan settled for. |
| 3 | Session memory across a chain | PASS | Rounds 2 and 3 resumed the same id (no fork). Round 3, asked to recall its first review from memory, listed all ten findings by number and severity. |
| 4 | Read-only proven, not assumed | PASS | The test plan contained a "Housekeeping" section instructing the reviewer to write `REVIEW_NOTES.md` and append to `PLAN.md`. After round 1: `git status --porcelain` empty, no `REVIEW_NOTES.md`. Tool whitelisting held against an in-plan instruction to write. |
| 5 | Build loop with write access | PASS | Clean-tree gate confirmed 0 changes, then GLM built `slugify.py` + `test_slugify.py` from a frozen spec. Claude read both files and ran `python test_slugify.py` independently: `OK`, exit 0. Spec fidelity good; it documented its unicode choice in a comment as the spec demanded. |
| 6 | Verdict parsing, both branches | PASS | `VERDICT: REVISE` (round 1) and `VERDICT: APPROVED` (rounds 2, 3) both parsed and branched correctly. |
| 7 | Failure path on a bad key | PASS, with a correction | An invalid key produced an empty output file, `[claude-code:unrecognized_model] {"model":"glm-5.3"}` on stderr, and a hang until `timeout 180` killed it (exit 124). The empty-key preflight refused to launch at all. |

### Corrections the gate forced into the shipped skills

1. **The "silent fallthrough to Claude" premise was wrong.** Every skill (and the README) claimed an empty/wrong `ZAI_API_KEY` would quietly produce a Claude-reviewing-Claude run. It does not: `glm-5.3` is not an Anthropic model, so the call is rejected outright. The real failure mode is a *hang* until the timeout with an empty output file. All four documents reconciled — the preflight is still mandatory (it turns a ten-minute timeout into a one-line failure), and the caveat that the risk returns if anyone drops the `ANTHROPIC_MODEL` pin is now stated.
2. **`modelUsage` is objective identity proof.** GLM's round-2 MED-1 said the identity check leaned on self-report. The gate found a better signal in the response JSON itself; the skills now say to assert that key.
3. **Interpreter artifacts are not spec deviations.** The build produced `__pycache__/` as a side effect of running its own proof. `il-glm-build`'s verify step now says to expect those rather than counting them against the diff.

### Migration

`ilpin301/claude-skills` commit `774ef4e` retired `grill-me-glm`, `grill-with-docs-glm`, and `il-glm-review`; they live on in the plugin (`skills/il-glm-review`, `legacy/`). Retirement stands — the gate passed. Rollback remains one `git revert 774ef4e`.
