[简体中文](README.md) | English

# night-shift-team — unattended automation pipeline (Agent Teams edition / B repo)

> **The name is historical.** It applies whenever **the human is away from the device** — night, commute, meetings, lunch. The only criterion: the agent can keep making progress without you.
>
> Design notes (in Chinese): [`DESIGN.md`](DESIGN.md)
> **Not on DSH?** See [`PORTING.md`](PORTING.md) — the skill is **bound to DSH's automation layer** (heartbeat / goal rounds / Agent Teams / crash recovery); the rules layer is portable.

> 🚨 **Hard prerequisite: enable BOTH experimental plugins in DSH — "Agent Teams" AND "git worktree" — or the full feature set will not work.** See the next section.

---

## 🔌 Hard prerequisite: enable two experimental plugins

> **In one sentence**: dispatch comes from `spawn_teammate` / `send_message` / `team_task_*` (**Agent Teams plugin**), while branch isolation for code projects comes from `create_worktree` (**git worktree plugin**). Those tools **exist only when the corresponding plugin is loaded**. Missing plugin → missing tools → the startup self-check returns `FAIL` → **the gate is blocked** (no card, no `GATE.md`, no Goal). **No downgrade, no fallback, no silent continue.**

**Two plugins, two jobs**

| Plugin | Tools it provides | When it is required |
|---|---|---|
| Agent Teams `dsh-experimental-agent-team-profile` | `spawn_teammate` / `list_agents` / `send_message` / `interrupt_agent` / `team_task_*` | **Always** (the dispatch layer depends on it 100%) |
| git worktree `dsh-experimental-tool-worktree` | `create_worktree` | **Only for `isolation: worktree` (code + git projects)**; not needed for `write-scopes` / non-git / `delivery: collect` |

> ℹ️ **Yes, `worktree` depends on the git worktree plugin**: Agent Teams itself has **no worktree isolation and no filesystem lock** (teammates share the cwd). `create_worktree` is a separate tool from a separate plugin; it only creates a branch and switches the calling session's cwd — it does not merge, delete branches, or open PRs. Content / `write-scopes` projects work with just Agent Teams enabled; to use B's **full** capabilities (code-project branch isolation + merge delivery), enable both.

### What happens if it is not enabled

| Symptom | Cause |
|---|---|
| `spawn_teammate` / `team_task_create` / `list_agents` missing from the tool list | Agent Teams plugin not loaded; those tools do not exist |
| Task cards, persistent teammates, shared board, rework loop all impossible | The dispatch layer depends on them 100% |
| Self-check `RESULT: FAIL`, gate blocked | Hard stop in `SKILL.md` §2A.1 |
| `create_worktree` missing in a code project → no branch isolation | git worktree plugin not enabled (use `isolation: write-scopes` instead, or enable that plugin) |
| Running A and B night-shift skills on the same instance interferes | Both contend for the same state dir / heartbeat / run registry / Goal / single-process watchdog assumption and have mutually exclusive orchestration; **not because subagent is disabled** |

### How to verify (1 minute)

Open a new conversation and check the **tool list**:

- ✅ `spawn_teammate` / `list_agents` / `send_message` / `interrupt_agent` / `team_task_create` / `team_task_get` / `team_task_list` / `team_task_update` are present
- ℹ️ **Do NOT check** whether `subagent` / `subagent_fork` disappeared: a 2026-11-06 test found them still listed, with no "not callable" note in the system prompt; the old criterion fails forever. This repo simply does not use them (an orchestration choice), not an environment check.
- ✅ **(only for code + git projects)** `create_worktree` is present

**Only the first item is the hard check — if it holds, the B track is ready.** The second is a note, not a criterion; the third matters only for `isolation: worktree`. You can paste the "Test A: plugin-track self-check" prompt from this README and ask the agent for `RESULT: PASS/FAIL`.

### How to enable

**UI path (recommended)**: DSH → **Plugins** → **Experimental plugins** → turn on both:

1. **Enable Agent Teams plugin**
2. **git worktree**

Then **restart DSH** (instance-level config; hot reload does not apply), open a new conversation and re-run the self-check until `PASS`.

> Equivalent config-layer route: edit `profiles/<profile>/package.json` → `dsh.profile.bundles` and add the built-in `@deepseek-ai/dsh-experimental-agent-team-profile` and `@deepseek-ai/dsh-experimental-tool-worktree`. Both live in the same "Experimental plugins" group, so enabling them together is simplest.

> ⚠️ **Instance-wide, not per-conversation**: new conversations pick a **preset**, never a profile. Every conversation on the same DSH instance shares this one track — you cannot have "teams in this chat, sub-agents in that chat".

### This is host config, not part of this repo

This repo is **plain markdown only**: no plugin fork, no `app.asar` edit, no `config.json` change, no registry edit, no packages installed. Enabling the plugins is a **one-time configuration on your local DSH**, separate from installing the skill (`Copy-Item` into `~\.dsh\skills\`) — **you must do both**.

> Skill installed but plugins off = a manual you cannot execute. Plugins on but skill missing = tools with no orchestration rules.

---

## Pre-flight checklist

> Check these in order before an unattended run. Items marked ⚠️ are **not part of the skill itself** and are the easiest host/project prerequisites to miss.

| # | Prerequisite | How to verify / do it | If missing |
|---|---|---|---|
| 1 | Both experimental plugins enabled (Agent Teams + git worktree) | See "How to enable" above; run the "How to verify" self-check in a new conversation | Gate blocked; the skill cannot run at all |
| 2 | This skill is installed | `Copy-Item -Recurse night-shift-team ~\.dsh\skills\ -Force`; `night-shift-team` appears in the catalog | Tools available, but no orchestration rules |
| 3 | ⚠️ **The workspace is a standalone, writable project directory** | See "Workspace requirements" below | State files land in the wrong place and pollute A repo's runtime records |
| 4 | ⚠️ **`merge` / `worktree` need a git repo with a local `main` branch** | `git -C <workspace> branch --list main` | `create_worktree` / `git merge` fail → `blocked` for daytime |
| 5 | ⚠️ **Your global `AGENTS.md` has the two night-shift waivers** | See "Waiver clauses" below | Night-shift `git commit` stalls on per-turn approval; feature-class installs cannot run autonomously |
| 6 | Host infrastructure installed (watchdog / heartbeat / scheduler / run registry / `config.json`) | Windows `pwsh -NoProfile -File scripts/windows/setup.ps1`; macOS `bash scripts/macos/setup.sh`; config at `$DSH_HOME/night-shift/config.json` | No automatic crash resume; project init will flag it in the preflight |
| 7 | Approval `never` + default 5 retries | DSH config layer; verify with the three "first-night checks" | Stalls waiting for a human / breaks on a network hiccup |
| 8 | Desktop GUI (Web host) + PowerShell 7.6 | `schedule` heartbeat only exists in the Web host; scripts need `pwsh` 7.6 (MSI build) | headless has no heartbeat; 5.1 mangles UTF-8 and scripts error out |
| 9 | Task list written to a file + preflight answered + you leave the device | List in `BUG-BOARD.md` or a file you name; preflight at `night-shift/preflights/<date>.md`; gate signals in `SKILL.md` §0 | Work never starts / stops after one round |

### Workspace requirements (⚠️ easiest to get wrong)

At runtime this skill writes into the **workspace root**:

```
<workspace>/night-shift/     state dir (preflights/ briefs/ summaries/ evidence/ DECISIONS.md …)
<workspace>/AGENTS.md        appended knowledge pointer block (idempotent, existing content untouched)
<workspace>/knowledge/       knowledge skeleton (on first entry)
```

- **Pick a standalone project directory as cwd, and make sure it is writable.**
- **Do NOT** use a parent directory that already contains A repo's `night-shift/` runtime state dir (e.g. the folder holding your whole skill collection) — B repo would write its state files into A repo's real records and pollute them.
- Both skills use the same state dir name (`night-shift/`), which is one more reason only one track may be live per instance.
- Non-git / `delivery: collect` content projects have no git requirement, but still need a standalone writable directory.

### Waiver clauses (downstream users must configure these)

The "commit waiver" and "feature-class dependency waiver" mentioned in this repo are clauses of **your machine's global `AGENTS.md`** (`~/.dsh/AGENTS.md`) — **not capabilities shipped with the skill**. Add these to your own global rules:

- **Commit waiver**: one commit per task under night-shift / unattended mode (`git add` + `git commit`) without per-turn approval; **push / amend / rebase remain permanently forbidden**.
- **Feature-class dependency waiver**: only for tasks declared unattended **and** of the new-feature class, and only when all four conditions hold (recorded provenance / exact pinned version / install hygiene / separate rollback-able `chore(deps)` commit). **Installing dependencies for fix-class tasks is always a red line.**

Without them the skill does not fail, but it will stall at night waiting for a human.

### Git prerequisites (only for `merge` / `worktree`)

| Combination | Requirement |
|---|---|
| `isolation: worktree` | Project is a git repo; `create_worktree` only creates a branch and switches cwd — no merge, no branch deletion, no PR |
| `delivery: merge` | A local `main` branch exists; Lead merges sequentially onto it, **only after `done + PASS`**; conflicts / binaries → `blocked` for daytime |
| Everything else (`write-scopes` / non-git / `collect`) | No git, no `main` branch needed |

> `git push` / opening a PR / cleaning merged worktree branches / final publishing of content artifacts → **always manual, daytime work**.

---

## ⚠️ Pick one track — never mix A and B

> One DSH **instance** can run only one of the two repos at a time: A (`night-shift`) or B (`night-shift-team`, this repo). Never enable both, and never run A in one conversation and B in another on the same night. **This is not because the tools are mutually exclusive** — a 2026-11-06 test found both tool sets coexisting on one instance. The real conflict is that the two skills contend for the same runtime infrastructure (project `night-shift/` state dir, `DECISIONS.md`, heartbeat, host-level `runs/` registry, Goal, single-process watchdog assumption) and have mutually exclusive orchestration (one-shot sub-agents vs persistent teammates + shared board). Agent Teams is controlled by **`dsh.profile.bundles` of a profile** — an **instance-wide** switch. New conversations pick a **preset**, not a profile. Both repos may exist on disk, but only one track is live per instance.
>
> **Switching tracks** = edit `profiles/<profile>/package.json` → `dsh.profile.bundles` and **restart DSH** (instance-level). Running two instances with different profiles is **not recommended**: the night-shift watchdog assumes a single `dshProcessName` / single DSH process (`references/host-level.md`); multiple instances break process detection and crash recovery.
>
> This is not a permanent choice: use A by day and B at night, as long as you change the bundle and restart in between.

## Which repo should you use?

> 🔗 **Sibling repo**: for solo/small projects, or when you want cheap-worker / expensive-orchestrator model routing, use [**night-shift** (original repo)](https://github.com/firstsaofan/dsh-night-shift). This repo uses persistent Agent Teams teammates, a shared task board, a persistent reviewer and a rework loop for medium/large projects; **the two cannot be mixed on the same DSH instance** (see below).

> Both repos are **the same unattended infrastructure with two different dispatch layers**: A (`night-shift`) uses one-shot `subagent` calls, B (`night-shift-team`, this repo) uses DSH's built-in Agent Teams **persistent teammates**. A is unchanged; the gate / heartbeat / watchdog / preflight / decision log / brief / evidence rules / circuit breaker are identical.

### Spec comparison

| Dimension | A: `night-shift` | B: `night-shift-team` (this repo) |
|---|---|---|
| Target | Solo / small projects | Medium–large projects, parallel tasks, review / rework / merge |
| Plugin | agent-team **off** (`subagent` available) | agent-team **on** (built-in, no fork) |
| Dispatch | `subagent` / `subagent_fork`, one-shot | `spawn_teammate` + `send_message` + `team_task_*`, all persistent |
| Teammates | none | 4 dev (feature/bugfix/ui/review), 2 content (general/review); Lead = main session, not spawned |
| Models | ✅ `tool-subagent.agentOptions` (cheap workers, expensive orchestrator) | ❌ inherited from the session, no per-role routing |
| Board | task file / checklist | `team_task_*` shared board (visible in Web GUI) |
| Isolation | none | code+git: `worktree` branches; else `write_scopes` (**advisory, not a lock**) |
| Review | sub-agent self-check + evidence | persistent **read-only** reviewer, `RESULT: PASS/FAIL` |
| Rework | spawn a new sub-agent | `send_message` back to the **same doer**, then the **same reviewer**; cumulative `review_notes`, `round>=3` breaker, no-op detection |
| Delivery | summaries / morning brief | git: Lead merges sequentially into `main` **only after PASS**; non-git/content: deliverable directory, no merge |
| Cross-session context | `DECISIONS.md` | `context-pack` (loaded once) + teammates' in-session memory |
| Infrastructure | preflight, gate card, watchdog, heartbeat, scheduling, run registry, habit tree, evidence, decision log, brief, escalation, commit waiver, circuit breaker | **kept in full** |
| push / PR | manual, daytime | manual, daytime (permanent gate) |

### When to pick which

**Pick A (`night-shift`)**
- Solo / small project, 1–3 mostly independent tasks per night;
- You want **model routing** to save money (cheap workers, expensive main loop);
- No code review, no rework loop — a task file + decision log + morning brief is enough;
- You don't want the Agent Teams bundle enabled;
- Already using A successfully — **no need to migrate**; A is maintained and unchanged.

**Pick B (`night-shift-team`, this repo)**
- Medium–large project, several parallel tasks with dependencies and a shared board;
- You need a **persistent reviewer** that remembers last round's issues and verifies fixes (a fresh sub-agent loses that context);
- You need a **rework loop**: FAIL → same doer → same reviewer → PASS;
- You need **merge delivery**: Lead merges into `main` only after PASS;
- Mixed code / drawing / video / document output in one night (`project-profile`);
- Heavy project context where persistent teammates beat re-scanning the repo every round.

**Neither**
- You are present and want to be asked → both skills fall back to normal Q&A;
- A single one-off task → just do it;
- Running both on the same instance → **impossible**, see "Pick one track" above.

### Switching cost

| Item | Note |
|---|---|
| Granularity | **Instance-wide**: the Agent Teams bundle lives in a profile's `dsh.profile.bundles`; new conversations pick a preset, not a profile |
| Switch | edit `profiles/<profile>/package.json` → `bundles`, then **restart DSH** |
| Coexist | both repos may be installed and listed in the catalog, but only one track is live per instance |
| Don't mix | running two DSH instances with different profiles breaks the watchdog's single-process assumption |
| Trade-off | A has no roster / board / rework / `context-pack`; B has no model routing and teammates share the session context |
| Suggested | A by day (supervised, cheap routing), B by night (heavy tasks + review), with one bundle change + restart in between |

> ⚠️ Both skills trigger on "night shift / run unattended". When both are installed, **name the repo explicitly** in your instruction (`night-shift` or `night-shift-team`).

**Prerequisites (B)**: (1) the instance's profile has the built-in `@deepseek-ai/dsh-experimental-agent-team-profile` bundle enabled — **no plugin fork, no `app.asar` edit, no DSH runtime change**; (2) install this skill; (3) host-level watchdog / heartbeat / scheduler / run registry are inherited from A — **check only, never install**. Every conversation starting this skill runs a **plugin-track self-check**; a mismatch blocks the gate (no card, no `GATE.md`, no Goal) and prints a switch-track hint — **no fallback, no silent continue**.

> Everything from A is kept: preflight, gate card, watchdog, heartbeat, scheduling, run registry, habit decision tree, verification evidence, decision log, morning brief, escalation markers, commit waiver, circuit breaker. Only the **dispatch layer** is replaced and a **team layer** added.

A DSH skill that makes an agent complete a task list while you are away: implement → verify (evidence-based) → decide by your habits → commit per task (never push) → write summaries → leave you a 5-minute morning brief.

---

## When it applies

Trigger phrases: "night shift", "run unattended", "keep working while I'm away", "finish the BUG-BOARD", or any statement that you are leaving the device.

If you are present, none of the "default decisions" apply — the agent asks as usual.

---

## Install

1. **Skill body** (once): copy this folder to `~/.dsh/skills/night-shift`. DSH hot-reloads it; no restart needed.
2. **Host-level watchdog** (optional, one per machine): `pwsh -NoProfile -File scripts/windows/setup.ps1` (Windows) or `bash scripts/macos/setup.sh` (macOS, untested). Registers a 10-minute scheduled check that starts DSH **only when a night-shift run is registered and the process is gone**. On Windows the task runs `watchdog-launch.vbs` through `wscript.exe`, so no console window flashes every 10 minutes. It does **not** register login auto-start.
3. **Models — inherited, not configurable (B repo)**: `SpawnTeammateRequest` has **no `model` field**; every persistent teammate inherits the session model. There is **no** `tool-subagent.agentOptions` / `modelSelectionSettings` / `maxTokens` here — those belong to A repo `night-shift`. To make a whole night cheap, create the session with a cheap model; for cheap-worker / expensive-orchestrator **model routing**, use A instead (never mix A and B, see above). Also **verify the plugin track** in every new conversation: `spawn_teammate` / `list_agents` / `send_message` / `team_task_*` must exist (**the only hard check**; `create_worktree` is checked only for `isolation: worktree`). Do **not** require `subagent` / `subagent_fork` to be absent — a 2026-11-06 test found them still listed; a mismatch blocks the gate and prints a switch-track hint.
4. **Retries**: keep the default (5). Do **not** set `retryPolicy: always` globally — it is provider-wide and would affect daytime sessions. Night-shift failures are absorbed by the heartbeat instead.

---

## How it works — four layers

| Layer | Contents | Installed | Owner |
|---|---|---|---|
| **Skill body** | `SKILL.md` + `references/*.md` (pure markdown) | once | repo → copied to `~/.dsh/skills` |
| **Project** | pointer block in project `AGENTS.md`, `night-shift/` state, `knowledge/` | once per project (automatic) | the skill at runtime |
| **Host** | `config.json`, `runs/` registry, watchdog + scheduled task | once per machine | you approve, scripts run |
| **Engine** | heartbeat (`schedule`), goal round driver, approvals, retries | built into DSH | DSH |

---

## One night, in short

```
you: "run night-shift-team on BUG-BOARD"  (plugin-track self-check first)
  → preflight questions (one shot; unanswered → habit defaults)
  → goal mounted (max_goal_rounds 20–40) + gate card + heartbeat + run registry
  → read roster + project-profile → create shared task cards (team_task_create)
  → spawn persistent teammates (4 dev / 2 content) → send context-pack
  → doers work + self-verify + evidence → persistent read-only reviewer: RESULT: PASS/FAIL
  → FAIL: send_message back to the SAME doer, same reviewer, round+1 (max 3)
  → PASS: Lead merges sequentially into main (git) or collects deliverables
  → wrap up: brief by role → heartbeat deleted → run entry deleted (push stays manual)
```

---

## Failure recovery

| Situation | What happens |
|---|---|
| DSH process crashes | Watchdog (10 min) starts it; next heartbeat restores the cold session and delivers "night-shift heartbeat"; the agent reads state and continues |
| One model call fails | Retried 5 times with backoff |
| All retries exhausted | That round ends in error; the **goal driver** starts the next round automatically |
| 3 consecutive blocked rounds | `blockedAfterConsecutiveRounds=3` lets the model mark the goal blocked |
| Rounds exhausted | Engine records a `round-limit` blocker and stops |
| Model unavailable / out of credit | Skill rule: 3 consecutive same-error rounds → delete heartbeat, mark blocked, write brief (no all-night spin) |

**Two independent drivers** — do not confuse them:

- **Goal round driver**: same session, "one round done → next round". Disabled after session resume/fork until you explicitly resume.
- **Heartbeat**: cross-process wake-up. Restores a cold session after a crash and delivers a prompt.

---

## Knowledge layers

| Layer | Where | Rule |
|---|---|---|
| **Raw** | `knowledge/{summaries,pitfalls,research}/`, `night-shift/{summaries,briefs,preflights,evidence}` | append-only: never rewrite, never delete; yearly archive migration allowed |
| **Derived** | `knowledge/patterns/` | editable, must point back to raw sources |
| **Universal** | `<centralKnowledge>` (host config) | admission gate: ≥2–3 independent occurrences, same root cause, valid across projects, with provenance; retired entries are status-marked, never deleted |

---

## Key parameters

| Parameter | Value | Where |
|---|---|---|
| Heartbeat interval | 15 min (default) | `config.json.heartbeatSeconds` — the skill reads it when creating the reminder |
| Watchdog interval | 10 min | scheduled task |
| Stale run | 18 h | `config.json.staleHours` |
| Stall check | off by default (40 min) | `config.json.enableStallCheck` / `stallMinutes` (keep ≈ 2× heartbeat + 10 min) |
| Model retries | 5 (default) | profile `retryPolicy.maxRetries` |
| Research time box | ~10 min per task | project template |
| Goal rounds | 20–40 | `max_goal_rounds` at goal creation |

---

## Outputs

- `night-shift/preflights/<date>.md` — questions asked before starting
- `night-shift/briefs/<date>.md` — morning brief (includes a "run integrity" field)
- `night-shift/DECISIONS.md` — decision log (including habit rewrites)
- `night-shift/ESCALATION.md` — candidates for escalation
- `night-shift/summaries/` — one summary per task
- `night-shift/evidence/<date>/` — test output and screenshots
- `night-shift/HEARTBEAT.md` — heartbeat reminder id

---

## Hard limits

- The `schedule` heartbeat only exists in the **Web host (desktop GUI)**; headless profiles cannot mount it.
- Timers do not run while the machine sleeps — keep it awake (screen off is fine).
- The watchdog only acts while a night-shift run is registered; it never touches DSH otherwise.
- macOS scripts are design-aligned but **untested**.
- Heartbeat cost ≈ the whole session context per idle round, not the 100-token reminder text.

---

## Portability

This skill is **DSH-bound**, but only in its automation layer:

| Layer | Portable? | What to change on another agent |
|---|---|---|
| Rules (habits, verification, templates, dependency/git policy, knowledge layers) | ✅ copy as-is | nothing |
| Heartbeat (crash resume) | ❌ DSH-only | cron/scheduler + agent CLI; **new session each time**, rebuild context from `night-shift/` state |
| Goal rounds | ❌ DSH-only | external loop script, or manual "continue" |
| Agent Teams persistent teammates + shared task board | 🔴 DSH-only | platform-native persistent agents / shared board; otherwise serial + file-based board, keeping the read-only review checklist and `RESULT: PASS/FAIL` |
| Instruction injection (`AGENTS.md`) | 🟢 concept is common | rename to `CLAUDE.md` / keep `AGENTS.md` |
| Liveness signal | 🟡 different format | the platform's session log path |

Full checklist: [`PORTING.md`](PORTING.md) (Chinese).

---

## Files

```
night-shift-team/
├── SKILL.md              agent manual (loaded by DSH) — incl. §2A team mode
├── README.md             human manual (Chinese)
├── README.en.md          human manual (English, this file)
├── DESIGN.md             design notes: diagrams, flows, file map (Chinese)
├── PORTING.md            porting guide: what to change on another agent (Chinese)
├── INTERNAL-dsh-mechanisms.md  internal DSH mechanism notes
├── LICENSE               MIT
├── DESENSITIZE.md        redaction rules + report format
├── references/           habits / verification / templates / project-init / host-level (kept from A)
│                         + team-roster-dev / team-roster-content / project-profile
│                         + team-mode / context-pack / validation-checklist (new)
└── scripts/              optional host scripts (not loaded by the skill)
    ├── windows/{setup.ps1, watchdog-launch.vbs, watchdog.ps1}
    ├── macos/{setup.sh, watchdog.sh}
    └── sync-to-open-source.ps1   redacted one-way sync (dry-run by default)
```
