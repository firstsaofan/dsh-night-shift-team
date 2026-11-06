# 项目档案 `project-profile.md`（声明式）

> 一份**数据文件**，由 Lead 在启动阶段读取，决定加载哪份 `team-roster-*.md` 以及用哪种隔离 / 审阅 / 交付方式。
> **不是 if 分支**：加组合只加配置，不改 `references/team-mode.md` 的编排逻辑。
> 建议放在项目工作区根（可由人在睡前预检单里确认），或由 Lead 按目录特征自动判定后写入 `context-pack`。

## 1. 字段定义

```yaml
projectType: code | content        # 编码 / 内容产出
vcs:         git | none            # 是否 git 仓库
isolation:   worktree | write-scopes
reviewTarget: diff | content-checklist
delivery:    merge | collect        # 合并 / 交付目录汇总
evidence:    test-output | validation-checklist
```

| 字段 | 取值 | 决定什么 |
|---|---|---|
| `projectType` | `code` / `content` | 加载 `team-roster-dev.md` 还是 `team-roster-content.md`；决定队友数（4 / 2） |
| `vcs` | `git` / `none` | 是否可用 `create_worktree`、是否走 `git merge` |
| `isolation` | `worktree` / `write-scopes` | doer 之间怎么划清写范围 |
| `reviewTarget` | `diff` / `content-checklist` | 持久「审阅」读分支 diff 还是 L0–L4 清单 |
| `delivery` | `merge` / `collect` | PASS 后由 Lead 顺序 merge 到 `main`，还是交付目录汇总 |
| `evidence` | `test-output` / `validation-checklist` | 证据用测试 / 构建输出，还是校验清单 + 预览 |

## 2. 三种定稿组合

| 组合 | `projectType` / `vcs` | 隔离 | 审阅对象 | 收尾 | 证据 |
|---|---|---|---|---|---|
| 编码 + git | `code` / `git` | `worktree` 分支（`feature/`、`fix/`、`ui/` + 任务 id） | 分支 `diff` | `git merge` 到 `main` | 测试 / 构建输出 |
| 非编码 + git | `content` / `git` | `write_scopes` | 内容清单 / 预览 | `git merge` 或交付汇总 | 校验清单 |
| 非编码 + 非 git | `content` / `none` | `write_scopes` | 内容清单 / 预览 | **交付目录汇总，无 merge** | 校验清单 |

```yaml
# 组合 A：编码 + git
projectType: code
vcs: git
isolation: worktree
reviewTarget: diff
delivery: merge
evidence: test-output
```

```yaml
# 组合 B：非编码 + git
projectType: content
vcs: git
isolation: write-scopes
reviewTarget: content-checklist
delivery: collect
evidence: validation-checklist
```

```yaml
# 组合 C：非编码 + 非 git
projectType: content
vcs: none
isolation: write-scopes
reviewTarget: content-checklist
delivery: collect
evidence: validation-checklist
```

## 3. 自动判定（拿不准就问，或落回 code）

| 判据 | 结论 |
|---|---|
| 有 `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / `*.csproj` / `*.sln` / `src` / `tests` | `projectType: code` |
| 以 `*.md` / 图片 / 视频 / 设计稿为主，无构建清单 | `projectType: content` |
| 工作区根有 `.git` | `vcs: git`，否则 `none` |
| 两者特征都有 | 先按 `code`，在预检单里请人确认 |

> 与 `references/project-init.md` 的「项目类型判定」保持同一口径；不一致时以本文件为准，并把差异记入 `DECISIONS.md`。

## 4. 约束（别踩）

- `create_worktree` **只在 `isolation: worktree` 且 `vcs: git` 时可用**；非 git 目录用不了。它由**独立的「git 工作树」实验性插件**（`dsh-experimental-tool-worktree`）提供，**不是 Agent Teams 自带的**——agent-team 本身无 worktree 隔离、无文件系统锁。因此 `isolation: worktree` 需要该插件也已启用；`isolation: write-scopes` / `vcs: none` / `delivery: collect` 不需要它。插件启用路径：DSH → 插件 → 实验性插件 → 「git 工作树」。
- `write_scopes` 是 `team_task_*` 自带的「预期写范围前缀」，**advisory 非锁**：靠队友遵守 + 任务卡把输出路径划清；范围重叠会让返工回路出现「A 改完 B 又改回去」的震荡（第二道防死循环措施，第一道是 `round >= 3` 熔断 / 空转检测）。
- 二进制文件（docx / pptx / mp4 / png）即便在 git 里也无法 diff / 自动合并 → 冲突一律 `blocked` 交白天。
- 合并目标固定为本地 **`main`**，顺序、仅 `done + PASS` 后由 Lead 执行；不 rebase / amend / force。
- `git push` / 开 PR / 清理已合并 worktree 分支 / 非编码产物最终发布 / 依赖变更复核 / 所有 `blocked` 卡裁定 → **留白天**（永久人工闸门）。
