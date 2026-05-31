<!-- Language switch -->
**English** | [中文](README.zh-CN.md)

# claude-codex-review

A [Claude Code](https://claude.com/claude-code) skill that hands code or plans **Claude just produced** to **Codex** (the `codex` CLI) for an independent second-opinion review.

**Why:** the author and the reviewer shouldn't be the same model. Claude can have blind spots on its own output; Codex runs a different model and training path, so it often catches what Claude misses on self-review.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [`codex` CLI](https://github.com/openai/codex), authenticated via `codex login`. Tested with `codex-cli 0.135.0`.

## Install

In Claude Code, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add maoningge/claude-codex-review
/plugin install claude-codex-review@claude-codex-review
```

Restart Claude Code (or run `/reload-plugins`) so the skill is picked up.

## Usage

The skill triggers when Claude has just written/edited code or drafted a plan and a cross-check adds value — or when you explicitly ask: *"have codex review this"*, *"cross-check with codex"*, *"second opinion"*, *"codex 审一下"*, *"codex 过一遍"*…

Claude **asks before running** (a review costs codex tokens); you confirm, then it runs the right command and evaluates the findings critically.

### Path A — review code (git diff)

Preferred, with focus instructions (pure-prompt mode):

```bash
codex review "Review the current uncommitted changes only. <focus instructions>"
```

Flag mode (no focus prompt):

```bash
codex review --uncommitted        # staged + unstaged + untracked
codex review --base <branch>      # whole feature branch
codex review --commit <sha>       # a specific commit
```

> ⚠️ `--uncommitted` / `--base` / `--commit` are **mutually exclusive with a PROMPT**. To pass focus instructions, use the pure-prompt form above and let codex locate the changes itself.

### Path B — review a plan / doc (non-diff)

```bash
codex exec --skip-git-repo-check "Critically review the plan at <path>. Identify correctness gaps, \
unstated assumptions, risky/out-of-order steps, missing edge cases, and simpler/safer alternatives. \
Tag each issue with a severity (P1/P2/P3). Do not rubber-stamp." </dev/null
```

- `codex exec` requires a git repo by default → add `--skip-git-repo-check` for non-git dirs.
- It reads stdin → `</dev/null` prevents hanging.
- When reviewing plans, codex may **search the web and cite sources** (deeper, but more tokens).

## Parsing codex output

codex stdout is mostly noise; the real review is **after the final `codex` marker**. Findings use `[P1]/[P2]/[P3]` severity + `file:line`, and are **printed twice** (dedupe). Ignore the metadata block, the `exec` process trace, and macOS git warnings like `couldn't create cache file '/tmp/xcrun_db-...'`.

## Core principle: no blind trust

Codex is a second opinion, not a verdict. Claude evaluates **each** finding: adopt the valid ones (and fix), push back on the wrong ones (with reasoning), and surface trade-offs for you to decide. It never silently applies or silently ignores everything.

## License

[MIT](LICENSE)
