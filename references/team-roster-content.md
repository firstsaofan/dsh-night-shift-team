# 花名册 · 非开发项目（`roster-content`）

> 声明式配置数据：由 `references/project-profile.md` 的 `projectType: content` 决定加载本文件。
> 适用：绘图、剪辑、文档、小说、设计稿、PPT/视频等**非编码产出**。规则正文见 `references/team-mode.md`。
> 内部名必须 `lower-kebab-case`；中文名用于任务卡、晨间简报、Web GUI。

## 1. 角色表

| 中文显示名 | 内部 kebab 名 | 是否 spawn | 并发名额 | 职责 | 写权限 |
|---|---|---|---|---|---|
| 领导 | `lead` | ❌ **不 spawn** | — | **Lead = 主会话**，常驻：轨道自检、拆票建卡、下发 `context-pack`、派工、交付目录汇总、写简报 | 可写任务板；不写 doer 的交付文件 |
| 通用 | `general` | ✅ | 1 | 按任务卡产出内容（图 / 视频 / 文档 / 页面），自验 L0–L2 + 落 L4 证据 | 仅自己的 `write_scopes`（输出路径前缀） |
| 审阅 | `review` | ✅ | 1 | **持久只读队友**：按 `validation-checklist.md` 的 L3 内容合规清单核对 + 抽查 L4 证据，出 `RESULT: PASS/FAIL` | **只读共享 cwd，不改产物**；只写任务卡与结论 |

- **单会话队友并发 = 2**（`general` / `review`），Lead 不计入。
- 全持久队友（含「审阅」）：持久「审阅」能记住上轮问题、验证是否真修。
- **模型**：`SpawnTeammateRequest` 没有 `model` 字段，**所有角色一律继承会话模型**。

## 2. 隔离与写范围

- 非开发项目通常**没有 git worktree**，统一用 `team_task_create` 的 `write_scopes` 按**交付目录 / 输出路径前缀**分配（例：`deliver/<任务 id>/`、`evidence/<日期>/<任务 id>/`）。
- ⚠️ `write_scopes` 是 **advisory 非锁**：agent-team 无 fs 锁、队友共享 cwd，靠队友遵守 + 任务卡把输出路径划清。
- 二进制产物（docx / pptx / mp4 / png）**无法 diff / 自动合并** → 冲突一律 `blocked` 交白天，不自行裁决。
- `delivery: collect`：**交付目录汇总，无 merge**；每个 doer 只写自己的交付目录，Lead 汇总清单。

## 3. 验收与审阅

按 `references/validation-checklist.md` 的 **L0–L4**：

| 层 | 执行者 |
|---|---|
| L0 存在性 / L1 完整性 / L2 规格 / L4 证据 | 脚本自动 + doer 自验（**不进预检单**） |
| L3 内容合规（无 `TODO` / `lorem` / 空白帧 / 与任务卡描述一致） | 持久「审阅」按清单核对 |
| 抽查 L4 预览图 / 关键帧 / 首屏截图 | 持久「审阅」 |

**审阅队友不做审美评审**。仅 **L2 的项目专属数值**（16:9、3 分钟、5 张图、主题、命名）需人睡前在预检单里定。

## 4. spawn 与通信约定

1. Lead `team_task_create` 拆票建卡（含 `owner` / `blocked_by` / `write_scopes` / `round` / `review_notes` / 验收规格）。
2. `spawn_teammate(name=general, context=fresh)`、`spawn_teammate(name=review, context=fresh)`，拿到返回的 target。
3. Lead `send_message` 下发 `context-pack` + 任务卡 id + 交付路径 + 预检单里的专属规格。
4. doer `team_task_get` → `claim` → 产出 + 自验 → `complete` → 卡 `status → review`。
5. `review` 出 `RESULT: PASS/FAIL`；FAIL 走 `team-mode.md` 返工闭环（打回**同一 doer** + 送回**同一审阅**）。
6. `list_agents` 查可用性；`inactive` 只表示"没有 turn 在跑"，**不代表任务完成**。

## 5. 与开发预设的差异

| 项 | `roster-dev` | `roster-content` |
|---|---|---|
| 队友数 | 4（新增/修复/界面/审阅） | 2（通用/审阅） |
| 隔离 | `worktree` 分支 或 `write_scopes` | 仅 `write_scopes` |
| 审阅对象 | 分支 diff | 内容清单 L0–L4 |
| 收尾 | Lead 在 `main` 顺序 `git merge` | **交付目录汇总，无 merge** |
| 证据 | 测试 / 构建输出 | 校验清单 + 预览图 / 关键帧 |
