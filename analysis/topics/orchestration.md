# 워크플로 / 오케스트레이션 비교

대상 커밋: opencode `907b3bc`, oh-my-openagent `251cbfe`, pi-mono `8369268`, oh-my-pi `69e8c9e`, codex `b741e48`, claude-code `1c229fc`, openclaw `ee9127b7`, hermes-agent `620ceb86` (2026-10-02~04)

## 요약

| 하니스 | 계획 모드 | 루프/연속 실행 | 병렬화 | 상태 관리 |
|---|---|---|---|---|
| opencode | `plan` 에이전트 = permission deny (`edit:"*"`) | **없음** | **없음** (프롬프트 단계 병렬만) | `todo` 테이블 + `todowrite`, 세션 `--continue/--fork` |
| oh-my-openagent | Prometheus + `ulw-plan` + **강제 훅** | `boulder` 상태기 + `ulw-execute` 재개 | **senpi-task DAG 엔진** (frontier, WAL, 복구) | 작업 레지스트리 + compaction 보존 |
| pi-mono | 없음 (예시 extension만) | 없음 (`onYield` 훅만) | **`durable` TaskGraph** (`failFast`/`allSettled`) | extension 예시, 세션 JSONL 분기 |
| oh-my-pi | `plan-mode/` + **settle 시 강제 루프** | **`/goal` + `/loop`** (exit-code 조건) | workpool 배치 fan-out (DAG 없음) | `TodoTracker` 미완료 알림 |
| codex | `ModeKind{Plan,Default}` (프롬프트 게이트) | **`ext/goal`** (3-tool, 토큰 예산) | 스레드 fan-out + `FuturesOrdered` | `update_plan` + **12개 훅** (Stop block) |
| claude-code | 6개 permission mode (`plan` 포함) | **`/loop`** + ralph 플러그인 (Stop hook) | **`Workflow` tool** (`agent()/parallel()/pipeline()`) | `TodoWrite`/`Task*` (**모델 게이트**) + 33개 훅 |
| openclaw | 없음 (ACP passthrough만) | `goal` = 모델 주도 + **loop 감지 서킷브레이커** | Swarm 수집기 + workboard 카드 링크 | `progress_card` (교체 방식) |
| hermes-agent | `/plan` (단일 턴, `.hermes/plans/`) | **Ralph 원본 `/goal`** + `/loop` + `/heartbeat` | **Kanban `task_links` DAG** + swarm | `todo_list` rev 카운터 + **stop gate** |

## opencode

- **계획 모드**: `packages/opencode/src/agent/agent.ts:156-181` — `plan` 프라이머리 에이전트, `edit: "*": "deny"` + `task` deny로 읽기 전용. `plan_enter`/`plan_exit` permission은 있으나 `plan-enter.txt`는 어디서도 import되지 않는 **고아 자산** → 진입은 유저, 종료(`tool/plan.ts`)만 툴. `OPENCODE_EXPERIMENTAL_PLAN_MODE` (`effect/runtime-flags.ts:47`) 게이트, 5단계 프롬프트 `session/prompt/plan-mode.txt`
- **루프**: **전무**. 트리에서 `ralph`/`boulder`/`DagRun` 0건. 루프 인접 기능은 `doom_loop: "ask"`뿐
- **병렬화**: DAG 없음. 병렬성은 "한 메시지에 최대 3개 에이전트"라는 **프롬프트 지시**. `subagent_depth` 기본 1 (`core/v1/config/config.ts:84`), experimental `tool/code-mode.ts`는 스크립트 샌드박스일 뿐 순서/영속성 없음
- **상태**: `todo` 테이블 (`core/session/sql.ts:100`) + `tool/todo.ts`. 다만 **어떤 코드도 todo를 근거로 재실행하지 않음**. 세션 재개는 사람 손: `cli/cmd/run.ts:147-158` `--continue/--session/--fork`
- **위임**: `tool/task.ts` 단일 툴, `<task_result>` 봉투, builtin 서브에이전트 2개

## oh-my-openagent (OmO)

- **계획 모드 3중**: `shared-skills/skills/ulw-plan/SKILL.md` (sticky plan mode, CLEAR/UNCLEAR 라우팅, 승인 게이트, 읽기 전용 위임 화이트리스트 85-93) + Prometheus 에이전트 (`agents/prometheus/AGENTS.md`, md-only 46-56) + **강제 훅** `hooks/prometheus-md-only/hook.ts:14-45` (비-`.md` 쓰기 차단), `hooks/plan-format-validator/hook.ts`
- **루프**: Ralph는 **트리에 있지만 폐기됨** — `hooks/ralph-loop/AGENTS.md:7-20` PR #6184 `goal-replaces-ralph`. 대체재 = `hooks/goal/` (session.idle 재주입) + `hooks/todo-continuation-enforcer/` (boulder 연속 기제: 카운트다운 토스트, 지수 백오프, 턴 경계 일시정지) + `skills/ulw-loop`. `/stop-continuation` (`plugin/stop-continuation.ts:25-29`)이 **5가지 기제를 한 번에 해체**
- **병렬화**: **진짜 DAG 엔진** `packages/senpi-task/src/dag/` (35파일, ~13.9k LOC) — `graph.ts` 사이클 검출·critical path, `scheduler.ts:1131` 의존성-frontier 승인, `journal.ts` WAL, `recovery*` 재시작 복구. 표면은 8-동작 `workflow` 툴 (`omo-senpi/src/components/task/dag-tool.ts:25`), `mass-ulw` 스킬, `docs/reference/mass-ulw-protocol.md` (4채널/17이벤트)가 계약 테스트로 고정
- **상태**: `tools/task/{task-create,task-get,task-list,task-update}.ts`가 `todowrite`를 **대체**하고 compaction을 통과하며, 미완료 todo가 연속 실행의 연료가 됨. boulder 상태기 `packages/boulder-state/src/types.ts` (`active|paused|completed|abandoned` + `stale_since`)
- **위임**: ~60파일 delegate 엔진 (sync/bg), 9개 의미 카테고리, 12개 `team_*` 툴 (mailbox + tmux), `call-omo-agent`
- **주의**: OmO는 대상이 **둘이다** — OpenCode 플러그인(`omo-opencode`, 훅 54~62종 = goal/todo/boulder)과 네이티브 `omo-senpi`+`senpi-task` (DAG/workpool/team). 귀속 시 대상을 명기해야 함

## pi-mono

- **계획 모드 없음**. 빌트인 슬래시 24종에 plan/goal/loop/todo 없음 (`coding-agent/src/core/slash-commands.ts:20-43`). `examples/extensions/plan-mode/index.ts`가 예시로 읽기 전용 툴 화이트리스트 + `Plan:` 섹션에서 번호 매긴 단계 추출 + `[DONE:n]` 마커를 제공
- **루프 없음**. 유일한 연속 원시물은 `durable/src/harness/generation.ts:511-515` `onYield` 훅 — **1회**만 fire되고, `agent-session.ts:1859-1862`에서 컨텍스트가 없으면 continuation을 **거부** (`_reportInvalidBoundaryContinuation`)
- **DAG = 이 저장소의 강점**. `packages/durable/src/harness/task-graph.ts:19-41` — `TaskGraphState`(`pending|running|waiting|completing`), `on: TaskId[]` + `JoinPolicy`(`failFast|allSettled`), `owner`가 곧 간선. 그래프는 durable commit에서 파생 (`advance()`, :152). 스펙 `durable/docs/spec.md:1568-1640`, 실행 예제 `test/examples/24-child-tasks.ts`
- **상태**: `examples/extensions/todo.ts`만 — tool result details에 저장되어 브랜치 시 자동 정합. **연속 실행 강제 없음**
- **위임**: `experimental/durable/subagent.ts:31-39` (`replay: "safe"`, 부모 abort가 자식을 살림), vacation 백그라운드 fan-out, `examples/extensions/subagent` (`MAX_PARALLEL_TASKS=8`, `MAX_CONCURRENCY=4`)
- **재개**: `durable/src/harness/harness.ts:236` `resume()`가 미완료 테스크 스케줄러 재시작, `requestId` 멱등

## oh-my-pi

- **계획 모드가 가장 강함**: `packages/coding-agent/src/plan-mode/` 10파일. `state.ts`의 `workflow?: "parallel"|"iterative"`는 pi-mono에 없는 ENUM. **settle 시 강제** `session/agent-session.ts:9999-10054` `#enforcePlanModeDecisionAtSettle()` — 결정 툴(`ask`/`write xd://propose`)을 안 불렀으면 reminder 누적 후 `toolChoiceQueue.pushOnce("required")`로 강제. 모델 role 전환 `interactive-mode.ts:4599`, compaction 생존 `plan-protection.ts:5` (`local://PLAN.md` 매처)
- **루프 2종**: `/loop` (`slash-commands/builtin-modes.ts:413-439`)의 continue 조건이 **셸 exit status** (`modes/loop-condition.ts:1-16`): exit 1만 false, 126/127/2는 "조건 자체가 깨짐"으로 분리 — 조건 오타가 완료로 착각되는 실패 모드 차단. 조건 셸은 `loop-condition:${sessionId}`로 격리 (:102). 한도 `modes/loop-limit.ts:5-12` (iterations|duration). `/goal` (`builtin-modes.ts:375-400`)은 `goals/state.ts:6-20` 상태기 + **무활동 감지** `goalContinuationActivity()` (:50-69) — 두 턴 연속 활동이 같으면 자동 continuation 중단
- **DAG 없음** (`\bdag\b` = mermaid layering만). 대신 `task/parallel.ts:26` `mapWithConcurrencyLimit`, `task/workpool.ts:110/629` keep-alive 워커 풀, `task/index.ts:1477` `#executeSyncFanout` (배치 한 번에 fan-out, 세마포어로 제한)
- **todo가 강제 루프**: `session/todo-tracker.ts:208` `checkCompletion()` — 미완료 있을 때 `<system-reminder>` + `scheduleAgentContinue({source:"todo-reminder"})`. eager prelude로 `tool_choice`를 `todo`에 강제 (:166), mid-run nudge (:299). settle 경로 `agent-session.ts:4225`
- **위임**: `registry/agent-registry.ts:29` (`main|sub|advisor`), `registry/agent-lifecycle.ts` TTL `idle→parked→revived`, worktree 격리 (`task/settings.ts:23-71` 9종 백엔드), `/vibe` 디렉터 모드 (`docs/vibe-mode.md`)
- **세션 정지**: `extensibility/shared-events.ts:110-120` `session_stop` — `decision:"block"` / `additionalContext` (Claude/Codex 호환)

## codex

- **계획 모드 = 협업 모드 2개 중 하나**: `codex-rs/protocol/src/config_types.rs:674` `enum ModeKind { Plan, #[default] Default }`. 하드 게이트는 `core/src/tools/handlers/plan.rs:87-91`의 **`update_plan` 거부 1건뿐**; 변이(mutation) 금지는 `collaboration-mode-templates/templates/plan.md:30-39`의 **프롬프트 정책**. 추가로 `core/src/session/turn_input.rs:66-83` `TurnStartKind::permits_mode`가 `Automatic` 턴의 Plan 진입/이탈을 차단. 별도로 `Op::Review` + `ReviewTarget{WorkingTree,BaseBranch,Commit}` (`protocol.rs:747,3464`)와 `core/src/guardian/` 21파일 승인 리뷰어
- **루프 = `/goal` 확장**: `ext/goal/` 13소스. 툴 3개 (`spec.rs:9-11` `get_goal/create_goal/update_goal`), 루프 본체 `runtime.rs:425-523` `continue_if_idle()` → `on_thread_idle` (`extension.rs:180-193`). continuation 프롬프트 `ext/goal/templates/goals/continuation.md`에 **Completion audit / Blocked audit**가 명문화 — blocked는 **3연속 턴** 반복 판정 (`spec.rs:77`), 토큰 예산 `accounting.rs`. `restore_after_resume` (`runtime.rs:401`)로 재개 시에도 유지. **Ralph·boulder·`/loop` 없음**
- **병렬화 = 스레드 fan-out**: `agent-graph-store`는 문서상 **parent/child tree**이지 DAG 아님 (`agent-graph-store/src/lib.rs:1`). `max_concurrent_threads_per_session` (`core/src/config/mod.rs:1343`)는 모델에 "{N} available concurrency slots"로 **알려주기만** (`prompts/src/multi_agent_instructions.rs:78`) — 세마포어 아님. 턴 내 툴 병렬은 `core/src/session/turn.rs:2604` `FuturesOrdered`
- **상태 + 훅**: `update_plan` (`plan_spec.rs:7-57`, pending/in_progress/completed). **12개 훅 이벤트** (`hooks/src/lib.rs:23-36`) — 엔진 이름이 `ClaudeHooksEngine` (`hooks/src/engine/mod.rs:222`). `Stop` 훅 `hooks/src/events/stop.rs:96-113`가 `should_block` + `block_reason` + `continuation_fragments`를 반환 → **Ralph 원시물 내장**
- **위임**: `multi_agents_v2/` 6툴 (`spawn/wait/followup_task/send_message/interrupt_agent/list_agents`). `ReasoningEffort::Ultra` → `MultiAgentMode::Proactive`, 나머지는 `ExplicitRequestOnly` (`core/src/session/multi_agents.rs:96-103`). delegate는 승인 정책 `Never` 강제 (`codex_delegate.rs:51-70`)

## claude-code

> 이 저장소는 CLI 소스가 아니라 **플러그인 + 타입 선언 + CHANGELOG**다. 근거는 `mods/types/claude-code.d.ts` (13,186행, 해당 빌드의 권위 있는 타입 표면), `CHANGELOG.md` (900KB), `plugins/`.

- **계획 모드**: **6개 permission mode** (`mods/types/claude-code.d.ts:6076`) `default|acceptEdits|bypassPermissions|plan|dontAsk|auto`. `plan` = "no actual tool execution", Shift+Tab 순환. **실제 강제 존재** — `CHANGELOG.md:3127` "plan mode auto-running file-modifying Bash commands (e.g. touch, rm)" 수정, `:316` fork가 부모의 plan mode를 벗어나지 못함. 별도 Plan 서브에이전트는 없음
- **루프 2갈래**: ① 네이티브 `/loop` — `ScheduleWakeup` (`d.ts:12512-12523`)가 `<<autonomous-loop-dynamic>>` 센티널로 self-paced 조절, `CronCreate`(`durable:true` → `.claude/scheduled_tasks.json`). ② **공식 ralph 플러그인** `plugins/ralph-wiggum/hooks/hooks.json`이 `Stop` 훅 등록 → `hooks/stop-hook.sh`가 `.claude/ralph-loop.local.md`의 iteration을 증가시키고 `{"decision":"block","reason":$prompt}`로 **턴을 되살림**. 선언형 강제는 `plugins/hookify/hooks/stop.py`
- **병렬화 = `Workflow` tool (8개 중 유일한 진짜 오케스트레이션 엔진)**: `d.ts:12574-12588` — `script`/`name`/`args`/`scriptPath`/`resumeFromRunId`. DSL은 `agent()/parallel()/pipeline()/phase()` + `export const meta = {name, description, phases}`. **`resumeFromRunId`는 (prompt, opts)가 같으면 캐시된 결과를 즉시 반환** — 부분 재실행 DAG. `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` 1–256 (`CHANGELOG.md:1510`), prefix stagger (`:2747`), `.claude/workflows/`, `workflow-authoring` 스킬
- **상태**: `TodoWrite` = `mods/diff/hooks/tools/todo-tool.ts` `export const TODO_TOOL = 'TodoWrite'`. 단 **이 빌드에서는 모델/버전 게이트로 꺼짐** — `CHANGELOG.md:2662` "no longer available on Opus 4.8, Sonnet 5 … set `CLAUDE_CODE_ENABLE_TODO_TOOLS=1`". 실제로 타입 표면엔 `TaskOutput`/`TaskStop`만 있고 `TaskCreate` 없음 (`d.ts:12374-12598`). **훅 33종** (`d.ts:4288` 유니온) — codex 12종 대비 `PostToolBatch`·`TaskCreated`·`TeammateIdle`·`FileChanged` 등
- **위임**: `Agent` 툴 `run_in_background`, `isolation: worktree|remote`, `ListAgents`/`SendMessage`, agent teams/teammates, fork 서브에이전트 (`CHANGELOG.md:2668`)
- **재개**: `SessionResume{source: startup|resume|clear|compact|fork}`, **워크플로 수준** `resumeFromRunId` + `scriptPath` 편집-재실행

## openclaw

- **계획 모드 없음**. ACP `session/set_mode("plan")` passthrough (`packages/acp-core/src/types.ts:61`)뿐이고, Codex `collaborationMode`는 `extensions/codex/src/app-server/turn-params.ts:240`에서 **`mode: "default"`로 하드코딩**. 소유하는 건 *permission* mode — `docs/tools/permission-modes.md:40` `tools.exec.mode` ∈ `deny|allowlist|ask|auto|full`
- **루프는 "감소적"**: `docs/tools/goal.md:151` "The model cannot silently pause, resume, clear, or replace a goal" + 매 턴 주입되는 목표 한 줄 (:166-172) — **모델 재량**이며 판정기 루프가 아님. 반면 `src/agents/tool-loop-detection.ts:496` `GLOBAL_CIRCUIT_BREAKER_THRESHOLD`는 **루프를 끊는** 서킷브레이커 (outcome-hash 기반, `:54` `UNKNOWN_TOOL_THRESHOLD = 10`). `/loop` 없음, boulder 없음
- **병렬화 = DSL 거부 선언**: `docs/tools/swarm.md:17-19` **"There is no graph DSL and no separate workflow format. The program is the orchestration."** (`:616-619` 재확인). 실행은 `Promise.allSettled`/`while`/`Promise.race`로 표현하고, 하부는 `src/agents/subagents/swarm/swarm-scheduler.ts:239`의 **FIFO 레인** (`swarm-config.ts:14-21`: maxConcurrent 32, maxChildrenPerGroup 50, maxTotalPerGroup 200, waitTimeoutSecondsMax 600). 준비 상태 게이트는 별도 확장 `extensions/workboard/src/tools.ts:181-190` `workboard_link` ("child becomes ready only after parents are done") + `:156` `parents: string[]`
- **상태 = `progress_card`**: `docs/tools/progress-card.md:10` "the single agent status tool for a session", `:32` plan ≤50단계·`in_progress` ≤1, `:56` "Every call is a replacement, not a patch". `TodoWrite`는 네이티브 로직이 아니라 `packages/ai/src/providers/anthropic-tool-projection.ts:48` 통과뿐. 미완료 체크리스트의 self-check는 프롬프트 텍스트에만 의존 (`:60-65`)
- **위임**: `sessions_spawn` (통지형) vs `collect: true` 수집기 — `src/agents/openclaw-tools.swarm.ts:12` `COLLECTOR_WITHHELD_TOOL_NAMES`로 수집기에게 `ask_user`/`sessions_send`/`sessions_yield`를 회수, `:86-98` 쓰기 권한을 **기록 시점이 아닌 실행 시점에 재검증**. 채택은 `src/agents/spawn-plan.ts:261` `resolveSpawnAdmission` (depth/캡/`allowAgents`), 역할은 `:283` `orchestrator|leaf`. 팀 프리셋 `docs/concepts/multi-agent.md:112`
- **재개 = `sessions_yield`**: `src/agents/tools/sessions-yield-tool.ts:83` (name), `:87-89` description "End this turn for pending child completion events; **this is not a final-result submission**" → `src/agents/embedded-agent-runner/run/attempt-sessions-yield.ts` `SESSIONS_YIELD_ABORT_REASON = { code:"sessions_yield", turnHandoff:true }`로 **abort를 핸드오프로 위장**해 러너가 `<turn_aborted>` 가이드를 건너뜀. **Stop 훅 없음** — `docs/automation/hooks/event-types.md:22-36` 전체 목록에 continuation을 주입할 이벤트가 없음

## hermes-agent

- **계획 모드 = 단일 턴 프롬프트**: `hermes_cli/commands.py:164` `CommandDef("plan", "Write a markdown implementation plan to .hermes/plans/ without executing anything")`. `agent/plan_prompt.py:11-26` `_PLAN_MODE_RULES` "Do not edit project files except the plan markdown file itself" — **샌드박스·승인 게이트 없음**, 산출물만 `.hermes/plans/*.md`
- **루프 = Ralph 원본**: `hermes_cli/goals.py:1` "Persistent session goals — the Ralph loop for Hermes". `judge_goal():893`가 6-way verdict (`done|blocked|continue|wait{on_session,on_pid,for_seconds}`) 반환, `evaluate_after_turn():1479`가 턴마다 판정. **판정기 앞 결정적 게이트** `GoalGate/run_gate():344,396` (실패 시 판정기 호출 안 함), **완료 계약 5필드** `GoalContract` (`outcome, verification, constraints, boundaries, stop_when`, :265). **fail-open** + `goals.max_turns: 20` 백스톱 (`website/docs/user-guide/features/goals.md:193,197`). 별도로 `/loop` (`hermes_cli/loops.py:386`, `LOOP_COMPLETE` 센티널, `--until`에 **판정기 재사용** :568, self-paced 60s→15min)과 `/heartbeat` (`heartbeat.py:149`, 유휴 세션에만 발사)
- **병렬화 = 진짜 DAG**: `hermes_cli/kanban_db.py:1050` `CREATE TABLE task_links` (parent→child), 종속 게이트 `:2271` "Return whether every direct parent is terminal". 사이클 검출 `kanban_db_graph.py:87` `raise ValueError("cyclic dependency detected…")`. **LLM이 인덱스 기반 의존성을 직접 작성** — `kanban_decompose.py:65-70` "'parents' is a list of INDICES … Tasks with no parents run in **PARALLEL**". 고정 3단 swarm `kanban_swarm.py:165` root → 병렬 worker → **모든 worker 대기 verifier** → synthesizer, 블랙보드 `:26` `BLACKBOARD_PREFIX = "[swarm:blackboard] "`. 메모리 인지 동시성 `kanban_db_dispatch.py:1835` `derive_default_max_in_progress`
- **상태 = 코드로 강제되는 stop gate**: `tools/todo_tool.py:1` revisioned `todo_list` (`MAX_TODO_ITEMS = 256`, `:21` `TODO_INJECTION_HEADER = "[Your active task list was preserved across context compression]"`). **14개 `kanban_*` 툴**이 크로스 프로세스 체크리스트. `agent/turn_stop_gates.py:105` `apply_stop_gates`가 3개 게이트(verify → plugin hook → kanban guard)를 순서대로 돌리며 **합성 nudge로 턴을 재진입** — `agent/kanban_stop.py:64` `build_kanban_stop_nudge` (:87-97 "Do not stop without calling one of them")
- **위임 = OS 프로세스**: `website/docs/user-guide/features/kanban.md:11` "every worker is a full OS process with its own identity". 디스패처가 `subprocess.Popen(..., start_new_session=True)` + 고정 env(`HERMES_KANBAN_*`)로 띄우고, `agent/delegation_context.py:122,141`가 env scrub/경로 펜스. 재촉은 `agent/subagent_lifecycle.py:237` `SubagentLifecycleService{launch,wait,cancel,reconnect}`. 좀비/고아/재시작 서킷브레이커 스윕 `kanban_db_dispatch.py:281,504,845,1525`
- **재개**: 상태 전부 `SessionDB.state_meta` (`goals.md:219` "/resume picks up right where you left off"), 세션 회전 시 `goals.py:679` `migrate_goal_to_session`. 틱은 **crash-safe 예약** `loops.py:501` (중도 사망 시 tight loop 방지용 provisional `next_due_at`). 유저 메시지가 continuation을 **선점** (`goals.md:207`, `_pending_input` / adapter FIFO)

## 관찰

1. **연속 실행은 3계보로 갈린다.** (a) **판정기 루프** — hermes `/goal`이 원본이고 codex `ext/goal`이 같은 계보 (판정기 대신 자체 프롬프트 audit로 대체). (b) **Stop-훅 block** — claude-code ralph 플러그인·hookify, codex `ClaudeHooksEngine::Stop{should_block}`, oh-my-pi `session_stop`. (c) **명시적 거부** — openclaw는 서킷브레이커만, opencode·pi-mono는 아예 없음. boulder라는 이름은 **OmO에만** 존재하고 claude-code·codex 트리에는 0건.
2. **계획 모드의 실질 강도는 "코드 게이트 유무"로만 판정된다.** 강한 순: oh-my-pi(settle 강제 루프) ≈ claude-code(`plan` = 도구 실행 없음, Bash 정적 분석) > opencode(permission deny) > OmO(md-only 훅) > codex(`update_plan` 1건 차단, 나머지 프롬프트) > pi-mono·hermes(프롬프트만) > openclaw(없음).
3. **DAG는 4종류의 구현 철학으로 나뉜다.** pi-mono = durable commit에서 파생되는 **join 정책 그래프**; OmO = WAL+journal+복구를 가진 **운영체제형 스케줄러**; hermes = **SQLite 행**으로 선언하고 LLM이 인덱스를 써먹는 그래프; claude-code = **스크립트 DSL + 결과 캐시 재개**. 반대로 openclaw는 "graph DSL 없음"을 명시 선언하고, codex는 tree만, oh-my-pi는 세마포어 pool만, opencode는 없다.
4. **todo를 연속 실행의 "연료"로 쓰는 하니스만 장기 실행이 된다.** oh-my-pi `TodoTracker.checkCompletion`, OmO `todo-continuation-enforcer`, hermes `turn_stop_gates`는 미완료 상태를 턴 종료 조건으로 삼는다. opencode·codex·claude-code·openclaw는 todo/plan을 **기록만** 하며 재실행 주체는 인간이다.
5. **위임의 격리 단위가 곧 스케일 체계다.** 세션(openclaw, opencode, OmO) = 한 프로세스 안 안전한 병렬; in-process 테스크(pi-mono, oh-my-pi) = durable 상태 + worktree; OS 프로세스(hermes kanban) = 각 카드가 자격증명·env 펜스를 가진 독립 워커; 스레드 스포 codex) = 모델에게 슬롯 수를 일러주는 소프트 제약.
