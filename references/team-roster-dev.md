# 花名册 · 开发项目（`roster-dev`）

> 声明式配置数据，**不是 if 分支**：由 `references/project-profile.md` 的 `projectType: code` 决定加载本文件。
> 加预设只加文件，不动编排逻辑。规则正文见 `references/team-mode.md`。
> 内部名必须 `lower-kebab-case`（`spawn_teammate` 的 `name` 工具限制）；中文名用于任务卡、晨间简报、Web GUI。

## 1. 角色表

| 中文显示名 | 内部 kebab 名 | 是否 spawn | 并发名额 | 职责 | 写权限 |
|---|---|---|---|---|---|
| 领导 | `lead` | ❌ **不 spawn** | — | **Lead = 主会话**，常驻：轨道自检、拆票建卡、下发 `context-pack`、派工、收口 merge、写简报 | 可写任务板与最终合并，不写 doer 的交付文件 |
| 新增 | `feature` | ✅ | 1 | 新功能 / 新模块实现，自验 + 落 evidence | 仅自己的 `write_scopes` / worktree 分支 |
| 修复 | `bugfix` | ✅ | 1 | 修 bug、重构、性能修复，自验 + 落 evidence | 同上 |
| 界面 | `ui` | ✅ | 1 | UI / 页面 / 交互改动 + 无头 Playwright 截图证据 | 同上 |
| 审阅 | `review` | ✅ | 1 | **持久只读队友**：读分支 diff / 内容清单，出 `RESULT: PASS/FAIL`，累积 `review_notes` | **只读共享 cwd，不改代码**；只写任务卡与结论 |

- **单会话队友并发 = 4**（`feature` / `bugfix` / `ui` / `review`），Lead 不计入。
- 全持久队友（含「审阅」）：一次性 reviewer 每轮要重扫项目结构；持久「审阅」能记住上轮问题、验证是否真修。
- **模型**：`SpawnTeammateRequest` 没有 `model` 字段，**所有角色一律继承会话模型**，不可单独配。

## 2. 隔离与写范围

| `project-profile.isolation` | 做法 |
|---|---|
| `worktree`（编码 + git） | doer 各自 `create_worktree`，分支 `feature/<任务 id>` / `fix/<任务 id>` / `ui/<任务 id>`；自己分支提交，PASS 后由 Lead 在 `main` 顺序 merge |
| `write-scopes`（其余） | 按输出路径前缀分配 `team_task_create` 的 `write_scopes`；⚠️ **advisory 非锁**，靠队友遵守 + 任务卡把输出路径划清 |

agent-team **无文件系统锁、队友共享 cwd**：范围重叠会让返工回路出现「A 改完 B 又改回去」的震荡。这是第二道防死循环措施（第一道是 `round >= 3` 熔断 / 空转检测）。

## 3. spawn 与通信约定

1. Lead 先 `team_task_create` 拆票建卡（含 `owner` / `blocked_by` / `write_scopes` / `round` / `review_notes`）。
2. 按上表顺序 `spawn_teammate(name=<内部名>, context=fresh)`，拿到返回的 target。
3. Lead 用 `send_message` 给每个 doer 下发 `context-pack` + 任务卡 id + 输出路径。
4. doer 用 `team_task_get` 取最新 `revision` → `update(action=claim)` → 干活 → `update(action=complete)`（CAS，先读后写）。
5. 卡 `status → review` → `send_message` 给 `review` → 它回 `RESULT: PASS/FAIL`。
6. FAIL → 见 `team-mode.md` 返工闭环：打回**同一 doer**，再送回**同一个 `review`**。
7. `list_agents` 查可用性；`inactive` 只表示"没有 turn 在跑"，**不代表任务完成**。

## 4. 扩展

并行度高时可在 roster 里加 `审阅-前端`（`review-frontend`）/ `审阅-后端`（`review-backend`）条目，仍由 `project-profile` 数据驱动，编排逻辑不变。加角色后同步更新本表「并发名额」与 `team-mode.md` 的并发上限。

## 5. 与 A 仓的对照

| 项 | A 仓 `night-shift` | 本仓 `roster-dev` |
|---|---|---|
| 派工 | `subagent` / `subagent_fork`，一次性 | `spawn_teammate` 全持久 |
| 审阅 | 子代理自验 + 验证证据 | 持久只读 `review`，`RESULT: PASS/FAIL` |
| 返工 | 重新派新 subagent | `send_message` 打回同一 doer + 同一审阅 |
| 模型 | `tool-subagent.agentOptions` 可配便宜模型 | 全部继承会话模型 |
| 看板 | 任务文件 / 清单 | `team_task_*` 共享看板（GUI 可见） |
