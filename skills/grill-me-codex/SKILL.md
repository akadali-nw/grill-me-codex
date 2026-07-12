---
name: grill-me-codex
description: Two-act plan hardening. ACT 1 (you ↔ Claude) — Claude consults you like a non-technical client, one simple question at a time (brain-dumps welcome — Claude captures and refines them), owning the technical HOW itself and capturing your goal in intent.md. ACT 2 (Claude ↔ Codex) — Claude translates intent.md into a technical PLAN.md and OpenAI Codex adversarially reviews it in a read-only sandbox (VERDICT:APPROVED/REVISE) bounded by your intent, Claude revises and re-submits to the SAME Codex session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-me-codex", "grill me then have codex review", "grill me and stress-test the plan", "interview me about this plan then get a second model on it", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants both alignment AND a cross-model sanity check before implementation. Every run captures the intent to a durable idea registry (notes/ideas/<slug>/ + top index.md); the review arg sets what happens next: review=off (stop, parked, no Codex), review=light (one quick surface pass), review=full (default, the intensive multi-round loop). Builds on Matt Pocock's grill-me (MIT). For the docs-aware variant use /grill-with-docs-codex; if you already have a plan and want only the Codex review use /codex-review. NOT for reviewing already-written code (use /codex:review) and NOT for trivial changes.
---

# Grill-Me-Codex — Get Grilled, Then Get Reviewed

Two acts, two different jobs:

- **Act 1 fixes the #1 failure mode: building the wrong thing.** Claude consults *you* like a non-technical client — plain language, no jargon — until your intent is captured in `intent.md`. (Act 1 is Matt Pocock's `grill-me`, used under MIT — see `THIRD-PARTY-NOTICES.md`.)
- **Act 2 fixes the #2 failure mode: a plan that sounds right but breaks.** Claude translates `intent.md` into a technical plan (`PLAN.md`) and a *different model* (Codex) adversarially attacks it — bounded by your intent, cross-model, no echo chamber.

You enter at two points only: the consultation, and signing off the converged plan. Codex is read-only the whole time and never touches a file. Everything relayed to you stays non-technical.

---

## Idea registry — where every intent lives (all modes)

Every consultation produces an intent worth keeping, built or not. **All of them — and all their thinking — live in one place: the user's idea registry**, never in a project's codebase (that repo has its own README/CLAUDE.md; the *why* behind an idea doesn't belong there).

- **One folder per idea:** `<notes>/ideas/<slug>/` holds `intent.md` (always), and once a plan exists, `PLAN.md` + `PLAN-REVIEW-LOG.md` too. This is the durable home for *all* of them at every status — even a built idea's plan stays here; only its **code** goes to a project repo. *(The concrete notes path may be in the user's config/memory — check there; else ask.)*
- **One index at the top:** `<notes>/ideas/index.md` — a single table (`Idea · Hook · Captured · Status · Built-in`). This is your scan surface: **read the index, never walk the folder tree.** Create it if absent.
- **Dedup when an idea comes in:** when the user names or writes a new idea, check `index.md` first — if it's already there, offer to **resume / build on** it instead of duplicating. (Read the index only, once, when there's actually an idea to match — not on unrelated runs.)
- **Status ladder:** `parked` (intent only) → `planned` (intent + plan + log, reviewed but not built) → `built` (all three still here; code shipped to a repo, `Built-in` links to it) / `dropped` (kept for the record).
- **These are documents, not code — auto-commit AND auto-push** the registry (intent / plan / log / index) to the notes vault. No troubleshooting risk; keeping thoughts synced beats ceremony. **The code implementation of an idea follows the usual rule: commit only, never auto-push.**

---

## ACT 1 — CONSULT (you ↔ Claude)

Run this like a **non-technical client consultation.** The user knows the *problem*, not the *how* — treat them as a smart client who is not a software or systems engineer. Your job is to draw out what they actually want and capture it, never to make them design.

**How to run it:**

- **Ask simple, plain questions — one at a time.** Short, single-topic, no jargon, no architecture semantics. Wait for the answer before the next. Verbose or compound questions lose the client.
- **Accept messy input.** They may dump random, unstructured thoughts — that's expected and welcome. Capture them, refine them, and reflect the cleaned-up version back for confirmation. Organizing the mess is *your* job, not theirs.
- **Probe intent and feasibility, listen, nudge.** Draw out goals, priorities, what "done" looks like, and what's explicitly *not* wanted. Talk feasibility in plain terms. Nudge toward sane direction — and if they wander into a tangent, gently pull them back to the goal.
- **Own the HOW; explain, don't extract.** Implementation decisions — algorithms, naming, syntax, data structures, complexity — are *yours*. Decide them; never ask the client to adjudicate them. When they ask "how does this work" or "why did you do it that way," that is a request to *understand*, not a signal you erred and must reverse. Explain it plainly, then keep the decision — they'll rule "makes sense" or "let's do it another way." A question is not a correction. **But *owning* a decision isn't *hiding* it** — tell them plainly what you'll build and which tools you'll use ("to notify you, I'll use ntfy"). Naming the approach in plain words is informing them, not making them adjudicate — and it's how they catch things early.
- **Explore the code yourself** when a question can be answered there instead of asked.

**When you believe the client is fully heard — before spending tokens on the full playback, check in first.** Say plainly: *"I think I've got everything I need. Anything you want to add, or ready for me to summarize it back?"* Then wait. Only once they say ready do you produce the summary — don't summarize, collect three more additions, and re-summarize; that round-trip is the token waste to avoid.

**Once they're ready**, play the whole thing back to them **non-technically — every corner** — so they confirm their intent was captured as they meant it. Then write **`intent.md`** (the authoritative, plain-language statement of what they want — the constitution the rest of the flow is measured against):

```markdown
# Intent: <task>
_Captured with <user> — plain language, non-technical. This is the goal of record._

## What they want
<the outcome, in the client's own terms>

## Why
<the problem it solves / the motivation>

## Priorities
<what matters most, in order — what to optimize for>

## Non-goals
<what is explicitly out — bounds the client set>

## Success looks like
<how the client will know it worked>
```

The intent is **non-technical and stable** — it changes only when the *client* changes their mind, never for technical churn.

**Capture it now — in every mode — to the idea registry** (above): write `intent.md` into `<notes>/ideas/<slug>/` and add/update its `index.md` row. Then branch on `review`:

- **`review=off`** → done. Status `parked`. Present the idea back to the client in plain language and stop — no `PLAN.md`, no Codex. (Revisit later from `index.md`; re-run at `light`/`full`.)
- **`review=light` / `review=full`** → continue. Translate the intent into `PLAN.md` **in the same idea folder** (Claude's technical work, not the client's — what Codex bites), init the log there, and run Act 2. Status is finalized at sign-off: `built` (code shipped to a repo), `planned` (reviewed, not built), or `dropped`.

```markdown
# Plan: <task>
_Claude's technical translation of intent.md — grounded to it._

## Goal
<one paragraph — the technical objective, serving intent.md>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices — name them so Codex has something to bite. Mark client-locked ones LOCKED.>

## Risks / open questions
<anything still genuinely open>

## Out of scope
<bounds, per intent.md non-goals>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Act 1 (consult) complete — intent.md captured + confirmed with the user, PLAN.md drafted. MAX_ROUNDS=<n>.
```

**Translation discipline — the plan must trace to the intent:**
- **No silent new scope.** Every element of `PLAN.md` must trace to something in `intent.md`; the plan may not add a goal or feature the intent doesn't have. If translating surfaces something genuinely needed, that's a **new intent item** — surface it to the client non-technically, add it to `intent.md` first, then plan it.
- **Default to the smallest realization that satisfies the intent.** When a plain intent item has several technical solutions of different weight (e.g. "ping me when done" → an in-session ping *or* standing up an external notification service), pick the **minimal** one by default. Don't reach for the heavier build; if a bigger option might genuinely be wanted, raise it as an *optional* choice non-technically — never bake it in. *(Real miss: "ping when done" got specced as an external `ntfy` service the client never asked for, caught only because the client noticed.)*
- **Plan playback — before Act 2, play the whole plan back in plain language and get the OK.** Not the technical doc: a plain-language walk-through of *how* you'll build it, naming the key approach choices ("to notify you, I'll use ntfy"; "the worker lives on the Mini") and how each intent item was resolved ("your 'ping when done' → an in-session ping, not an external service"). Ask: *"here's how I'd build it — anything you don't want, or want different?"* Adjust on their word (update `intent.md` if intent moved). Only then run Act 2 — catching a wrong approach here is free; catching it after Codex has hardened it isn't.

---

## ACT 2 — REVIEW (Claude ↔ Codex)

Now hand the locked plan to Codex for adversarial review. Same engine, mechanics verified end-to-end (2026-06-04). `PLAN.md` and `PLAN-REVIEW-LOG.md` live in the idea folder (`<notes>/ideas/<slug>/`) — run the review with that folder as the working dir (and, if the plan targets an existing codebase, with that repo readable to Codex). They stay in notes and auto-push; the project repo only ever gets code.

### Review modes
- **`review=full` (default):** everything below — up to `MAX_ROUNDS`, trajectory check, cold check.
- **`review=light`:** ONE surface pass, then stop. Use the light prompt below, run Round 1 only, present the findings, Claude fixes the clearly-worth-it ones, done. No resume loop, no trajectory check, no cold check — a single pass can't spiral, so the heavy discipline doesn't apply. A gut-check, not a hardening.
- **`review=off`** never reaches Act 2 — it captured the intent to the idea registry (folder + index), `Status: parked`. No PLAN.md.

**Light-mode review prompt** (replaces the full prompt when `review=light`):
> You are doing a quick sanity check on an implementation plan (read-only). Read `intent.md` (the goal) and `PLAN.md`. Flag ONLY clear, serious problems — security holes, broken assumptions, major missing pieces. Do NOT nitpick, hunt edge cases, or propose redesigns. If it's basically sound, say so. End with EXACTLY one line: `VERDICT: APPROVED` or `VERDICT: REVISE`.

### Prerequisites (verify once, fast)
- `codex --version` ≥ 0.130 (older CLIs error on the default `gpt-5.5` model).
- Codex authenticated (prior `codex login`; ChatGPT account is fine). On auth/model error, surface it — don't silently retry.
- Do NOT pin `-m`. Use the config default. Pinning `gpt-5.x-codex` variants 400s on ChatGPT-account auth.
- **Echo the active model before Round 1** so the user can confirm: read the `model` line from `~/.codex/config.toml` (if absent, report "CLI default"). State it alongside the resolved tunables, e.g. `Reviewer model: CLI default (config unpinned) — codex-cli 0.137.0`. If the user objects, stop and let them adjust config before burning a review round.

### Tunables (read from args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `review` | `full` | Every mode captures the intent to the idea registry. `off` = stop there (parked, no Codex); `light` = one gentle surface pass; `full` = the intensive loop below. See **Review modes** + **Idea registry**. |
| `MAX_ROUNDS` | `5` | Hard cap on review rounds (`full` mode). The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan Act 1 produced. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. The artifact. |

If invoked with e.g. `rounds=3`, use that for `MAX_ROUNDS`. Echo resolved values before starting.

### The review prompt (sent each round)
> You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to redesign it. Read `intent.md` first (the authoritative goal — everything must serve it), then the plan at `PLAN.md` and any repo files you need (you are read-only). Decisions marked LOCKED are settled by the product owner: flag one ONLY if it is genuinely broken (a correctness, security, or feasibility failure), and never propose an alternative design for it — you are a reviewer, not the architect. For everything else, identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. Do NOT modify any files. End your reply with EXACTLY one line: `VERDICT: APPROVED` if the plan is sound enough to implement, or `VERDICT: REVISE` if it still has material problems.

### Round 1 — fresh session (capture `thread_id`)
```bash
codex exec -s read-only --json -o /tmp/codex-verdict.txt "$(cat REVIEW_PROMPT)" \
  < /dev/null 2>/dev/null | grep '"type":"thread.started"'
```
Parse `thread_id` from the `{"type":"thread.started","thread_id":"..."}` line → that's `THREAD_ID`. The critique is in `/tmp/codex-verdict.txt`. Confirm success by the verdict file + a `thread.started` line; if neither appears, the run failed (auth/model) — stop and tell the user. `2>/dev/null` suppresses cosmetic MCP/auth stderr noise. **`< /dev/null` is mandatory:** `codex exec` reads stdin *in addition to* the prompt arg, so under a non-interactive driver (Claude Code's Bash tool, CI, any non-TTY pipeline) it blocks forever waiting on stdin EOF — a silent ~0% CPU hang. The redirect gives it immediate EOF.

### Rounds 2..MAX — resume the SAME session (Codex remembers its prior critiques)
```bash
# resume REJECTS -s. Force read-only via -c sandbox_mode, or Codex inherits
# config.toml (possibly danger-full-access) and could WRITE files. This is the
# single most important safety line in the skill — verified 2026-06-04.
codex exec resume "$THREAD_ID" -c sandbox_mode="read-only" --json \
  -o /tmp/codex-verdict.txt \
  "I revised the plan. Re-review PLAN.md — check whether your prior findings are addressed and flag anything new. End with VERDICT: APPROVED or VERDICT: REVISE." \
  < /dev/null 2>/dev/null >/dev/null
```
Both `codex exec` and `codex exec resume` support `--json` and `-o/--output-last-message`. The `< /dev/null` redirect is required on the resume call too — same non-interactive stdin hang as Round 1.

**Timeout guard (both rounds):** run every `codex exec` / `codex exec resume` with a 10-minute ceiling so any future stall fails loud instead of hanging silently. Via Claude Code's Bash tool, pass `timeout: 600000` on the tool call (the default 2-minute tool timeout is too short for real reviews and would kill them mid-run). In a plain shell, prefix the command with `timeout 600` (Linux / Git Bash) or `gtimeout 600` (macOS via coreutils — stock macOS has no `timeout`). If the ceiling trips, treat it as a failed run: stop and tell the user rather than retrying blind.

### Each round, after Codex returns
1. Read `/tmp/codex-verdict.txt`; append to `LOG_FILE`: `## Round <n> — Codex` + the full critique.
2. **Gauge every finding against `intent.md` before acting.** Claude is the architect; Codex advises. A finding that would pull the plan *off intent* — or redesign a LOCKED decision that isn't actually broken — is **rejected and logged "off-intent."** Don't take Codex at face value and change every decision; that's how the plan spirals into Codex's idea instead of the client's.
3. Grep the last line for the verdict:
   - `VERDICT: APPROVED` → break to Resolution (converged).
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (final arbiter — incorporate real flaws, reject off-intent and low-ROI ones *with a logged reason*). Revise `PLAN_FILE`. Append `### Claude's response` to `LOG_FILE`: what changed, what was rejected, why.
   - **If a finding challenges a LOCKED (client) decision and is genuinely broken** → do NOT silently reverse it. Relay it to the client **non-technically**, flagged *"this needs your input,"* capture their call, update `intent.md` if their intent moved, then continue. The client owns their decisions.
   - Increment round.
4. Before sending the next round, **re-check `PLAN.md` still serves `intent.md`.** If a revision drifted, realign it first.
5. If round > `MAX_ROUNDS` → break to Resolution (deadlock).

### Act 2 discipline — direction over round-count

`MAX_ROUNDS` (5) is a *ceiling*, not a target. The real question every round is **which way is this heading — closing in, or circling?** The loop has a structural bias you must counter: a reviewer's only way to add value is to return `REVISE` with new findings; it never says "delete half of this." Left unchecked the plan grows monotonically, and an arbiter who only checks findings for *correctness* folds hardening forever.

**Trajectory check — before you fold each round, classify it in `LOG_FILE`, one word: CONVERGING or SPIRALING.**

Converging (keep going — spend the rounds):
- Codex *concedes or closes* prior findings, not just raises new ones.
- New findings are *tightening* — spec gaps, edge cases — not *architectural* (no design reversal).
- Finding count trending down.

Spiraling (any one — stop folding):
- The **same finding recircles** in new clothes across rounds.
- The round **only added guards** — no decision changed.
- The plan **grew again** (more machinery) with nothing resolved.
- A "fixed" finding **reappears reframed**.

**On a spiraling round, do NOT fold-and-continue. Step out:**
- **Ask the altitude question, not the next detail:** *"what's the irreversible-harm profile here — do the elaborate and the simple version actually differ on it? If not, the machinery is theater."* Most spirals are guards piled on a boundary that was *cooperative* (trusted, not mechanically enforced) all along.
- **Per finding, judge ROI, not just truth:** does it *change a decision*, or just *add a guard*? A guard on a cooperative boundary usually isn't worth it — reject it in the log with a reason. "Correct but not worth it" is a valid rejection; arbiter means ROI judgment, not fact-checking.
- **Two spiraling rounds in a row → the loop won't converge on merits.** Stop early (you don't owe it the full 5) and hand the user the framing question — optionally with one *fresh, cold* third-model read (no memory of the rounds) on that single question. This ends more spirals than any further round.

**Re-consolidate, don't patch-fold.** After any round that *reverses* a decision, rewrite that section of `PLAN.md` clean. Surgical edits around now-dead text leave the plan describing two contradictory designs at once — the document churns exactly as the design did.

### Resolution (you sign off — final gate)
- **Before you declare APPROVED, run ONE cold check.** Spawn a *fresh* Codex thread — no memory of the rounds, the plain neutral review prompt, nothing about "final round" or "downgrade nits." A same-session APPROVE can just echo your own framing; if a cold thread still says REVISE, the convergence was biased — treat its findings as real signal, not a formality. Converged = APPROVED-in-loop **and** cold-check-clean.
- **APPROVED:** relay to the client **non-technically** — confirm `intent.md` is fully served, give a 3-bullet plain-language summary of what the two acts improved, and the round count. Ask: *"This matches everything in intent.md and survived N rounds of Codex. Build it now — hand it to Codex via `/offload` (medium/high), Claude builds it, or stop here?"* Code only on a yes. **No code is written during either act.**
- **MAX_ROUNDS hit without APPROVED (deadlock):** do NOT fake convergence. List each unresolved point + Claude's counter-position; hand it to the user to break the tie. A flagged disagreement beats a false "approved."

### ACT 3 (optional) — BUILD

If the user picks Codex: hand the APPROVED `PLAN.md` to Codex via `/offload` — **medium** (Codex writes, Claude reviews the diff / verifies builds+tests) or **high** (conserve tokens, commit gate as the net). Codex implements against the plan; the built **code** goes to its project repo, while the plan + log stay in the idea registry. If the user picks Claude, implement directly as usual. Either way, set the idea's `Status` to `built` in `index.md` and link the repo.

---

## Hard rules
- Act 1 is a **non-technical client consultation** — plain language, simple one-at-a-time questions, Claude owns the HOW. Capture intent in `intent.md` and confirm it with the user before Act 2.
- `intent.md` is the constitution — non-technical, stable, the goal every round is measured against. `PLAN.md` is Claude's technical translation of it.
- **Every mode captures to the idea registry** (`<notes>/ideas/<slug>/` + top `index.md`): all thinking (intent / plan / log) lives there for every status; a built idea's **code** goes to a project repo, its thinking never does. Dedup against `index.md` when an idea comes in. Registry docs auto-commit + **auto-push** (they're notes); the code build is commit-only, no push.
- **Plan traces to intent.** `PLAN.md` adds no goal or feature absent from `intent.md`; default to the smallest realization that satisfies each intent item; surface anything heavier or new to the client (non-technically) and fold it into `intent.md` first — never silently. **Before Act 2, play the plan back in plain language** (naming approach + tools, e.g. "I'll use ntfy") so the client catches/redirects before Codex hardens it. Naming the approach plainly is informing, not "being technical."
- **`review` knob:** `off` = capture intent, stop (parked, no Codex); `light` = one gentle surface pass (no loop, no cold check); `full` (default) = the intensive loop. Echo the resolved mode at kickoff.
- Codex is read-only EVERY round — `-s read-only` first call, `-c sandbox_mode="read-only"` on every resume (resume has no `-s`). It never writes.
- **Codex is a reviewer, not the architect.** Claude stays the architect and gauges every finding against `intent.md`; don't take Codex at face value and reverse every decision. LOCKED decisions change only if genuinely broken — and that goes back to the client, non-technically.
- The loop ALWAYS terminates at `MAX_ROUNDS` (5) — but **direction beats round-count**: classify each round CONVERGING/SPIRALING; on a spiraling round step out (altitude question) instead of folding; re-consolidate after any reversal.
- Claude is final arbiter on every REVISE — incorporate real flaws, reject the rest *with a logged reason*: **off-intent** and **not-worth-the-ROI** are both valid rejections, not just factual wrongness. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- **One cold check** (fresh Codex, neutral prompt, no memory) gates every APPROVED — never lead the reviewer toward it.
- Everything relayed to the client is **non-technical**, flagged when it needs their input.
- Code only after the user's final sign-off.
- `LOG_FILE` is the deliverable — keep the whole argument.

## What NOT to do
- Don't review already-written code — that's `/codex:review`.
- Don't turn the consultation technical — no jargon, no asking the client to pick algorithms/naming/syntax. Explain the HOW plainly; decide it yourself.
- Don't treat a client question ("why that way?") as a correction — answer it, keep the decision, let them rule.
- Don't let Codex re-architect — it flags flaws bounded by `intent.md`; it doesn't redesign LOCKED decisions.
- Don't lead the reviewer toward APPROVED. Never tell Codex "this is the last round" or "separate blocking flaws from nits" — that manufactures convergence. Same neutral prompt every round; for a clean read use the cold check.
- Don't pin a `-codex` model variant on ChatGPT-account auth — it 400s.
- Don't let Codex edit files. Read-only, always.
- Don't skip Act 1 — capturing intent is half the value.
