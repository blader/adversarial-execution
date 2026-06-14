---
name: adversarial-execution
description: |
  Use when executing an implementation plan and nearing completion — before any task or the plan
  as a whole is marked fully done — and the work has correctness or user-facing surface that must
  be proven, not assumed. Triggers: a plan reaching its review gate, "verify before calling it
  done", wanting independent dual-model review of real execution quality and accuracy (Claude Code
  Opus 4.8 max + Codex gpt-5.5 xhigh) with browser-driven visual evidence, catching work that
  compiles and passes tests but doesn't actually work or doesn't match intent. Not for autonomous
  loop skills ($turborazor / $ultrarazor / $ultrareview / $turbovac).
metadata:
  version: 1.0.0
  date: 2026-06-13
---

# Adversarial Execution

This is the execution-time counterpart to `adversarial-planning`. Where `adversarial-planning`
bakes a review gate into the plan text, this skill *runs* that gate during execution — and runs it
hard. Before a task or the whole plan is marked done, two independent strong models review the
actual work adversarially, gathering their own evidence including browser-driven screenshots, and
you loop until both converge on "intent achieved and the work actually works."

The gate sits between "I think I'm done" and "this is done." It catches the failure modes that
green tests and a clean diff hide:

1. **Intent not achieved.** It compiles and tests pass, but the work doesn't do what the task or
   plan actually asked for.
2. **Not accurate in reality.** The code reads right but the running product misbehaves — the
   button does nothing, the number is wrong, the layout breaks. Code reading can't see this.
3. **Plan-intent drift.** The task closed green, but the work undermines the plan's overall goal.

**This skill activates whenever you are about to mark a task or plan done during execution.**

**Exemption — autonomous loop skills.** `$turborazor`, `$ultrarazor`, `$ultrareview`, and
`$turbovac` define their own convergence criteria and forbid review gates. Do not apply this gate
to them or to plans executed inside them.

## The two independent reviewers

Run two reviewers as independent processes — no shared context with the executor or with each
other. Two different models give diverse failure-mode coverage; **both must pass.**

**Claude Code CLI — Opus 4.8, max thinking:**

```bash
claude --model claude-opus-4-8 --effort max --output-format json "<review prompt>"
```

**Codex CLI — gpt-5.5, xhigh reasoning (read-only; attach screenshots; capture the verdict):**

```bash
codex exec --model gpt-5.5 -c model_reasoning_effort="xhigh" -s read-only \
  -i shot-1.png -i shot-2.png -o codex-verdict.txt "<review prompt>"
```

Do not silently substitute a different model, CLI, or a lower thinking/effort level. Confirm what
each CLI actually resolved to — for Claude Code, the `modelUsage` key in the JSON output names the
served model; for Codex, confirm the resolved model/effort in its run output — and **record the
selector, the resolved model, and the reasoning level for each reviewer.** If either CLI cannot run
its required model at the required effort, leave the gate unchecked and record the blocker instead
of downgrading silently.

Both reviewers are read-only on the work: they review and gather evidence, they do not fix. The
executor owns the fixes.

## What each reviewer gets

- The **full original plan text**, exactly as the executor is following it — not a summary. If it
  is long, attach it as a file path or paste the complete contents.
- The **stated intent / acceptance criteria** for the task and for the plan as a whole.
- **Access to the real working state** — the repo, the running app, logs, the test runner — so the
  reviewer inspects evidence itself rather than trusting a curated handoff. Pass external artifacts
  directly only when they are not locally reachable (CI logs, production screenshots, remote
  traces, customer reports).
- An **adversarial framing**: assume the intent was not met, assume the work is inaccurate, assume
  it drifts from the plan — then look for evidence otherwise, citing specific files / lines /
  behaviors / plan sections.

## Visual verification is mandatory for user-facing work

"Looks right in the code" is not evidence. For any user-facing surface, the review must drive the
actual product in a browser, exercise the relevant flows, and capture **screenshots** as evidence.

- The executor drives the browser, exercises each relevant flow/state, and captures a screenshot of
  each — then attaches those screenshots to both reviewers (Codex via `-i`, Claude Code via the
  prompt / file paths). A reviewer that can drive the browser itself should additionally gather its
  own visual evidence.
- The screenshots also land in the executor's own stream, so there is a visual artifact backing the
  verdict. **A pass with no screenshot of a user-facing change is not a pass.**

## Requirements the gate must enforce

- **Intent, accuracy, and plan-fit — all three.** Each reviewer answers: (a) did the work achieve
  the stated intent? (b) does it actually work, verified in the running product — visually, where
  user-facing? (c) does it advance the plan's overall intent, or close green while drifting? Any
  "no" is a fail.
- **Independence.** Fresh processes, no shared context. Self-review can't catch "this is basically
  the intent."
- **Full-plan context.** Each reviewer sees the entire original plan, not an executor-chosen
  excerpt. A review against a summarized plan is invalid and must be rerun.
- **Both reviewers must pass.** A pass requires an explicit "intent achieved, works, plan advanced"
  with evidence from *both* the Claude Code and Codex reviewers. A disagreement is signal, not
  noise — investigate the disputed gap; do not average the verdicts or take the friendlier one.
- **Binary outcome.** Each review returns either an explicit pass with evidence (including
  screenshots) or a concrete list of gaps. Vague hedging is a soft pass — treat it as not done and
  re-review with a sharper prompt or a stronger reviewer.

## Loop until convergence

1. Run both reviewers on the current work.
2. If either returns any gap, close it, then re-review with **fresh** sessions for both (not a
   continuation of the prior thread).
3. Repeat. Convergence is reached only when a fresh round from **both** reviewers surfaces no new
   gaps and both explicitly pass with evidence.
4. Only then mark the task or plan done.

Don't cap the loop. If the reviewers keep surfacing contradictory or oscillating gaps, the
acceptance criteria are ambiguous or the task is misaligned with the plan — sharpen the criteria or
revisit the task's place in the plan, rather than declaring done to escape the loop.

For long-running CLI reviews, monitor the session logs for progress instead of waiting silently;
report meaningful progress, and use the log evidence to decide whether to keep waiting or terminate
and rerun the same required command. Log monitoring never substitutes for the final binary verdict.

## What NOT to do

- Don't accept work as done because tests pass or the diff is clean — that is exactly what this
  gate exists to get past.
- Don't skip the browser / screenshot step for user-facing work. Code-only review can't see a
  broken interaction.
- Don't run only one reviewer. Both Claude Code (Opus 4.8 max) and Codex (gpt-5.5 xhigh) must pass.
- Don't downgrade the model or effort silently, or present a downgrade as equivalent.
- Don't merge a reviewer into the executor's context — independence is the point.
- Don't withhold the full original plan, and don't accept "looks aligned" — require explicit
  confirmation of intent + accuracy + plan-fit with evidence, or a concrete gap list.
- Don't apply this gate to the exempt autonomous loop skills.

## Relationship to adversarial-planning

`adversarial-planning` is the authoring-time skill: it bakes the review gate into the plan you
write. `adversarial-execution` is the execution-time skill: it runs that gate as the plan executes.
Use them together — a plan written under `adversarial-planning` is satisfied at its review gate by
running `adversarial-execution`.
