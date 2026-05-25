---
name: adversarial-execution
description: |
  When writing or updating any plan, bake an adversarial subagent review gate into it. A
  task is not marked done until an independent subagent verifies that the task's stated
  intent was actually achieved by the work AND that the work advances the overall plan's
  intent — not just that the work has no obvious defects, and not just that the task
  succeeded in isolation. Adding this gate dramatically raises plan quality and lengthens
  runs in the right way: fewer prematurely-closed tasks, fewer locally-correct tasks that
  drift away from the plan's actual goal.
metadata:
  version: 1.4.0
  date: 2026-05-24
---

# Adversarial Execution

You add adversarial subagent review gates to plans before they get executed. The gate sits
between "I think I'm done with this task" and "I'm marking this task done." It catches the
two most common failure modes of long-running plans:

1. **Local-intent failure.** The executor thinks the task is done — code compiles, tests
   pass — but the task's actual stated intent was never achieved.
2. **Plan-intent drift.** The task's local intent was achieved, but the way it was achieved
   doesn't advance (or actively works against) the plan's overall goal. The task closes
   green; the plan quietly drifts.

The gate catches both. Later tasks then can't build on unstable ground or on work that
satisfied a checkbox but violated the spirit of the plan.

**This skill activates whenever you are writing, editing, or revising any plan.**

## The pattern to insert

The plan must enforce that every task passes an independent adversarial review before it's
marked done. The review dispatches a fresh subagent with no context from the executing
session, hands it:

- the **full original plan text** exactly as the executor is following it, not a summary,
- the **task's stated intent / acceptance criteria**,
- the **work product** (diff, file contents, command output, screenshot — whatever the task
  produced),

and asks it to verify adversarially that the work achieved the task's intent **and** that
the work is consistent with the plan's overall intent. A task moves to done only if the
reviewer can confirm both, with evidence.

Use the strongest available reviewer by default. For Codex subagents, request the latest
available frontier model; in the current tool surface that means `gpt-5.5` with
`reasoning_effort: xhigh`. For Claude Code review gates, request Opus 4.7 when available.
If the requested model is unavailable, use the strongest available frontier alternative,
record the requested model, resolved model, and reasoning/thinking level, and do not present
the downgrade as equivalent.

If the plan is long, pass the complete plan as an attached file path or paste the full
contents into the review prompt. Do not shorten it to "goal, principles, and anti-patterns"
unless the original plan itself consists only of those sections. The reviewer must be able
to compare the work against every explicit requirement, exception, invariant, and sequencing
rule in the original plan.

Express this once in the plan — usually a single "Review gate" / "Validation discipline"
section near the top — and have every task fall under it implicitly. Don't repeat the gate
copy per task. The wording is yours; the requirements below are what make the gate work.

## Requirements the gate must enforce

- **Two intents, both checked.** The reviewer answers two questions, not one:
  - (a) Did this work actually achieve the stated intent of the task?
  - (b) Does the work advance the plan's overall intent — or did the task close in a way
    that satisfies (a) while drifting from the plan's goal (violating an anti-pattern,
    reintroducing what the plan was trying to remove, preserving what the plan was trying
    to eliminate, etc.)?
  Either question coming back "no" is a fail. (b) is the more important check — it's the
  one that catches plans drifting silently across many locally-correct tasks.
- **Full-plan context.** The reviewer receives the entire original plan, not an excerpt
  chosen by the executor. The executor may additionally call out relevant sections, but
  those callouts never replace the full plan. Reviews that only see a summarized plan are
  invalid and must be rerun with full-plan context.
- **Intent before defects.** Defects matter only insofar as they prevent the intent (local
  or plan-level) from being achieved. A clean diff that doesn't accomplish the task is a
  fail. A clean diff that accomplishes the task but undermines the plan is also a fail,
  even if no individual line is wrong.
- **Independence.** The reviewer is a fresh subagent (or a different model entirely) — no
  shared context with the executor. Self-review can't catch the executor's rationalization
  that "this is basically the intent" or "this is in the spirit of the plan."
- **Adversarial framing.** The review prompt instructs the reviewer to treat the work as
  guilty until proven correct on both axes: assume the task intent was not met, assume the
  work violates the plan's intent, then look for evidence otherwise. Cite specific files /
  lines / behaviors / plan sections.
- **Binary outcome.** The review returns either an explicit "both intents satisfied" with
  evidence, or a list of concrete gaps (task-level, plan-level, or both). Vague hedging
  is a soft pass — treat it as not done and re-review with a sharper prompt or a stronger
  model.
- **Loop on failure.** If gaps come back, close them and dispatch a fresh review (not a
  continuation of the prior thread). Keep looping until the review confirms both intents
  were met. If reviews keep surfacing contradictory gaps, either the task's intent is
  ambiguous or it's misaligned with the plan — rewrite the acceptance criteria or revisit
  the task's place in the plan, rather than capping the loop.

## What NOT to do

- Don't frame the review as bug-hunting. Bug-hunting misses both the "this code is fine
  but doesn't do the task" and "this task is done but undermines the plan" failure modes.
- Don't withhold the full original plan from the reviewer. The reviewer can't catch
  plan-level drift from a summarized goal/principles/anti-patterns excerpt because the
  missing details often contain the exception or sequencing rule that matters.
- Don't merge the review into the executor's context — independence is the point.
- Don't accept work as done just because tests pass. Tests verify what the executor
  thought they were building; the gate verifies the intent at both levels.
- Don't skip the gate on tasks that look obvious. Obvious tasks are exactly where
  executors substitute their own reading of the plan for the plan itself.
- Don't accept "looks aligned" as a pass — require either explicit confirmation that both
  intents were achieved, with evidence, or a list of concrete gaps.
