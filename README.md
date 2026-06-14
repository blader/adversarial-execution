# Adversarial Execution

An execution-time review gate: before any task or plan is marked done, two independent strong models review the *actual work* adversarially — gathering their own evidence, including browser-driven screenshots — and you loop until both converge on "intent achieved and the work actually works."

This is the execution-time counterpart to [adversarial-planning](https://github.com/blader/adversarial-planning). Where adversarial-planning bakes a review gate into the plan text, this skill *runs* that gate during execution.

## Why

Green tests and a clean diff hide three failure modes. The gate sits between "I think I'm done" and "this is done" to catch them:

1. **Intent not achieved.** It compiles and tests pass, but the work doesn't do what the task actually asked for.
2. **Not accurate in reality.** The code reads right but the running product misbehaves — the button does nothing, the number is wrong, the layout breaks. Code reading can't see this.
3. **Plan-intent drift.** The task closed green, but the work undermines the plan's overall goal.

## How it works

1. Run **two independent reviewers** as fresh processes — no shared context with the executor or each other:
   - **Claude Code** — Opus 4.8, max thinking
   - **Codex** — gpt-5.5, xhigh reasoning (read-only)
2. Each reviewer gets the **full original plan**, the stated intent/acceptance criteria, and **access to the real working state** (repo, running app, logs, tests).
3. For any user-facing surface, the review **drives the product in a browser** and captures screenshots as evidence — "looks right in the code" is not a pass.
4. Each reviewer answers all three questions — intent achieved? actually works? advances the plan? — and returns either an explicit pass with evidence or a concrete gap list.
5. **Loop until convergence:** close every gap, then re-review with fresh sessions for both. Done only when a fresh round from *both* reviewers surfaces no new gaps.

**Both reviewers must pass.** A disagreement is signal, not noise — investigate the disputed gap rather than averaging the verdicts.

## Install

### Claude Code

```bash
git clone https://github.com/blader/adversarial-execution.git ~/.claude/skills/adversarial-execution
```

### Codex

```bash
git clone https://github.com/blader/adversarial-execution.git ~/.codex/skills/adversarial-execution
```

### One-liner (auto-detect)

```bash
SKILLS_DIR="${CLAUDE_SKILLS_DIR:-${CODEX_SKILLS_DIR:-$HOME/.claude/skills}}" && \
git clone https://github.com/blader/adversarial-execution.git "$SKILLS_DIR/adversarial-execution"
```

Then restart your agent. The skill activates automatically whenever you are about to mark a task or plan done during execution. You can also invoke it explicitly:

> Verify this is actually done before I call it.
> Run the adversarial execution gate on this work.

## Scope

Use it when nearing completion of an implementation plan and the work has correctness or user-facing surface that must be proven, not assumed.

**Not for autonomous loop skills** (`$turborazor`, `$ultrarazor`, `$ultrareview`, `$turbovac`) — they define their own convergence criteria and forbid review gates.

## License

MIT
