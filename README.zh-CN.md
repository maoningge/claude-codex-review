<!-- Language switch -->
[English](README.md) | **中文**

# claude-codex-review

一个 [Claude Code](https://claude.com/claude-code) skill：把 **Claude 刚产出的**代码或计划，交给 **Codex**（`codex` CLI）做独立的第二意见审查。

**为什么：** 出题人和审卷人不该是同一个模型。Claude 容易对自己的实现有盲点，Codex 走不同的模型和训练路径，常能抓到 Claude 自查时看不到的问题。

## 依赖

- [Claude Code](https://claude.com/claude-code)
- [`codex` CLI](https://github.com/openai/codex)，已登录（`codex login`）。基于 `codex-cli 0.135.0` 实测。

## 安装

在 Claude Code 里，把本仓库添加为 plugin marketplace，再安装 plugin：

```
/plugin marketplace add maoningge/claude-codex-review
/plugin install claude-codex-review@claude-codex-review
```

重启 Claude Code（或运行 `/reload-plugins`）让 skill 生效。

## 用法

当 Claude 刚写完/改完代码、或刚起草了计划、且交叉审有价值时触发；或你显式要求：*"codex 审一下"*、*"过一遍"*、*"cross-check"*、*"第二意见"*……

Claude 会**先主动问**（审查消耗 codex 额度），你点头才跑，然后逐条批判性评估结果。

### 路径 A —— 审代码（git diff）

首选，带聚焦指令（纯 prompt 模式）：

```bash
codex review "Review the current uncommitted changes only. <聚焦指令>"
```

flag 模式（不带聚焦）：

```bash
codex review --uncommitted        # staged + unstaged + untracked
codex review --base <分支>         # 整条 feature 分支
codex review --commit <sha>       # 某次提交
```

> ⚠️ `--uncommitted` / `--base` / `--commit` 与 PROMPT **互斥**。要传聚焦指令就用上面的纯 prompt 形式，让 codex 自己定位改动。

### 路径 B —— 审 plan / 文档（非 diff）

```bash
codex exec --skip-git-repo-check "Critically review the plan at <路径>. Identify correctness gaps, \
unstated assumptions, risky/out-of-order steps, missing edge cases, and simpler/safer alternatives. \
Tag each issue with a severity (P1/P2/P3). Do not rubber-stamp." </dev/null
```

- `codex exec` 默认要求 git 仓库 → 非 git 目录加 `--skip-git-repo-check`。
- 它会读 stdin → `</dev/null` 防挂起。
- 审 plan 时 codex 可能**联网搜索并给文献引用**（更深，但更费 token）。

## 解析 codex 输出

codex stdout 大部分是噪音；真正的审查在**最后一个 `codex` 标记之后**。Findings 用 `[P1]/[P2]/[P3]` 标级 + `文件:行号`，且**重复打印两遍**（需去重）。忽略元信息块、`exec` 过程、以及 macOS git 告警如 `couldn't create cache file '/tmp/xcrun_db-...'`。

## 核心原则：不盲从

Codex 是第二意见，不是最终裁决。Claude 逐条评估：成立的采纳并改，不成立的反驳并给理由，取舍点摆给你定。绝不默默照单全改，也绝不默默全忽略。

## 许可

[MIT](LICENSE)
