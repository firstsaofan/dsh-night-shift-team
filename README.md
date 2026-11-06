简体中文 | [English](README.en.md)

# night-shift-team — 无人值守流水线（Agent Teams 版 / B 仓）

> **名字叫 night-shift-team，但适用条件是"人不在设备前"，不是"晚上"**：出门、开会、通勤、吃饭、睡觉都算无人值守时段。判断标准只有一条——**你不在场时它也能按规则自主推进**。

> 🚨 **硬前提：本 skill 必须先启用 DSH 的两个实验性插件 ——「智能体团队」+「git 工作树」，否则完整功能跑不起来。** 见下节。

---

## 🔌 硬前提：先启用两个实验性插件

> **一句话**：`night-shift-team` 的派工能力来自 `spawn_teammate` / `send_message` / `team_task_*`（**智能体团队插件**），编码项目的分支隔离来自 `create_worktree`（**git 工作树插件**）。这些工具**只在对应插件加载后才存在**。插件没启用 → 工具不存在 → **启动自检直接判 `FAIL`、门禁不放行**（不弹卡片、不生成 `GATE.md`、不挂 Goal）。**没有降级、没有 fallback、不会静默继续**。

**两个插件的分工（别混）**

| 插件 | 提供的工具 | 什么时候必需 |
|---|---|---|
| 智能体团队 `dsh-experimental-agent-team-profile` | `spawn_teammate` / `list_agents` / `send_message` / `interrupt_agent` / `team_task_*` | **所有场景都必需**（本仓派工层 100% 依赖） |
| git 工作树 `dsh-experimental-tool-worktree` | `create_worktree` | **仅 `isolation: worktree`（编码 + git 项目）**；`write-scopes` / 非 git / `delivery: collect` 不需要 |

> ℹ️ **worktree 确实依赖「git 工作树」插件**：agent-team 本身**不带 worktree 隔离、也没有文件系统锁**（队友共享 cwd）；`create_worktree` 是另一个插件提供的独立工具，只创建分支 + 切换调用会话 cwd，不合并、不删分支、不开 PR。所以只想跑非编码 / `write-scopes` 的项目，只开「智能体团队」就能用；但要用 B 仓的**完整**能力（编码项目分支隔离 + merge 交付），两个都得开。

### 没启用会怎样

| 现象 | 原因 |
|---|---|
| 工具清单里找不到 `spawn_teammate` / `team_task_create` / `list_agents` | 智能体团队插件未加载，这些工具根本不存在 |
| 拆票建卡、spawn 持久队友、共享看板、返工闭环全部无法执行 | 本仓派工层 100% 依赖这些工具 |
| 启动自检 `RESULT: FAIL`，门禁不放行 | `SKILL.md` §2A.1「每对话必做插件轨道自检」硬阻断 |
| 编码项目里 `create_worktree` 不存在，无法建分支隔离 | git 工作树插件未启用（此时应改用 `isolation: write-scopes`，或启用该插件） |
| 同一实例同时跑 A、B 两个夜班 skill 会互相干扰 | 两者争用同一套状态目录 / 心跳 / 运行登记表 / Goal 与看门狗单进程假设，且编排互斥；**不是因为 subagent 被禁**（见下节） |

### 怎么确认已经启用（1 分钟）

新开一个会话，看**工具清单**是否同时满足：

- ✅ 存在 `spawn_teammate` / `list_agents` / `send_message` / `interrupt_agent` / `team_task_create` / `team_task_get` / `team_task_list` / `team_task_update`
- ℹ️ **不要检查** `subagent` / `subagent_fork` 是否消失：2026-11-06 实测它们仍列在工具清单、系统提示未标注不可调用，旧判据会恒判 FAIL。本仓不使用它们是编排选择，不是环境检查项。
- ✅ **（仅编码 + git 项目需要）** 存在 `create_worktree`

**只要第一项满足就是 B 轨就绪**；第二项是说明、不作判据；第三项只在你要用 `isolation: worktree` 时检查。可直接复制本文「首次使用必做：可用性实测」里的 **实测 A：插件轨道自检**，让 agent 报 `RESULT: PASS/FAIL`。

### 怎么启用

**界面路径（推荐）**：DSH → **插件** → **实验性插件** → 把下面两个都打开：

1. **启动智能体团队插件**
2. **git 工作树**

然后**重启 DSH**（实例级配置，热更新不生效），再新开一个会话跑上面的自检确认 `PASS`。

> 等价的配置层写法：改 `profiles/<profile>/package.json` 的 `dsh.profile.bundles`，加入内置的 `@deepseek-ai/dsh-experimental-agent-team-profile` 与 `@deepseek-ai/dsh-experimental-tool-worktree`。两个插件本来就放在同一个「实验性插件」分组里，一起启用最省事。

> ⚠️ **粒度是实例级、不是会话级**：新对话只能选 preset，**选不了 profile**。同一 DSH 实例上的所有会话共享这一轨，不要指望「这个对话用团队、那个对话用子代理」。

### 这是本机配置，不是本仓库的内容

本仓库**只是纯 markdown 文本文件**：不 fork 插件、不改 `app.asar`、不改 `config.json`、不改注册表、不装任何包。启用插件是你在**本机 DSH 侧**的一次性配置，与本 skill 的安装（`Copy-Item` 到 `~\.dsh\skills\`）是**两件事，两件都要做**。

> 装了 skill 但没启用插件 = 装了一本用不了的手册；启用了插件但没装 skill = 有工具、没有编排规则。

---

## 使用前清单（Checklist）

> 按顺序核对再挂夜班。标 ⚠️ 的几条是**不在 skill 本体里**、最容易漏的本机 / 项目前置条件。

| # | 前置条件 | 怎么确认 / 怎么做 | 缺了会怎样 |
|---|---|---|---|
| 1 | 两个实验性插件已启用（智能体团队 + git 工作树） | 见上节「怎么启用」；新开会话跑「怎么确认」的自检 | 门禁不放行，整个 skill 跑不起来 |
| 2 | 本 skill 已安装 | `Copy-Item -Recurse night-shift-team ~\.dsh\skills\ -Force`；catalog 里能看到 `night-shift-team` | 有工具、没有编排规则 |
| 3 | ⚠️ **工作区是独立、可写的项目目录** | 见下方「工作区要求」 | 状态文件写错位置，污染 A 仓运行时记录 |
| 4 | ⚠️ **`merge` / `worktree` 要求 git 仓库 + 本地 `main` 分支已存在** | `git -C <工作区> branch --list main` | `create_worktree` / `git merge` 失败 → `blocked` 交白天 |
| 5 | ⚠️ **本机全局 `AGENTS.md` 已配置两条夜班豁免** | 见下方「豁免条款」 | 夜班 `git commit` 被逐轮确认卡住；功能类装包无法自主执行 |
| 6 | 主机级基础设施已装（看门狗 / 心跳 / 调度 / 运行登记表 / `config.json`） | Windows `pwsh -NoProfile -File scripts/windows/setup.ps1`；macOS `bash scripts/macos/setup.sh`；配置在 `$DSH_HOME/night-shift/config.json` | 崩溃后不会自动续跑；项目初始化会在预检单提示 |
| 7 | 审批 `never` + 默认 5 次重试 | DSH 配置层；首夜按「首夜前置检查」三项验证 | 夜里卡在等人确认 / 网络抖动直接断 |
| 8 | 桌面 GUI（Web 宿主）+ PowerShell 7.6 | `schedule` 心跳只在 Web 宿主可用；脚本用 `pwsh` 7.6（MSI 版） | headless 无心跳；5.1 读 UTF-8 乱码、脚本报错 |
| 9 | 任务清单已落文件 + 睡前答完预检单 + 你离开设备 | 清单写 `BUG-BOARD.md` 或你指定的文件；预检单在 `night-shift/preflights/<日期>.md`；门禁四件套见 `SKILL.md` §0 | 任务从没开始 / 只跑一轮就停 |

### 工作区要求（⚠️ 最容易踩）

本 skill 运行时会在**工作区根**写入：

```
<工作区>/night-shift/     状态目录（preflights/ briefs/ summaries/ evidence/ DECISIONS.md …）
<工作区>/AGENTS.md         追加知识沉淀指针块（幂等，不改已有内容）
<工作区>/knowledge/        知识层骨架（首次进入时）
```

- **必须选独立的项目目录当 cwd，且该目录可写**；
- **不要**把「包含 A 仓 `night-shift/` 运行时状态目录的上级目录」当工作区（例如同时装着 A 仓 skill 集合的父目录）—— B 仓会把自己的状态文件写进 A 仓的真实记录里，造成污染；
- 两个 skill 的状态目录同名（`night-shift/`），这也是「同一实例同一时刻只跑一轨」的原因之一；
- `delivery: collect` 的非 git / 内容项目没有 git 要求，但仍需要独立可写目录。

### 豁免条款（下游用户需自己配置）

文档里写的「commit 豁免」「功能类装包豁免」是**本机全局 `AGENTS.md`**（`~/.dsh/AGENTS.md`）的条款，**不是本 skill 自带能力**。你要在自己的全局规范里加：

- **commit 豁免**：夜班 / 无人值守模式下一任务一提交（`git add` + `git commit`），无需逐轮确认；**push / amend / rebase 仍永久禁止**；
- **功能类依赖豁免**：仅限声明为无人值守的**新增功能类**任务，同时满足「有出处 / 精确锁版本 / 装包卫生 / 单独 `chore(deps)` commit 可回滚」四条；**修复类任务装包永远是红线**。

缺这两条时 skill 不会失败，但会在夜里停在等人确认。

### git 项目前提（仅 `merge` / `worktree`）

| 组合 | 前提 |
|---|---|
| `isolation: worktree` | 项目是 git 仓库；`create_worktree` 只建分支 + 切 cwd，不合并 / 不删分支 / 不开 PR |
| `delivery: merge` | 本地 `main` 分支存在，Lead 在其上顺序 `git merge`；仅 `done + PASS` 后合并；冲突 / 二进制一律 `blocked` 交白天 |
| 其余（`write-scopes` / 非 git / `collect`） | 不需要 git，也不需要 `main` 分支 |

> `git push` / 开 PR / 清理已合并 worktree 分支 / 非编码产物最终发布 → **永远留白天人工做**。

---

## ⚠️ 二选一不可混用（先读这一段）

> 同一台 DSH 实例、同一时间只能使用 **A 仓 `night-shift`** 或 **B 仓 `night-shift-team`** 其中之一，不可同时启用，也不可同一晚在不同对话分别跑 A 与 B。**理由不是「工具互斥」**：2026-11-06 实测两套派工工具可在同一实例共存；真正的冲突是两个 skill 争用同一套运行时基础设施（项目内 `night-shift/` 状态目录、`DECISIONS.md`、心跳、主机级 `runs/` 运行登记表、Goal、看门狗单进程假设），且编排方式互斥（一次性 subagent vs 持久队友 + 共享任务板）。Agent Teams 插件由 **profile 的 `dsh.profile.bundles`** 决定，是**实例级全局**开关；**新对话只能选 preset，选不了 profile**。两个仓库可以同时装在磁盘上，但同一实例同一时刻只有一轨生效。
>
> **切换轨道** = 改 `profiles/<profile>/package.json` 的 `dsh.profile.bundles` **并重启 DSH**，属实例级操作。**不建议靠「开两个实例、各用不同 profile」混跑**：night-shift 看门狗按单一 `dshProcessName` 检测进程、假设「DSH 只有一个进程」（`references/host-level.md`），多实例会让进程判定与崩溃恢复失效。
>
> 不是「永久二选一」：白天用 A 仓、晚上切到 B 仓可以，只要中间改 bundle 并重启。

## 与 A 仓 `night-shift` 的区别（该选哪个）

> 🔗 **姐妹仓库**：个人 / 小项目、想要「worker 便宜模型 / 主循环贵模型」的模型路由 → 用 [**night-shift**（原仓）](https://github.com/firstsaofan/dsh-night-shift)。本仓用 Agent Teams 全持久队友 + 共享任务板 + 持久审阅 + 返工闭环，面向中大型项目；**同一 DSH 实例不可混用**（详见下节）。

> 两个仓库是**同一套无人值守基础设施的两种派工实现**：A 仓用 `subagent` 一次性子代理，B 仓用 DSH 内置 Agent Teams 全持久队友。**A 仓一行未改**；除派工层外，门禁 / 心跳 / 看门狗 / 预检单 / 决策日志 / 简报 / 证据制 / 熔断全部一样。

### 规格对照

| 维度 | A 仓 `night-shift`（原仓） | B 仓 `night-shift-team`（本仓） |
|---|---|---|
| 定位 | 个人 / 小项目 / 全栈单人 | 中大型项目、多任务并发、需要 review / 返工 / 合并 |
| 插件前提 | agent-team **未启用**（`subagent` 可用） | agent-team **已启用**（内置，不 fork） |
| 派工 | `subagent` / `subagent_fork`，一次性 | `spawn_teammate` + `send_message` + `team_task_*`，全持久 |
| 队友 | 无 | 开发 4（新增/修复/界面/审阅）、非开发 2（通用/审阅）；Lead = 主会话，不 spawn |
| 模型 | ✅ `tool-subagent.agentOptions` 可配（worker 便宜 / 主循环贵） | ❌ 全角色继承会话，不可单独配 |
| 看板 | 任务文件 / 清单 | `team_task_*` 共享看板（Web GUI 可见） |
| 隔离 | 无 | 编码 + git：`worktree` 分支；其余：`write_scopes`（**advisory 非锁**） |
| 审阅 | 子代理自验 + 验证证据 | 持久只读「审阅」队友，出 `RESULT: PASS/FAIL` |
| 返工 | 重新派新 subagent | `send_message` 打回**同一 doer**、送回**同一审阅**；`review_notes` 累积、`round>=3` 熔断、空转检测 |
| 交付 | 任务总结 / 晨间简报 | git：仅 PASS 后 Lead 在 `main` 顺序 merge；非 git / 内容项目：交付目录汇总 |
| 跨会话上下文 | `DECISIONS.md` / 决策日志 | `context-pack`（新会话装载一次）+ 队友会话内记忆 |
| 项目类型 | 任意（已实战验证） | 编码 / 非编码均可，`project-profile` 声明式分流 |
| 基础设施 | 预检单、门禁卡片、看门狗、心跳、调度、运行登记表、习惯决策树、验证证据、决策日志、晨间简报、升级标记、commit 豁免、熔断 | **全部原样保留** |
| push / PR | 白天人工 | 白天人工（永久闸门） |

### 什么场景更适合用哪个

**选 A 仓 `night-shift`（原仓）**
- 一个人的小项目 / 全栈单人，一晚 1–3 个基本独立的任务；
- **想省钱**：需要「worker 便宜模型 / 主循环贵模型」的模型路由（`tool-subagent.agentOptions`）；
- 不需要 code review、不需要返工闭环，任务清单文件 + 决策日志 + 晨间简报就够；
- 不想启用 Agent Teams 插件，只想用 `subagent`；
- 已经在用 A 仓且跑得稳 —— **没必要迁移**，A 仓仍在维护、一行未改。

**选 B 仓 `night-shift-team`（本仓）**
- 中大型项目、一晚多任务并行，任务之间有依赖、需要共享看板；
- 需要**持久「审阅」队友**记住上轮问题、验证返工是否真修（A 仓每轮派新 subagent 会丢上下文）；
- 需要**返工闭环**：FAIL → 打回同一 doer → 同一审阅 → PASS，而不是「重派一个新子代理」；
- 需要**合并交付**：仅 PASS 后由 Lead 在 `main` 顺序 merge；
- 一晚混合编码 / 绘图 / 剪辑 / 文档等非编码产出，`project-profile` 声明式分流；
- 项目上下文重，持久队友复用比每轮重扫项目结构更划算。

**两个都不适合**
- 人在场、任务随时要问人确认 → 两个 skill 都按「该问就问」，不启用夜班默认处置；
- 只做一次性的单任务 → 直接做，不用挂夜班；
- 想在同一实例同时跑两个仓库 → **不可能**，见顶部「二选一不可混用」。

### 切换成本（先算清楚再选）

| 项 | 说明 |
|---|---|
| 粒度 | **实例级**：Agent Teams 由 profile 的 `dsh.profile.bundles` 决定；新对话只能选 preset、选不了 profile |
| 切换动作 | 改 `profiles/<profile>/package.json` 的 `bundles` **并重启 DSH** |
| 能否共存 | 两个仓库可同时装在磁盘上、同时出现在 skill catalog，但**同一实例同一时刻只有一轨生效** |
| 不要混跑 | 别靠「开两个实例、各用不同 profile」同时跑 A 与 B：看门狗按单一 `dshProcessName` 检测进程，多实例会让崩溃恢复失效 |
| 代价 | A 仓没有 roster / 任务板 / 返工 / `context-pack`；B 仓没有模型路由，且队友并发占用同一会话上下文 |
| 推荐用法 | 白天用 A 仓（人可监视、模型路由省钱），晚上切到 B 仓（重任务 + review）；中间改 bundle 重启一次 |

> ⚠️ 触发词重叠：两个 skill 的 `description` 都以「夜班 / 挂机 / 无人值守」触发。同时安装时请在指令里**明确点名仓库名**（`night-shift` 或 `night-shift-team`），避免选错。

**前置条件（B 仓）**：① 该 DSH 实例的 profile 已启用内置 `@deepseek-ai/dsh-experimental-agent-team-profile`（**不 fork 插件、不改 `app.asar`、不改 DSH 运行时**）；② 装本 skill（`Copy-Item -Recurse night-shift-team ~\.dsh\skills\ -Force`）；③ 主机级看门狗 / 心跳 / 调度 / 运行登记表沿用 A 仓，**只检查不安装**。每个启动本 skill 的对话都会做**插件轨道自检**，不匹配则门禁不放行（不弹卡片、不生成 `GATE.md`、不挂 Goal），并给出切轨指引——**不降级、不静默继续**。

> 本仓完整保留 A 仓的：预检单、门禁卡片、看门狗、心跳、调度、运行登记表、习惯决策树、验证证据、决策日志、晨间简报、升级标记、commit 豁免、熔断。只替换了「派工层」并新增团队层。

> **想先搞懂结构？** 看 [`DESIGN.md`](DESIGN.md)：一页看懂 → 全景架构 → 一夜时序 → 单任务闭环 → 崩溃恢复 → 依赖决策 → 知识流向 → 文件地图 → 参数表 → 常见误解。
> **给 agent 的操作手册**：本目录 `SKILL.md` + `references/`（agent 会按需加载，人不必读完）。
> **本文 README 是给人看的**：怎么用、为什么这么设计、什么时候该加脚本/小系统。
> **不是 DSH 用户？** 看 [`PORTING.md`](PORTING.md)：这个 skill **绑定 DSH 的自动化层**（心跳/轮次/子代理/崩溃恢复），但规则层可移植；里面有逐项改造清单（Claude Code / Codex / 任意 CLI agent）。

让 DSH 在你睡觉/不在电脑前时自主完成任务列表：实现 → 验证（证据制）→ 按你的习惯决策 → 提交（不 push）→ 总结沉淀 → 早上给你一份 5 分钟能扫完的简报。

## 目录结构

```
night-shift-team/
├── SKILL.md                      # agent 主手册：模式判定/三原则/主流程/§2A 团队模式/git 规则
├── README.md                     # 本文件：给人的使用说明 + 设计理由 + 升级路线
├── README.en.md                  # 英文手册
├── DESIGN.md                     # 设计说明：架构图/时序图/流程图/文件地图/参数表
├── PORTING.md                    # 移植指南：换 agent 需要改什么（心跳/轮次/团队层/看门狗替代方案）
├── INTERNAL-dsh-mechanisms.md    # DSH 内置机制详解（内部文档）
├── LICENSE                       # MIT
├── DESENSITIZE.md                # 脱敏规则清单与报告格式（开源同步用）
└── references/
    ├── habits.md                 # 习惯决策树：自由区/禁区/拿不准的默认倾向【A 仓原样保留】
    ├── verification.md           # 验证协议：按任务类型的最低证据要求【原样保留】
    ├── templates.md              # 产出物模板：预检单/总结/决策日志/晨间简报【原样保留】
    ├── project-init.md           # 项目初始化 + B 仓插件轨道自检 / run 占用自检
    ├── host-level.md             # 主机级单例：运行登记表/看门狗/OS 安装脚本【原样保留】
    ├── team-roster-dev.md        # 【新增】开发花名册：新增/修复/界面/审阅
    ├── team-roster-content.md    # 【新增】非开发花名册：通用/审阅
    ├── project-profile.md        # 【新增】声明式字段与三种组合
    ├── team-mode.md              # 【新增】编排规则/任务卡/状态机/返工闭环/熔断/RESULT
    ├── context-pack.md           # 【新增】跨会话上下文包
    └── validation-checklist.md   # 【新增】非编码通用验收 L0–L4

scripts/                          # 【可选】主机级组件，不被 skill 加载
├── windows/{setup.ps1, watchdog-launch.vbs, watchdog.ps1}
├── macos/{setup.sh, watchdog.sh} # 仅设计对齐，不实测
├── README.md                     # 【原样保留】
└── sync-to-open-source.ps1       # 【新增】脱敏同步脚本（默认 dry-run，-Apply 才写）
```

**三层边界（别混）**：

| 层 | 装在哪 | 谁装 | 唯一性 |
|---|---|---|---|
| **主机级**（看门狗/调度/运行登记表/config） | `$DSH_HOME/night-shift/` | **一次性安装脚本**（按 OS） | 全局唯一，多项目共用 |
| **项目级**（AGENTS.md 注入规则 + knowledge 骨架 + night-shift 状态目录） | 各项目工作区 | skill 首次进项目 | 每项目一份 |
| **知识层**（原始/派生 + 集中通用层） | 项目内 + `<centralKnowledge>` | skill 运行中 | 项目内为主，通用层集中 |

**知识沉淀规则在项目级、不在全局**（2026-10-30 决定）：开工调研/任务后总结随项目走——首次夜班进入某项目时，skill 把适配该项目类型（代码/非代码）的规则**追加**注入该项目的 `AGENTS.md`（幂等标记，已有内容不动）。全局 AGENTS.md 只保留安全硬约束；小说这类非代码项目不会被"验收命令/lint 门禁"刷噪音。

**产物归属（三类别混）**：

| 类别 | 产物 | 装几次 | 含脚本吗 |
|---|---|---|---|
| **skill 本体** | `SKILL.md` + `references/*.md` | 一次（复制到 `~\.dsh\skills\`） | ❌ 纯 markdown |
| **项目初始化** | 项目 `AGENTS.md` 指针块 + `knowledge/` 骨架 + `night-shift/` 状态目录 | 每项目一次（skill 自动） | ❌ 零脚本 |
| **主机级（可选）** | `config.json` + 看门狗脚本 + 一条计划任务 | 每机一次（你批准后手动） | ✅ 有脚本 |

> 看门狗脚本**不是 skill 产物、也不是项目初始化产物**，是"崩溃恢复"这个主机能力的实现（README 升级路线里的定时器/守护脚本类）。skill 本体保持纯 markdown，脚本只是可选外挂；源文件在 `night-shift/scripts/`，安装时复制到 `$DSH_HOME/night-shift/scripts/` 由计划任务调用。

## 人类怎么用

### 一次性安装

```powershell
Copy-Item -Recurse night-shift ~\.dsh\skills\ -Force
```

（本仓库是基准，改完复制过去即生效，skill 目录被实时监视，**无需重启**。）

### 一次性配置（三件事，均为 DSH 配置层，非本 skill 内容）

1. **重试：保持默认 5 次，不设 `always`**（2026-10-30 裁定）。`retryPolicy: always` 是 **provider 级**配置，会作用于**所有会话（含白天）**，不现实；且模型不可用/余额不足时无限重试毫无意义。默认 5 次在白天网络与余额正常时够用，需要时可临时调大：

   ```yaml
   retryPolicy:
     mode: normal
     maxRetries: 10      # 按需调大；默认 5
   ```

   夜班的重试耗尽**由心跳接管**（见下）：单轮耗尽 → 该轮失败 → 下一轮心跳继续；连续多轮失败 → 判"模型不可用/余额不足" → 停心跳 + 标 blocked + 写简报，不整夜无效重试。

2. **审批策略**：设为 `never`（无人值守前提；引擎会硬拒敏感操作，是 skill 软约束之下的兜底）。
3. **AGENTS.md 夜班豁免条款**（两条子条款，状态不同）：
   - **commit 豁免 ✅ 已落地**（2026-10-30 写入全局 AGENTS.md §4.1.1"唯一常设例外"）：动机=**按任务分隔工作区**（每个任务收尾自主 commit 一次，工作区对下一任务保持干净）；夜班/挂机模式一任务一提交，commit 规范与测试状态不豁免，diff 审查移至晨检；**push/amend/rebase 仍永久禁止**；违规即撤销豁免。
   - **依赖豁免 ✅ 已落地**（2026-10-30 写入全局 AGENTS.md §2.1"唯一常设例外"）：声明为无人值守的**新增功能类**任务中，满足 `habits §〇-1 四条件`（调研有出处 / 精确锁版本 / 装包卫生（默认 registry+防拼写劫持+禁 `--force`/禁全局安装）/ 单独 `chore(deps)` commit 可回滚）的装包可自主执行，BRIEF 单列供晨检；**修复类任务装包不在豁免范围（红线不放）**。违规（无出处/拼写劫持包/混包进功能 commit/修复类装包）即豁免撤销退回逐轮确认。
   > 留给用户决策的依据：原规范要求 commit 与装包均逐轮授权（2026-10-26 教训）；挂机取舍为"commit 提权 + 功能类装包条件提权、push 与修复类装包留人"。改变条件见《夜班流水线落地清单》§0 决策 1。

### 主机级（跨项目唯一：**要崩溃恢复，不要开机自启**）

看门狗/运行登记表是**主机能力，不是项目能力**——DSH 只有一个进程，崩了所有项目会话一起挂，所以看门狗必须全局唯一。完整设计见 `references/host-level.md`。

> **术语澄清**："运行登记表"= `$DSH_HOME/night-shift/runs/` 下的 JSON 文件清单（每个进行中的夜班一条），**与 Windows 系统注册表（regedit）无关**，不读写任何系统注册表项。

| 组件 | 要不要 | 作用 |
|---|---|---|
| **看门狗 + 一条调度任务** | ✅ **要**（崩溃恢复） | 每 10 分钟检查：运行登记表有活动夜班 **且** DSH 不在 → 拉起；运行登记表为空 → 检查完直接退出，**绝不拉起 DSH** |
| **开机自启** | ❌ **不要** | 登录时无条件启动 DSH——**不打开=你当前不需要 DSH**，任何东西都不得无条件拉起它 |

> **调度任务常驻 ≠ 开机自启**：前者只是每 10 分钟一次轻量检查（无活动夜班立即退出），后者是登录就启动。两者别混。

```
$DSH_HOME/night-shift/            # Win: ~\.dsh\night-shift\   Mac: ~/.dsh/night-shift/
├── config.json                   # centralKnowledge / dshExecutable / heartbeatSeconds / stallMinutes / staleHours
├── runs/<sessionId>.json         # 进行中的夜班，每会话一条（多项目并发互不覆盖）
├── watchdog.log / watchdog.lock
└── scripts/windows|macos/        # setup + watchdog（按 OS 分；macOS 仅设计对齐，不实测）
```

- **装**：Windows `pwsh -NoProfile -File scripts/windows/setup.ps1`（任务计划每 10 分钟，经 `watchdog-launch.vbs` 由 `wscript.exe` 无窗口启动 pwsh 7）/ macOS `bash scripts/macos/setup.sh`（LaunchAgent `StartInterval 600`）——**都不注册开机自启**；用法与行为契约见 [`scripts/README.md`](scripts/README.md)
- **幂等**：已装则更新；`--uninstall` 卸载；**重启/杀进程都不需要重新初始化**
- **项目初始化只检查、不安装**：缺主机级时在预检单提示并给安装命令
- **降级**：未装看门狗时，崩溃后你手动打开 DSH，心跳会自动恢复冷会话并续跑（只是失去"自动拉起"这一层）

### 模型：全角色继承会话（B 仓不可单独配）

**主窗口模型 = 你建会话时选的模型，skill 不改它**——调度、方案判断、串行修复回路、所有持久队友都用它。

> ⚠️ **B 仓不使用子代理模型配置**：`SpawnTeammateRequest` **没有 `model` 字段**（JSDoc：inherits the stored configuration; do not supply a new model），`spawn_teammate` 建立的**持久队友一律继承会话模型**；`list_agents` 里的 `provider?` / `model?` 只是展示字段。**没有 `tool-subagent.agentOptions` / `modelSelectionSettings` / `maxTokens` 这类可配项**——那是 A 仓 `dsh-tool-subagent` 的能力。

因此：

- 想让整夜便宜，最直接的是**用便宜模型建夜班会话**（skill 规则不变）；
- 想要「worker 便宜 / 主循环贵」的**模型路由**，请用 **A 仓 `night-shift`**（A/B 不可混用，见顶部声明）；
- 本仓的优化方向不是模型单价，而是**持久队友复用上下文**（不用每轮重扫项目结构）+ **共享任务板** + **持久「审阅」记住上轮问题、验证是否真修**。

**不需要在全局 AGENTS.md 加标记**：插件是否已启用是主机级一次安装、全局生效的事，不是每项目初始化；判定走**每对话插件轨道自检**（工具列表里有没有 `spawn_teammate` / `team_task_*`），`config.json` 的 `hostSetup.agentTeamPlugin` 只作提示，项目初始化读它即可跳过重复询问。

> **profile 差异**：Agent Teams 由 profile 的 `dsh.profile.bundles` 决定，是**实例级**开关；用户级 preset 跨 profile 共用，但新建对话只能选 preset、选不了 profile。切轨要改 bundle + 重启 DSH。

### 崩溃后自动续跑（三层，只有第一层是原生）

| 层 | 机制 | 谁负责 | 状态 |
|---|---|---|---|
| **续跑** | `schedule_create` **`every_seconds`** 周期心跳提醒（15 分钟）——宿主重启后提醒表仍在，到点**恢复冷会话**并投递，agent 收到后接着跑。⚠️ 参数必须是 `every_seconds`，不是一次性 `after_seconds` | skill 自动建 | ✅ 已写进 SKILL §0 |
| **进程** | **看门狗（崩溃恢复，要）**：运行登记表有活动夜班 且 DSH 进程不在 → 拉起；运行登记表为空 → 不动。单例、跨项目共用；判定逻辑见下与 `references/host-level.md` | 需你批准才装（一条调度任务） | ✅ 已装（2026-10-30，`DSH Night Watchdog` 每 10 分钟；2026-11-01 实测两次成功拉起） |
| **状态** | `night-shift/` 全部 append-only 状态文件，任何时刻重启都能无损接上 | skill 自动维护 | ✅ 已有 |
| **整机** | 电脑关机/断电/蓝屏 → 整机离线；**不做开机自启**（你自己打开 DSH），恢复后由上面两层自动接上 | 你 | ✅ 设计如此 |

**术语区分**：本方案**不做开机自启**（不打开 DSH = 你不需要它）；但**做崩溃恢复**——看门狗只在"夜班进行中"才可能拉起 DSH。

### 看门狗判定逻辑（别用"任务时长"当阈值）

**错误做法（已作废）**：看某个文件"20 分钟没更新"就判定卡死 → 长任务（>1 小时）会被误杀。原因：长任务期间模型在忙，没机会更新文件；用"模型写的信号"判存活，会把"忙"误判成"死"。

**正确做法**：用**引擎写的**信号——会话日志（`~\.dsh\sessions\--<工作区编码>--\session-<uuid>\session.jsonl.zstd`）的最后写入时间。关键：**心跳提醒每 15 分钟投递一次，投递本身就会写会话**（收件箱 followup + flush），与模型是否在忙无关。

| 观察结果 | 判定 | 动作 |
|---|---|---|
| 进程不存在 | 崩溃 | 拉起 DSH → 下次心跳到期时宿主恢复冷会话并投递 → 自动续跑 |
| 进程存在 + 会话日志 40 分钟内有过写入 | 活着（**长任务也安全**） | 不动 |
| 进程存在 + 会话日志 >40 分钟无写入 | 卡死（或投递也停了） | 先优雅关闭，再强杀 + 拉起 |

- 阈值 = **2×心跳间隔 + 10 分钟余量**（心跳 15 分钟 → 40 分钟），**与任务时长无关**：跑 3 小时的任务照样每 15 分钟有一次投递写入。
- **前提：只在夜班进行中守护**——开工在**主机级运行登记表**写一条 run（`$DSH_HOME/night-shift/runs/<sessionId>.json`），收尾删除；**运行登记表为空时看门狗只做一次检查就退出，普通打开 DSH 绝不受影响**。
- **三条防误杀**：① 条目 `startedAt` 超 **18 小时**视为陈旧，看门狗忽略（防收尾漏删导致白天反复重启）；② **心跳 / run / 项目内状态同生共死**（删心跳必删 run）；③ v1 **只判"进程不存在"**，进程活着绝不动手。
- 多项目并发：每个会话一条 run，互不覆盖；**看门狗始终只有一个**（全局单例，用 `watchdog.lock` 防重叠）。
- 第一版建议**只判"进程不存在"**（保守、零误杀），"日志停滞"判定先观察几晚误报率再开。完整协议见 `references/host-level.md`。

**限制**：① 心跳只在 **Web 宿主（桌面 GUI）** 可用，headless 组合不支持 schedule；② **睡眠期间定时器不跑**（进程冻结），唤醒后只补发最近一次——所以"关屏幕不睡眠"仍是硬要求；③ **成本模型**：每次心跳 ≈ "当时整段会话上下文"（固定开销 15k~30k + 历史 10k→150k 递增），**不是那条提醒文字的 100 tokens**；只有"真正起了一轮 turn"才花钱（⚠️ 2026-11-01 实测：会话**忙时**到点的提醒既没当场投递、turn 结束后也没补投——"只进队列"这一条**未被证实**，别指望忙时错过的那次还回来）；④ 只补发最近一次，长停机不会补一串；⑤ 夜班结束删心跳 = 白天零成本。**间隔默认 15 分钟**（读 `config.json` 的 `heartbeatSeconds`，可在该文件里改；心跳与直线任务走便宜模型、夜间价格约白天一半，成本可接受；跑一晚看 GUI token 统计再校准）。

### 首次使用必做：可用性实测（无人值守，先验证再挂机）

> **为什么必须测**：这个 skill 的使用场景是**人不在场、全自动跑**。它"看起来装好了"不代表"能自己跑"——这条链上有 4 个环节容易静默失效：① DSH 会静默丢弃 frontmatter 解析失败的 skill；② 子代理 `agentOptions` 配错时**不报错，直接退回父代理模型**；③ 心跳用错时序参数（一次性而非周期）时，到点会话忙就永久错过；④ 挂机门禁没执行时，你拿不到"可以走了"的信号。
>
> **所以：第一次用、或每次升级 DSH / 换机器后，先花 ~15 分钟跑完下面两项实测，全绿再挂机。** 不测就挂机，风险是"第二天早上任务从没开始 / 跑一轮就停 / 一直空转烧 token"，而且失败时不报错。

#### 实测 A：子代理路由（1 分钟，必做）

**新开一个会话**，整段粘贴：

```text
帮我做一个「子代理路由自检」。只做这几步，不要修改任何文件、不要提交、不要安装任何东西、不要派第二个子代理：

1. 调用 subagent 工具派一个子代理，它的 prompt 用这一段（原样）：
   不要调用任何工具、不要读取任何文件。用一句话回答两件事：
   ① 你系统提示开头 "powered by … model" 那句的原文是什么？
   ② 你能看到的模型标识是什么？

2. 等它返回，把它的话原样贴出来，不要改写、不要总结。

3. 按下面规则给出结论行：
   - 子代理报告的模型标识与你在 agentOptions 里配的便宜模型一致 → RESULT: PASS
   - 报告的是父代理模型或其它值 → RESULT: FAIL
   - subagent 工具本身报错 → 原样贴出错误并 RESULT: ERROR

最后只回：子代理原话 / RESULT 行。
```

- `PASS` → 子代理路由正常。
- `FAIL` → 查 `references/host-level.md §十`（两个坑：provider 拼写、agentOptions 写在哪个 preset）。
- `ERROR` → 错误信息通常点名 provider，贴回排查。

> ⚠️ **B 仓不适用上面这段「子代理路由」实测**：本仓**不使用** `subagent`（编排选择，不是工具被禁；实测两套工具可共存），且**没有模型可配**（`SpawnTeammateRequest` 无 `model` 字段）。B 仓的「实测 A」改为下面的**插件轨道自检**，1 分钟、不修改任何文件、不提交、不安装、不 spawn 队友：
>
> ```text
> 看本会话的工具列表，原样回答两件事：
> ① 是否存在 spawn_teammate / list_agents / send_message / interrupt_agent /
>    team_task_create / team_task_get / team_task_list / team_task_update？
> ② 是否存在 subagent / subagent_fork？
> ① 全在 且 ② 全不在 → RESULT: PASS（B 轨就绪）
> 否则 → RESULT: FAIL（列出缺哪几个 / 多哪几个）；工具列表读不到 → RESULT: ERROR
> ```
>
> `PASS` 才能继续门禁；`FAIL` → **门禁不放行**（不弹卡片、不生成 `GATE.md`、不挂 Goal），切轨 = 改 `profiles/<profile>/package.json` 的 `dsh.profile.bundles` **并重启 DSH**（实例级，新对话只能选 preset）。**不降级、不静默继续**。

#### 实测 B：skill 主链路（约 10 分钟，必做）

**测前先把间隔临时调短**（否则一次测试要等十几分钟）：

```powershell
# 看门狗 3 分钟（提权；改计划任务触发器）
pwsh -NoProfile -Command "$t=Get-ScheduledTask -TaskName 'DSH Night Watchdog'; $a=$t.Actions; $tr=New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) -RepetitionInterval (New-TimeSpan -Minutes 3); $s=Get-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable -MultipleInstances IgnoreNew; Set-ScheduledTask -TaskName 'DSH Night Watchdog' -Action $a -Trigger $tr -Settings $s"

# 心跳 900 → 60 秒
(Get-Content "<DSH_HOME>\night-shift\config.json" -Raw) -replace '"heartbeatSeconds": \d+', '"heartbeatSeconds": 60' | Set-Content "<DSH_HOME>\night-shift\config.json" -Encoding utf8
```

**准备一个 5~10 分钟能做完的小任务**（写进文件，别粘进对话），例：`.tmp\smoke-plan.md`

```markdown
# 冒烟任务
- [ ] 1. 生成 .tmp\smoke\index.html：单文件、无外部依赖、内联样式，含 3 段说明文字
- [ ] 2. 用 pwsh 统计文件大小并写进 night-shift/summaries 的任务总结
- [ ] 3. 在文件里把两条任务标 ✅ 并附总结链接
```

**新开一个会话**，粘贴（路径换成你的）：

```text
按 night-shift skill 进入无人值守夜班模式，完成 <你的工作区>/.tmp/smoke-plan.md 里的全部任务。
清单就在那个文件里。我现在离开设备，不会回答任何问题；拿不准按 skill 的习惯默认处置并记 DECISIONS.md。
做完输出 === DONE === 加产物路径与文件大小。
```

然后按信号操作，逐步核对：

| # | 你要做的 / 要看到的 | 判定 |
|---|---|---|
| 1 | 聊天窗口弹出 **`present` 的门禁卡片** / 正文出现 `✅ NIGHT-SHIFT READY TO LEAVE` 块 | 没有 = 门禁未执行 → 查安装副本是否最新 |
| 2 | 磁盘上出现 `night-shift/GATE.md` | 硬留证，没有就说明门禁没做 |
| 3 | `~/.dsh/night-shift/runs/` 出现本会话 JSON | 看门狗才认这个 |
| 4 | 在它干活途中杀 DSH：`Stop-Process -Name 'DeepSeek Harness' -ErrorAction SilentlyContinue` | — |
| 5 | 等看门狗间隔（测试 3 分钟），`watchdog.log` 出现新的 `starting` 行 | 没拉起 → 查计划任务是否存在、间隔、动作 |
| 6 | DSH 自动弹回后打开会话，它自己接着干完（`=== RESUMED` → `=== DONE ===`） | 不续跑 → 查 HEARTBEAT.md / schedule_list / runs |

**四项全绿才算 skill 可用**：门禁卡片 + GATE.md + 看门狗拉起 + 任务续完并自清（删心跳/run、有插件则调 `sweep_schedule_inbox`）。

**测完把间隔改回去**：看门狗 10 分钟、心跳 900 秒（同样两条命令把 `-Minutes 3` 换回 `-Minutes 10`、`60` 换回 `900`）。真实夜班用大间隔，间隔太小会烧 token。

> 详细版（含每个失败现象的排查表、插件安装、积压处理）在 `references/host-level.md §十一 / §十二`。

### 首夜前置检查（三项全绿才开夜班）

1. **续轮验证**：建一个测试 goal，确认 agent 空闲后会自动进入下一轮（说明 goal-round-driver 已挂载）；
2. **重试验证**：确认默认 5 次重试可用（看会话日志 `llm/retry` 事件）；**不装 `always`**；
3. **审批验证**：确认 `never` 下没有工具被意外硬拒（被拒就检查该工具是否要求审批）。

> 不验证就跑，风险是"只跑一轮就停"或"网络一抖就断"，而且两种都不报错。

### 每天睡前

1. **把任务写进 `BUG-BOARD.md`**（或 `TODO.md` / 你自己指定的清单文件），**不要直接在指令里列**——指令里列的清单会随会话历史每轮重付，且你半夜改不了；落文件后 agent 按需读取、未读不花钱，你也能中途增删改（SKILL §2 第 2 条）。
2. 对 DSH 说一句话，例如：

   > 按 night-shift 挂机，自动完成 BUG-BOARD 全部任务。

3. **回答预检单**（约 3–5 分钟，这是你睡前唯一要花的交互）：agent 挂 Goal 前会扫一遍任务列表，把"夜里没法自己解决的事"一次性问完——需求歧义 / 设计取舍 / 预测依赖（含包名+出处，可当场预授权）。**可以只回一句"都按你建议的"**；没答的挂起，夜里落回默认。预检单存 `night-shift\preflights\<日期>.md`（按日期留档不覆盖）。
4. **关屏幕不关机**（睡眠会停一切）；确认网线/代理稳定。
5. **确认 goal 续行已启用**：若会话曾 resume/fork，续行会保持**停用**，需你明确授权 resume——引擎不会自行复活。这是夜班最常见的"以为在跑、其实停了"。

### 每天早上（5 分钟，模板在 `references/templates.md` §5）

1. 打开 `<工作区>\night-shift\briefs\<今日>.md` 扫总览表 + **"依赖变更"单列表**（有装包的夜必看：包名/出处/授权级别/lockfile diff 摘要）。简报**按日期留档永不覆盖**（`BRIEF.md` 只是最新一份的指针），晨检改动 habits.md 时须按 templates.md §2"习惯回写记录"留旧行原文。
2. 抽查决策日志（**按来源级别从低到高**：`默认表`/`白名单` 是没经你眼的，先看这两级）：不同意某条默认 → 改 `~\.dsh\skills\night-shift\references\habits.md` 对应行（**习惯蒸馏闭环**），按 templates.md §2 记"习惯回写"；**再把改动复制回仓库基准**（你 fork/克隆的 skill 仓库里 `night-shift/references/habits.md`）——否则下次同步会被旧版覆盖。
3. 批掉"待拍板"里当场能批的（缺条件的装包四要素齐全才批）。
4. 检查 commit 与证据文件 → 没问题**手动 push**（这是流程中唯一"必须等你"的动作）。
5. 把"顺手发现"排进今晚任务列表。

### 三阶段放量（别跳级）

| 阶段 | 跑什么 | 进入下一阶段的门槛 |
|---|---|---|
| ① 验证夜 | 单任务挂机 | 连续 ≥3 夜验收全绿、决策日志无严重偏差 |
| ② 并行夜 | 2 个独立任务并行 | 验收通过率 ≥90%、平均返工 ≤1 轮（口径：从当天 BRIEF 任务清单手数，完成数/总数） |
| ③ 全量夜 | 批量任务 + 自动返工循环 | 稳定高位后再考虑放开更多（如 push） |

### 依赖引入策略（2026-10-30 定稿：按任务类型分治 + 开工预检）

**当初的设计漏洞**：初稿把"装依赖"一刀切进禁区——对修 bug 是对的（红线不放），但**新增功能合法地需要新依赖**，一刀切等于功能类任务夜里基本只能 blocked，夜班价值减半。

**定稿方案**（用户裁决）：

| 任务类型 | 处置 | 授权从哪来 |
|---|---|---|
| 修复类（bug/重构/性能） | **红线：永不装** | 不授权——修复有纯本地替代路径，装包会改变故障面、掩盖真因；绕不开就重新定性为功能 |
| 新增功能类 · 档 2 来源白名单 | 满足四条件可自主装（见上文 AGENTS.md 依赖豁免） | agent 开工调研的**出处**（官方文档/GitHub/知识库）+ 锁版本 + 单独 commit 兜底 + 晨检复核 |
| 新增功能类 · 预检预授权 | 睡前预检单里你**当场点名"允许装 X"** | 你亲自背书，最高优先级 |

**为什么是"档 2 + 开工预检"而不是纯档 1（全靠你预点名）或纯档 3（全 blocked 次日续跑）**——用户的理由：任务少时你能先看看再决定，但**没见过的新功能你本人不调研也定不了**，与其让你凭空点名（档 1 做不到）、不如让 agent 带着出处来问 + 白名单兜底（档 2）；同时预检把人工决策点从"夜里半路卡住"平移到"睡前 5 分钟人在场时"，两头的短板都补掉。

**配套审计**：决策日志每条带 `来源` 字段（预授权 > 预检默认 > 白名单 > 默认表），晨检**从低级别看起**——低级别 = 没经你眼的；依赖变更在 BRIEF 单列 lockfile diff 摘要。发现问题时不要只纠正这一次，改 `habits.md` 对应行。

**lockfile 还原（`npm ci` 等）不在此列**：只复现既有依赖图、不改变它，修复与新功能一律自由区（用户确认维持）。

### 上线前用 eval 验证习惯规则

habit 规则（habits.md 默认表 + 夜班 preset）在正式挂机前用 eval 两层验证：① 共享端到端集（整条流水线跑通）② 每个 skill 小集；改动前后**逐任务对比找"结果反转"**（总分不降也要揪出变差的任务）；**跨模型矩阵必跑**（便宜直线模型单独测）。详见《夜班流水线落地清单》§1.3 与修订 2。

## 为什么现在是纯 skill（不写插件/小系统）

**总原则：skill 写"怎么做"，配置写"引擎强制什么"，脚本写"质量硬门禁"，插件只留给"模型自觉不可靠"的部分——先用 skill 跑，用实测缺口决定写不写，不预付开发和维护成本。**

> **唯一例外**：崩溃恢复的**看门狗脚本**是可选的主机级外挂（见上"产物归属"），不参与 skill 加载、不影响纯 markdown 本体——它解决的是"进程崩了没人拉起"这种引擎层问题，指令层管不了。

当前纯 skill 成立的四个理由：

1. **需求的 90% 本来就是"指令"**：验证取证、习惯决策、总结沉淀、写简报——这些是行为规范，markdown 恰好是表达行为规范的载体；DSH skill 机制支持热更新（改完即生效，无需重启），迭代成本几乎为零。
2. **硬的部分引擎已经有了**：审批 `never`、沙箱、默认 5 次重试都是 DSH 配置层（硬约束），skill 不重复造；skill 是软约束，叠在引擎硬闸**之上**而不是替代。
3. **质量硬门禁独立于 agent 存在**：验收命令（tsc/lint/test）是项目里的普通脚本，agent 换了、skill 卸了，门禁还在——这是讲话"只靠软层代码库迟早变垃圾"的正确落法，而它本来就不该长在 DSH 里。
4. **维护成本不对称**：一个插件要写代码、打包、跟 DSH 版本迭代；一个 skill 就是几页 markdown，实测发现写错了改一行。**先 skill 跑一到两周，让 ESCALATION.md 攒出真实缺口**，再决定为哪个缺口付费——缺口是攒出来的，不是猜出来的。

## 未来什么时候加脚本 / 小系统 / 插件（升级路线）

夜班执行中遇到"这次靠默认蒙对了，但规则没法保证下次"的场景，agent 会记一行 `ESCALATION.md`。攒出来的候选按下表对号升级：

| # | 触发场景（ESCALATION 里长什么样） | 升级到什么 | 为什么 skill 不够 |
|---|---|---|---|
| 1 | "项目 X 每次现拼验收命令，有漂移风险" | **仓库脚本** `scripts/verify.ps1`（普通脚本，非 DSH 插件） | 命令拼接是确定性工作，脚本一次写对永久复用；质量门禁本就该独立于 agent |
| 2 | "担心模型偶尔忘写 BRIEF/总结"（晨检发现漏报） | **小插件**挂 goal 收尾/turn 结束钩子：检测文件缺失则强制补 | 模型自觉是概率性的；"文件必须存在"是确定性要求，只能引擎保证 |
| 3 | "想要 23:00 自动开跑，不想睡前手动说一句" | **小插件**（cordis-plugin-timer）定时拉起会话 | 定时触发发生在模型不参与的时机，指令层管不到 |
| 4 | "发现模型有几次试图 push（虽被 AGENTS.md 挡住）" | **仓库 git hook** `pre-push` 硬拒（普通 git 脚本） | 最在意的红线应有硬闸，几行脚本零 DSH 依赖 |
| 5 | "eval 手工跑太费劲，跨模型矩阵想自动化" | **小系统**：eval runner 脚本调 DSH workflow/CLI | 批量执行+计分是机械工作，不该烧模型 token |
| 6 | "决策日志想按月统计习惯命中率，喂回 habits.md" | **小脚本**解析 DECISIONS.md 出报表 | 统计聚合是纯计算 |
| 7 | "多项目都想用夜班，配置复制来复制去" | **preset + 配置模板**（仍是配置层，不算插件） | DSH agent-preset 原生支持角色打包 |

**判断口诀**（写进 ESCALATION 记录时自问）：

- 模型"应该会做"但忘了后果严重？→ 钩子插件（场景 2）
- 发生在模型不参与的时机？→ 定时器插件（场景 3）
- 防的恰恰是"模型不听话"？→ 硬闸脚本（场景 1、4）
- 纯机械、批量、重复计算？→ 小脚本/小系统（场景 5、6）
- 以上都不是，只是"希望它更规范"？→ **还是改 skill，不升级**

### 已落地的配套插件：`dsh-plugin-schedule-sweep`

目前唯一从"升级路线"变成真实代码的配套插件，开源地址：
<https://github.com/firstsaofan/dsh-plugin-schedule-sweep>

它解决**任务完成前心跳堆积、完成后空转烧 token** 的问题（心跳每 N 秒往 agent 收件箱推一条提醒，
`schedule_delete` 清不掉已排队消息）。装了它，skill 收尾会多调一次 `sweep_schedule_inbox`
把积压清空。**可选**——不装 skill 照常能用，只是完成后会多几轮空转。

```powershell
# 从 Git 直接装（仓库自带构建产物，无需本地 build），装完重启 DSH
dsh plugin --profile desktop add https://github.com/firstsaofan/dsh-plugin-schedule-sweep
```

#### 装好后怎么验（两级）

**① 确认工具在（30 秒）**：重启 DSH 后新开会话，让 agent 调用
`sweep_schedule_inbox`（`dry_run=true`），返回 `{removed:0, remaining:0, scannedKinds:["schedule"]}`
即装好；说"没这个工具"则未加载（查是否重启、bundles 是否加了）。

**② 确认真的能删（约 5 分钟，可选）**：建 `every_seconds=60` 心跳 → 忙几分钟让收件箱堆几条 →
调 `sweep_schedule_inbox`（不带 dry_run）→ 看 `removed > 0` 且之后不再空转。

#### 间隔怎么选（装了插件 vs 没装）

| 是否装插件 | 推荐心跳 `heartbeatSeconds` |
|---|---|
| **装了插件** | **300 秒（5 分钟）** —— 完成后积压被 sweep 清空，间隔回到"任务节奏"作用；300s 推进快、堆积可控 |
| **没装插件** | **900 秒（15 分钟，skill 默认）** —— 没 sweep 兜底，只能靠大间隔压完成后的空转；**建议去装插件** |

> 不要用 60s 跑真实夜班（只用于测试）。看门狗间隔与插件无关，默认 **10 分钟**。

详见 `references/host-level.md §十一 自检C` 与 `§十二`。

## 边界（本 skill 不做什么）

- **修复类**不装任何依赖（红线）；功能类仅在四条件齐备时自主装（见上"依赖引入策略"）；不删文件、不 push、不碰工作区外——需要时写进简报等人。
- 不做质量门禁本身（那是项目验收脚本的事），只负责**执行并交付证据**。
- 不替代 AGENTS.md 的全局安全规范；冲突时以 AGENTS.md 为准。
