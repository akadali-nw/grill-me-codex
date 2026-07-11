---
name: codex-review
description: A standalone adversarial PLAN-review loop where Claude Code (builder) and OpenAI Codex (read-only critic) tag-team an implementation plan before any code is written. Use this when you ALREADY have a plan or a clear idea and just want the cross-model stress-test — no requirements interview first. Claude drafts/loads the plan into PLAN.md, Codex reviews it in a read-only sandbox and returns VERDICT:APPROVED or VERDICT:REVISE, Claude revises and re-submits to the SAME Codex session (context preserved) until APPROVED or a configurable MAX_ROUNDS cap is hit. Human approves the converged plan before code. Use when the user says "/codex-review", "codex review my plan", "have Codex review my plan", "argue this plan with Codex", "adversarial plan review", "make Claude and Codex argue/fight over the plan", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants a second-model sanity check on the PLAN before implementation. For a guided requirements interview BEFORE the review, use /grill-me-codex instead. NOT for reviewing already-written CODE (that is the Codex plugin's /codex:review) and NOT for trivial changes.
---

# Codex-Review — Adversarial Plan-Review Loop

Two models, one plan, a bounded argument. **Claude is the builder and orchestrator. Codex is a read-only critic** that can read the repo and the plan but cannot touch a single file. They communicate strictly through `PLAN.md` + a Codex session that persists across rounds. The human enters at exactly two points: kickoff and final sign-off.

This is a **deliberate, high-stakes tool** — reach for it on auth, data models, concurrency, migrations, payments, anything expensive to get wrong. Skip it for obvious/cheap work.

## Prerequisites (verify once, fast)

- Codex CLI installed and recent: `codex --version` (need ≥ 0.130; the default `gpt-5.5` model errors on older CLIs).
- Codex authenticated: a prior `codex login` (ChatGPT account is fine). If a run returns an auth/model error, surface it to the user — do not silently retry.
- Do NOT pin `-m` unless the user asks. The user's `~/.codex/config.toml` default model is used. Pinning `gpt-5.x-codex` variants fails on ChatGPT-account auth.
- **Echo the active model before Round 1** so the user can confirm: read the `model` line from `~/.codex/config.toml` (absent = "CLI default"); state it with the resolved tunables. If the user objects, stop before burning a round.
- **Sandbox flag differs between the two commands.** `codex exec` accepts `-s read-only`. `codex exec resume` does NOT — it rejects `-s` ("unexpected argument"). On resume you MUST force read-only via `-c sandbox_mode="read-only"`, because `config.toml` may default `sandbox_mode` to `danger-full-access` (+ `approval_policy="never"`) — which would let Codex WRITE files mid-loop. This is the single most important safety detail in this skill: verified end-to-end on 2026-06-04.

## Tunable variables (read from skill args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates at this. |
| `PLAN_FILE` | `PLAN.md` | Where the evolving plan lives (repo root). |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only transcript of the argument (every round's critique + what changed). The artifact. |

If the user invoked the skill with an argument like `rounds=3`, use that for `MAX_ROUNDS`. Echo the resolved values back before starting.

## Flow

### Step 0 — Kickoff (human gate #1)

The invocation itself is the kickoff. Confirm scope in one line: what is being planned. If the user gave no task, ask for it (one question). Then proceed — do NOT ask for approval round-by-round; that comes at the end.

### Step 1 — Claude plans

Do real planning: read the relevant code, think through the approach, surface decisions and tradeoffs. Then write the plan to `PLAN_FILE` in this structure:

```markdown
# Plan: <task>
_Round 0 — initial draft by Claude_

## Goal
<one paragraph>

## Approach
<numbered steps, concrete>

## Key decisions & tradeoffs
<the contestable choices — name them explicitly so Codex has something to bite>

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

Show the user the plan inline and say you're sending it to Codex for adversarial review.

### Step 2 — The loop

Maintain `ROUND` (start 1) and `THREAD_ID` (empty until round 1 returns).

**The review prompt** sent to Codex each round (adjust the task line):

> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to redesign it. If an `intent.md` exists, read it first as the authoritative goal — everything must serve it. Read the plan at `PLAN.md` (and any repo files you need; you are read-only). Decisions marked LOCKED are settled by the product owner: flag one ONLY if genuinely broken (correctness, security, feasibility), never propose an alternative design for it — you are a reviewer, not the architect. Otherwise identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the plan is sound enough to implement, or `VERDICT: REVISE` if it still has material problems.

**Round 1** (creates the session — capture `thread_id`):

```bash
codex exec -s read-only --json \
  -o /tmp/codex-verdict.txt \
  "$(cat REVIEW_PROMPT)" \
  < /dev/null 2>/dev/null | grep '"type":"thread.started"'
```
Parse `thread_id` from the `{"type":"thread.started","thread_id":"..."}` line → that is `THREAD_ID`. The critique text lands in `/tmp/codex-verdict.txt` (Codex's last message). Read that file.

> Note: stderr carries cosmetic MCP/auth noise on some setups — `2>/dev/null` is intentional. Confirm success by the presence of the verdict file + a `thread.started` line. If neither appears, the run failed (auth/model) — stop and tell the user.
>
> **`< /dev/null` is mandatory:** `codex exec` reads stdin *in addition to* the prompt arg, so under a non-interactive driver (Claude Code's Bash tool, CI, any non-TTY pipeline) it blocks forever waiting on stdin EOF — a silent ~0% CPU hang. The redirect gives it immediate EOF. Required on the resume call below too.
>
> **Timeout guard:** run every `codex exec` / `codex exec resume` with a 10-minute ceiling so any future stall fails loud instead of hanging silently. Via Claude Code's Bash tool, pass `timeout: 600000` on the tool call (the default 2-minute tool timeout is too short for real reviews and would kill them mid-run). In a plain shell, prefix the command with `timeout 600` (Linux / Git Bash) or `gtimeout 600` (macOS via coreutils — stock macOS has no `timeout`). If the ceiling trips, treat it as a failed run: stop and tell the user rather than retrying blind. Applies to the resume call below too.

**Rounds 2..MAX** (resume the SAME session — Codex remembers its earlier critiques, won't re-litigate settled points):

```bash
# NOTE: resume rejects -s. Force read-only via -c sandbox_mode, or Codex
# inherits config.toml (possibly danger-full-access) and could write files.
codex exec resume "$THREAD_ID" -c sandbox_mode="read-only" --json \
  -o /tmp/codex-verdict.txt \
  "I revised the plan. Re-review PLAN.md. Same rules. End with VERDICT: APPROVED or VERDICT: REVISE." \
  < /dev/null 2>/dev/null >/dev/null
```

Both `codex exec` and `codex exec resume` support `--json` (stream → parse `thread_id` first round) and `-o/--output-last-message` (verdict capture).

**Each round, after Codex returns:**
1. Read `/tmp/codex-verdict.txt`. Append to `LOG_FILE`: `## Round <n> — Codex` + the full critique.
2. **Claude is the architect; Codex advises.** Gauge every finding against the goal (and `intent.md` if it exists) — a finding that redesigns a LOCKED decision that isn't actually broken, or pulls the plan off-goal, is rejected and logged "off-intent." Don't take Codex at face value and change every decision.
3. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → break the loop, go to Step 3 (converged).
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (final say — reject off-intent + low-ROI *with a logged reason*, not just factual wrongness). Revise `PLAN_FILE`. Append to `LOG_FILE`: `### Claude's response` + what changed, what was rejected, why. Increment `ROUND`.
4. If `ROUND > MAX_ROUNDS` → break to Step 3 (deadlock).

### Act 2 discipline — direction over round-count

`MAX_ROUNDS` (5) is a ceiling, not a target. The reviewer's only way to add value is more `REVISE` findings — it never says "delete half of this" — so the plan grows unless you fight it.

**Trajectory check — classify each round in `LOG_FILE`, one word: CONVERGING or SPIRALING.**
- Converging: Codex *concedes/closes* prior findings; new ones are tightening (spec gaps, edge cases), not architectural; count trending down.
- Spiraling (any one): the same finding recircles reframed; the round only *added guards*, no decision changed; the plan grew again with nothing resolved.

**On a spiraling round, stop folding and step out:** ask the altitude question — *"what's the irreversible-harm profile — do the elaborate and simple versions differ on it? If not, the machinery is theater"* — and judge each finding on ROI + goal, not just truth. Two spiraling rounds in a row → the loop won't converge on merits; stop early and hand the user the framing question (optionally one fresh, cold third-model read). **Re-consolidate, don't patch-fold:** after any reversal, rewrite that `PLAN.md` section clean.

### Step 3 — Resolution (human gate #2)

**Before declaring APPROVED, run ONE cold check** — a fresh Codex thread, no memory of the rounds, the plain neutral prompt (nothing about "final round"/"downgrade nits"). Still REVISE cold = the convergence was biased; treat as real signal, not a formality. Converged = APPROVED-in-loop AND cold-check-clean.

**If APPROVED:** Present to the user — the final `PLAN_FILE`, a 3-bullet summary of what the argument improved, and the round count. Ask: *"Plan survived N rounds of Codex. Implement it now — Codex builds it (`/codex-build`), Claude builds it, or stop here?"* Only on a yes is code written. **No code is written during the loop.** If the user picks Codex, invoke the `codex-build` skill with `SPEC_FILE=PLAN.md` and the same `LOG_FILE` — roles flip (Codex writes, Claude reviews the diff) and the build rounds append to the same log.

**If MAX_ROUNDS hit without APPROVED (deadlock):** Do NOT pretend it converged. Surface the unresolved disagreements explicitly: list each point Codex still flags and Claude's counter-position. Hand it to the human to break the tie. This is a legitimate, useful outcome — a flagged disagreement beats a false "approved."

## Hard rules

- Codex is read-only EVERY round — `-s read-only` for the first call, `-c sandbox_mode="read-only"` for every resume (resume has no `-s`). It never writes. If you're tempted to give it write access, stop — that's a different skill.
- The loop ALWAYS terminates at `MAX_ROUNDS` (5) — but **direction beats round-count**: classify each round CONVERGING/SPIRALING; on a spiraling round step out (altitude question) instead of folding; re-consolidate after any reversal.
- **Codex is a reviewer, not the architect.** Claude stays the architect and gauges findings against the goal (and `intent.md` if present); don't reverse decisions at face value.
- Claude is the final arbiter on every REVISE — incorporate real flaws, reject the rest *with a logged reason*: off-intent and not-worth-the-ROI are valid rejections, not just factual wrongness. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- **One cold check** (fresh Codex, neutral prompt, no memory) gates every APPROVED — never lead the reviewer toward it.
- Code only after human gate #2.
- `LOG_FILE` is the deliverable — it tells the whole story of the argument. Keep it complete.

## What NOT to do

- Don't use this to review existing code — that's `/codex:review`.
- Don't pin a `-codex` model variant on ChatGPT-account auth — it 400s.
- Don't skip the log — the argument transcript is the most valuable artifact.
- Don't let Codex edit files or re-architect — it flags flaws bounded by the goal/intent; it doesn't redesign LOCKED decisions. Read-only, always.
- Don't lead the reviewer toward APPROVED — no "last round"/"downgrade nits" framing; same neutral prompt every round.
