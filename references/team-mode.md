# 团队模式编排规则（`team-mode.md`）

> 本文件是 B 仓「派工层」的**规则正文**。`SKILL.md` §2A 是它的摘要，二者冲突时以 `SKILL.md` 为准（与 A 仓同样的「主手册优先」口径）。
> A 仓全部机制（预检单 / 门禁 / 心跳 / 看门狗 / 习惯决策树 / 验证证据 / 决策日志 / 简报 / 升级 / commit 豁免 / 熔断）**一条不少**，本文件只描述**团队层**。

## 1. Lead 与队友

- **Lead = 主会话，常驻，不 spawn**。职责：轨道自检 → 读 roster + `project-profile` → 拆票建卡 → spawn 全持久队友 → 下发 `context-pack` → 派工 / 打回 → 收口 merge 或交付汇总 → 写晨间简报。
- **全持久队友**（含「审阅」）：`spawn_teammate(name=<lower-kebab-case>, context=fresh)`；`context: fork` 继承 Lead 已完成轮次，用于需要对话历史的任务。
- `name` 必须 `lower-kebab-case`（工具限制）；中文显示名只用于任务卡 / 简报 / GUI。
- `list_agents` 的 `inactive` = **没有 turn 在跑**，不是任务完成、不是成功/失败；判断进度只看**共享任务板**。
- `send_message` 的 target 用 `spawn_teammate` / `list_agents` 返回的名字；成功发送 ≠ 对方已处理。
- `interrupt_agent` 打断某队友当前 turn（保留 pending inbox），仅用于死循环 / 明显跑偏。

## 2. 共享任务板（`team_task_*`）

**CAS 语义（必须先读后写）**：`team_task_list` / `team_task_get` 取最新 `revision` → `team_task_update(action=..., expected_revision=<最新值>)`。revision 过期会失败，重读再试，不要盲写。

工作流：`list`（看就绪）→ `get`（读完整详情）→ `claim`（认领）→ 干活 → `complete`（`action=complete`）；依赖用 `set_dependencies` / `blocked_by`，改派用 `reassign`，重开用 `reopen`。

### 任务卡字段

| 字段 | 类型 | 含义 |
|---|---|---|
| `subject` | string | 一句话任务标题 |
| `description` | string | 验收标准、输出路径、预检单里的专属规格、证据要求 |
| `owner` | target | 认领的 doer；审阅卡 owner = `review` |
| `blocked_by` | task id[] | 前置任务；**任务依赖只走这里**，不靠口头约定 |
| `status` | enum | 见 §3 状态机 |
| `round` | int | 当前返工轮次，初始 0；每次 FAIL `+1` |
| `review_notes` | string[] | **累积、不覆盖**的审阅意见（每次 FAIL append） |
| `write_scopes` | path[] | 预期写范围前缀（**advisory 非锁**） |
| `revision` | int | CAS 用，引擎维护 |

> 由 `team_task_create` 建卡时把 `round` / `review_notes` / `write_scopes` 写进 `description` 的 YAML 块（工具本身没有这些自定义列），后续 `edit` 时整体替换 description 并保留历史意见。

## 3. 状态机

```
pending → in_progress(claim) → review(doer 自验完成) → done(RESULT: PASS)
                                    │
                                    └─ RESULT: FAIL → 返工闭环(§4) → review(同一审阅)
任意状态 → blocked(round>=3 / 空转 / 冲突 / 二进制 / 会话级熔断)
```

| 状态 | 谁改 | 前置条件 |
|---|---|---|
| `pending` | Lead 建卡 | — |
| `in_progress` | doer `claim` | 卡的 `blocked_by` 全部完成 |
| `review` | doer 自验完成 | 证据已落 `evidence/<日期>/` |
| `done` | Lead / 审阅 | **仅 `RESULT: PASS`** |
| `blocked` | Lead | 写明卡点，进晨间简报「待白天裁定」 |

## 4. 返工闭环（双方都带上下文）

```
审阅 FAIL
  → review_notes.append(本轮意见)（累积，不覆盖）+ round += 1
  → 判 round >= 3 ?                        → 是 → blocked + 升级标记 → 简报「待白天裁定」
  → 判 本轮改动与上一轮实质相同（空转）?  → 是 → 同上
  → 否：send_message(累积 review_notes) → 打回【同一 doer 队友】
  → doer 带上下文改（同分支增量 commit / 同交付路径）
  → 卡 status → review → send_message → 送回【同一个「审阅」队友】
      （它记得上轮问题，可验证是否真修）
  → RESULT: PASS → done；FAIL → 回到本闭环开头
```

- **不换 doer、不换审阅**：持久性是 B 仓相对 A 仓的核心收益（A 仓重新派新 subagent 会丢上下文）。
- **空转检测**：本轮 diff / 产物与上一轮实质相同 → 立即 `blocked`，不再空转烧 token。
- 这是**第一道防死循环措施**；第二道是 `write_scopes` 划清输出路径，避免「A 改完 B 又改回去」。

## 5. `RESULT:` 结构化结论

「审阅」队友**只读**（共享 cwd、不改代码 / 产物），只写任务卡与结论：

```
RESULT: PASS
task: <task id>
evidence: <evidence 路径 / diff 范围>
notes: <可选，一句话>
```

```
RESULT: FAIL
task: <task id>
round: <当前轮>
issues:
  - <具体问题 1：文件:行号 / 与任务卡哪条不符>
  - <具体问题 2>
suggestion: <可执行的修改方向，不做审美评审>
```

- 非编码项目按 `references/validation-checklist.md` 的 **L0–L4** 核对，L3 内容合规 + 抽查 L4；**不做审美评审**。
- 编码项目读分支 diff + `references/verification.md` 的证据要求。

## 6. 隔离与交付

| `project-profile` | 隔离 | 收尾 |
|---|---|---|
| `code` + `git` + `worktree` | doer 各自 `create_worktree`（`feature/` / `fix/` / `ui/` + 任务 id） | **仅 `done + PASS` 后 Lead 在 `main` 顺序 `git merge`**，产生 merge commit 记录 |
| 其余 | `write_scopes` 输出路径前缀（advisory 非锁） | `collect`：交付目录汇总，**无 merge** |

- 冲突 / 二进制 → `blocked` 交白天；**不 rebase / amend / force / 自行裁决**。
- `create_worktree` 由**独立的「git 工作树」实验性插件**提供（**不是 Agent Teams 自带的**；agent-team 无 worktree 隔离、无 fs 锁）。启用路径：DSH → 插件 → 实验性插件 → 「git 工作树」。它只创建分支 + 切换调用会话 cwd，**不合并、不删分支、不开 PR**；清理已合并分支留白天。`isolation: worktree`（编码 + git）才需要它；`write-scopes` / 非 git 项目不需要。
- `git push` 永久人工闸门。

## 7. 熔断与参数

| 项 | 值 |
|---|---|
| 单卡 `maxRounds` | **3**（与引擎 `blockedAfterConsecutiveRounds=3` 对齐） |
| 会话级熔断 | 累计 `blocked >= 3` 或总时长触顶 → **停止派新卡、只收尾** |
| 空转检测 | 本轮改动与上一轮实质相同 → 立即 `blocked` |
| 单会话队友并发 | 开发 **4** / 非开发 **2**（= roster 预设，不另设限） |
| 一晚并发会话数 | ≤ 3（建议值，非硬值：单进程、每会话一组持久队友、主机级心跳共享） |
| 总时长 | 硬上限 `staleHours=18`；实际预算 = 人离开到回来 − 30 分钟缓冲（写进预检单） |
| 合并目标 | 本地 `main`，顺序、仅 `done + PASS` 后 |

**沿用 A 仓 / 引擎默认不改**：心跳 `900` 秒且必须 `every_seconds` 周期（非 `after_seconds`）、看门狗 10 分钟 + `watchdog.lock` 单例锁、`enableStallCheck=false`、`stallMinutes=40`、`staleHours=18`、`max_goal_rounds=30`（建 goal 时给，不写进 `config.json`）、`blockedAfterConsecutiveRounds=3`、门禁四信号、预检单挂 Goal 前写完。

## 8. 晨间简报（团队层增量）

在 A 仓简报模板（`references/templates.md`）基础上**按角色汇总**：每个 doer 的卡数 / PASS 数 / 返工轮次 / blocked 卡点；每个 `RESULT: FAIL` 的累积 `review_notes` 摘要；merge 列表（仅 `done + PASS`）；`context-pack` 索引；**依赖变更单列行**（包名 / 出处 / 授权级别 / lockfile diff 摘要）。

**留白天的清单**：① `git push` ② 开 PR（插件不内置）③ 清理已合并 worktree 分支 ④ 非编码产物最终交付 / 发布 ⑤ 依赖变更复核 ⑥ 所有 `blocked` 卡人工裁定。

## 9. 升级标记

遇到「这次靠默认蒙对了，但规则没法保证下次」或「需要引擎在我不参与时做事」的场景，按 `SKILL.md` §7 记一行 `ESCALATION.md`。团队层典型触发：`write_scopes` 反复重叠导致震荡（该加硬隔离）、`RESULT:` 格式被队友写歪（该做结构化校验）、任务板 revision 冲突频繁（该加串行队列）。
