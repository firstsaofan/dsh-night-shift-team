# 上下文包 `context-pack.md`

> **解决什么问题**：持久队友解决的是**会话内**记忆（同一晚不用每轮重扫项目结构）；`DECISIONS.md` 解决的是**跨夜**决策留痕；本文件解决的是**跨会话装载**——新会话 / 新队友第一次进来时，用一份外置上下文包把项目结构、约定、累积意见、历史 diff 索引一次性装载，不必重放整段对话。
>
> **装载时机**：新会话装载**一次**；会话内沿用队友自己的记忆，不反复重放。Lead 在 spawn 之后、派第一张卡之前用 `send_message` 下发。

## 1. 定位

| 层 | 载体 | 覆盖范围 |
|---|---|---|
| 会话内记忆 | 持久队友（`spawn_teammate`） | 一晚之内，队友记得上轮问题 |
| 跨会话上下文 | **`context-pack`**（本文件） | 新会话 / 新队友第一次装载 |
| 跨夜决策留痕 | `DECISIONS.md` / `ESCALATION.md` | 按日期 append-only，永不改写 |
| 验证证据 | `evidence/<日期>/` | 原始层，按日期留档 |

`context-pack` 是**索引 + 摘要**，不是全文复制：只放"指针 + 一句话结论"，详细内容留在原始文件里按需读取，避免撑爆上下文。

## 2. 建议结构

```markdown
# context-pack · <项目名> · <日期>

## A. 项目结构摘要
- 类型：<code|content>（见 references/project-profile.md）
- vcs：<git|none>；isolation：<worktree|write-scopes>；delivery：<merge|collect>
- 顶层目录与职责：<一行一个>
- 入口 / 构建 / 验收命令：<指向 package.json scripts 等，不贴全文>
- 关键约定：<命名、目录、分支前缀、不可改区域>

## B. 架构约定
- 本夜起的分支命名：feature/<id> / fix/<id> / ui/<id>
- 合并目标：main（本地、顺序、仅 done+PASS）
- 输出路径前缀：deliver/<任务 id>/、evidence/<日期>/<任务 id>/
- 红线：不 push / 不 amend / 不 rebase / 不 force / 不装包（除预检预授权）
- 依赖状况：<未新增任何依赖 / 列出包名@版本 + 出处>

## C. 累积 review_notes（按任务 id）
- <task id> round 1: <意见一句话>（原文见任务卡 review_notes）
- <task id> round 2: <意见一句话>
> 只放索引与摘要；**累积意见的正文留在任务卡**，不在这里复制一遍。

## D. 各轮 diff 与 evidence 索引
| 轮次 | 任务 id | 分支/交付路径 | diff 摘要 | evidence 路径 | RESULT |
|---|---|---|---|---|---|
| 1 | ... | ... | ... | night-shift/evidence/<日期>/... | FAIL |
| 2 | ... | ... | ... | ... | PASS |

## E. 状态与待办
- 任务板当前：pending N / in_progress N / review N / done N / blocked N
- 会话级熔断计数：blocked 累计 <N>/3
- 待白天裁定：<blocked 卡一句话>
- 预检单答复摘要：<roster 预设 / 专属规格 / 时长预算 / 依赖预授权>
```

## 3. 装载纪律

1. **一次**：新会话 / 新队友第一次装载；之后靠队友记忆 + 任务板，不每轮重发。
2. **指针优先**：只给路径与一句话结论，不贴全文；详细内容让队友按需 `read`。
3. **不写敏感原文**：`context-pack` 属运行时产物，**脱敏黑名单**（`DESENSITIZE.md`），绝不进发布仓库；里面若有本机绝对路径，只在本地使用。
4. **append-only 索引**：D 区表格每轮**追加一行**，不改写旧行；与 `DECISIONS.md` / `evidence/` 的原始层纪律一致。
5. **冲突以任务板为准**：`context-pack` 是缓存摘要，状态不一致时以 `team_task_get` 的当前 `revision` 为准，并回写修正摘要。

## 4. 与 A 仓的对照

| 项 | A 仓 `night-shift` | B 仓 `night-shift-team` |
|---|---|---|
| 跨会话上下文 | `DECISIONS.md` / 决策日志 | `context-pack`（新会话装载一次）+ 队友会话内记忆 |
| 队友记忆 | 无（一次性 subagent） | 持久队友自带，审阅能验证上轮问题是否真修 |
| 任务状态 | 任务文件 / 清单 | `team_task_*` 共享看板（GUI 可见） |
