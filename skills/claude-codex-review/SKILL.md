---
name: claude-codex-review
description: >-
  Get an independent second-opinion review from Codex (the `codex` CLI) on code
  or implementation plans that Claude just produced. Use this whenever Claude has
  just finished writing or substantially editing code, or has just drafted an
  implementation/design plan, and an independent cross-check would add value — and
  especially whenever the user says things like "have codex review this",
  "cross-check with codex", "get a second opinion", "codex 审一下", "codex 过一遍",
  or "让 codex 看看". Routes Claude's output to `codex review` (for code diffs)
  and `codex exec` (for plans), then evaluates Codex's findings critically rather
  than accepting them blindly.
---

# Claude-Codex Review

Hand code or plans Claude just produced to a **different model/agent (Codex) for an independent review** — a cross-perspective second opinion.

The value is that the author and the reviewer aren't the same model: Claude can have blind spots on its own output, while Codex runs a different training and reasoning path and often catches what Claude misses on self-review. But it's only a second opinion, not a verdict — see "Evaluate every finding" below.

## When to trigger

- You (Claude) just wrote or substantially changed code → proactively offer a codex review.
- You just drafted an implementation/design plan → proactively offer to have codex review it.
- The user explicitly asks: "codex 审一下", "过一遍", "cross-check", "second opinion".

**Trigger etiquette: don't run it automatically.** After finishing, ask "want me to have codex review this?" and run only once the user agrees. A review takes time and costs codex tokens — leave the decision to start it to the user.

## Pre-flight checks

Confirm in order; if a step is missing, tell the user instead of forcing it:

1. **codex available**: `command -v codex`. If absent, tell the user to install the codex CLI first.
2. **Reviewing code needs a git repo**: `git rev-parse --is-inside-work-tree`. If not a repo, use "Path B (plan/file review)" or tell the user.
3. **Confirm there are changes to review**: `git status --porcelain`. No uncommitted changes means nothing to review — don't run empty.
4. **Auth**: if codex reports an auth error, tell the user to run `codex login`.

## Path A — review code (git diff)

**Preferred — pure-prompt mode with focus instructions** (most flexible; best results in testing):

```bash
codex review "Review the current uncommitted changes only. <your focus instructions>"
```

codex locates the uncommitted changes itself and folds your focus into the review. For robotics/control code especially, naming the context pays off — in testing, adding "unsafe actuator behavior / real-time / division by zero" focus made codex surface deeper issues (unclamped output that could command dangerous motion, unbounded history causing missed real-time deadlines), not just surface-level bugs.

**Flag mode (no focus prompt):**

```bash
codex review --uncommitted      # staged + unstaged + untracked
codex review --base <branch>    # whole feature branch vs base
codex review --commit <sha>     # a specific commit
```

⚠️ **Key: `--uncommitted` / `--base` / `--commit` are mutually exclusive with a PROMPT** (errors with `cannot be used with '[PROMPT]'`). To pass focus instructions, use the pure-prompt form above and let codex locate the changes itself.

codex defaults to `approval: never` and `sandbox: read-only` — it won't modify your files, so it's safe to call non-interactively.

## Path B — review a plan / doc (non-diff)

A plan is usually text, not a git diff. Write it to a file and have codex read and critically review it:

```bash
codex exec --skip-git-repo-check "Critically review the implementation plan at <plan-file-path>. \
Identify: correctness gaps, unstated assumptions, risky or out-of-order steps, missing edge cases, \
and simpler or safer alternatives. Tag each issue with a severity (P1/P2/P3). Do not rubber-stamp." </dev/null
```

**Tested notes (codex 0.135.0):**

- **Requires a git repo by default**: when the plan is in a non-git dir (e.g. `/tmp`), you must add `--skip-git-repo-check`, otherwise it errors `Not inside a trusted directory`. If the plan lives in your project repo, omit the flag.
- **Block stdin**: `codex exec` tries to read stdin (prints `Reading additional input from stdin...`); use `</dev/null` to prevent hanging.
- **For plans, codex may search the web and cite sources**: in testing it searched relevant papers/official docs and returned real `Sources` URLs, sometimes with a `Safer Sequence` re-ordering of steps. Extra value for design review, but slower and more token-heavy (~53k tokens in one test) — worth flagging to the user before running.
- **Output format matches Path A**: the conclusion is after the final `codex` marker; `Findings` use `[P1]/[P2]/[P3]` + line numbers, printed twice, needs dedupe.

## Parsing codex output

codex stdout is mostly noise; only the last section is the review conclusion. Extraction rules:

- **The real review is after the final standalone `codex` line.** Ignore the metadata block above it (model / sandbox / session id…) and the mid-run `exec` tool-call trace.
- Conclusion structure: a one-line summary, then `Full review comments:`, then entries:
  ```
  - [P1] title — path:line
    detailed explanation…
  ```
- **Severity**: `[P1]` highest (critical), `[P2]`/`[P3]` descending. Keep the level in your summary.
- **The conclusion is usually printed twice** — dedupe, keep one copy.
- Ignore macOS environment noise: `confstr() failed` / `couldn't create cache file '/tmp/xcrun_db-...'` (harmless warnings from git running in a read-only sandbox, not review content).

## Evaluate every finding — no blind trust

**This is the core of the skill.** Codex is a second opinion, not a verdict. Treat it like a human code review (see superpowers' `receiving-code-review`: technical rigor, not performative agreement).

For **each** finding codex raises:

- **Valid** → adopt it, make the fix, and briefly note what you changed.
- **Dubious / wrong** → don't change it; push back explicitly with a technical reason. e.g. "codex worries about a null deref here, but the caller already validates at L40, so this path is unreachable."
- **Uncertain / a design trade-off** → surface it to the user with both sides and let them decide.

Never silently apply everything, and never silently ignore everything — both waste the cross-review.

## Summary format for the user

```
## Codex review (target: <uncommitted changes / plan / branch xxx>)
**Command**: <the actual codex command run>

### ✅ Adopted & fixed (X)
- [issue] → [what I changed]

### ❌ Pushed back (Y)
- [codex's point] → [why I think it doesn't hold]

### 🤔 Your call (Z)
- [trade-off] → [option A vs B]
```

Open with a one-line tally: "codex raised N — I adopted X, pushed back on Y, Z are yours to call." Let the user see the whole picture before the details.
