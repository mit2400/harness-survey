# 에이전트/서브에이전트 비교

## 요약

| 하니스 | 내장 에이전트 | 커스텀 정의 | 멀티에이전트 |
|---|---|---|---|
| opencode | 7 (build/plan/general/explore/compaction/title/summary) | `{agent,agents}/**/*.md` | task 툴로 subagent spawn |
| oh-my-openagent | **11** (Sisyphus/Hephaestus/Prometheus/Atlas/Oracle/Librarian/Explore/Metis/Momus/Multimodal-Looker/Sisyphus-Junior) | agentSources 팩토리 | 9 delegation 카테고리, team mode |
| pi-mono | **없음** (의도적) | 예제만 (`examples/extensions/subagent/`) | experimental durable subagent |
| oh-my-pi | 5 (scout/reviewer/security-reviewer/task/sonic) | `~/.omp/agent/agents/*.md`, `.omp/agents/*.md` | task 툴, advisor, vibe_spawn |
| codex | Default/Plan 모드 | agent-roles `.toml` (재귀 발견) | multi-agent V1/V2 |
| claude-code | 5 (general-purpose/Explore/Plan/claude/fork) | `.claude/agents/*.md`, `--agents` JSON | agent teams, worktree 격리 |
| openclaw | harness 5 (openclaw/codex/copilot/acpx/agentsapi) | agentDir별 격리 | subagents/swarm, system-agent |
| hermes-agent | delegate_task(leaf/orchestrator) | delegation config | kanban swarm, MoA, hosted rooms |

## opencode
`packages/opencode/src/agent/agent.ts`:
| agent | mode | 비고 |
|---|---|---|
| build | primary | **기본**, 전 툴 허용 |
| plan | primary | edit 거부(plans/*.md만), plan_exit 허용 |
| general | subagent | todowrite 제외 전 툴 |
| explore | subagent | 읽기 전용 (grep/glob/read/webfetch/websearch) |
| compaction | primary(hidden) | `*` deny |
| title | primary(hidden) | temp 0.5 |
| summary | primary(hidden) | |
- 커스텀: `{agent,agents}/**/*.md` YAML frontmatter, legacy `{mode,modes}/*.md`
- subagent는 task 툴 description에 런타임 주입, `subagent_depth`로 깊이 제한
- 참고: docs의 `scout`는 소스에 없음 (docs drift)

## oh-my-openagent
`packages/omo-opencode/src/agents/AGENTS.md` — 11개:
| agent | model | mode | 역할 |
|---|---|---|---|
| Sisyphus | claude-opus-5-5 max | primary | 메인 오케스트레이터 |
| Hephaestus | gpt-5.6-sol medium | primary | 자율 deep worker (GPT 전용) |
| Prometheus | claude-fable-5 xhigh | primary | 전략 플래너 (.md만 쓰기, hook 강제) |
| Atlas | claude-sonnet-5 | primary | todo-list 오케스트레이터 |
| Oracle | gpt-5.6-sol xhigh | subagent | 읽기 전용 컨설턴트 |
| Librarian | gpt-6-luna-fast | subagent | 외부 문서/코드 검색 |
| Explore | gpt-6-luna-fast | subagent | contextual grep |
| Metis | claude-fable-5-1 max | subagent | pre-planning 컨설턴트 |
| Momus | gpt-5.6-terra high | subagent | plan 리뷰어 |
| Multimodal-Looker | gpt-5.6-sol low | subagent | PDF/이미지 분석 |
| Sisyphus-Junior | claude-sonnet-5 | subagent | 카테고리 spawn executor |
- 9 delegation 카테고리: visual-engineering, architect, ultrabrain, deep-low, deep-high, artistry, quick, unspecified-low, unspecified-high, writing (deep-low→deep-high 에스컬레이션 레인)
- 툴 제한: Oracle/Librarian/Explore는 write/edit/task 거부, Multimodal-Looker는 read만, Atlas는 task 거부
- Team Mode (실험적, 기본 off): lead + 최대 8 멤버, mailbox, 공유 task list, 멤버별 worktree, tmux 레이아웃
- 프롬프트는 `dynamic-agent-prompt-builder.ts`가 런타임 조합

## pi-mono
- **내장 서브에이전트 없음** — README 명시: "Pi ships with powerful defaults but skips features like sub-agents and plan mode"
- experimental: `src/experimental/durable/subagent.ts` — child conversation 생성, 자기 자신을 child에서 제거(무한 재귀 방지), `replay: "safe"`
- `src/experimental/vacation/vacation.ts` — Vacation 플래너 + Search 서브에이전트
- 예제: `examples/extensions/subagent/`, `plan-mode/`

## oh-my-pi
`packages/coding-agent/src/task/agents.ts` — 5개 내장:
| agent | 비고 |
|---|---|
| scout | readSummarize:false |
| reviewer | |
| security-reviewer | |
| task | general purpose, `spawns: "*"`, model `@task` |
| sonic | task.md 본문, model `@smol`, 기계적 업데이트만 |
- Frontmatter: name, description, systemPrompt(필수), tools, spawns, model(우선순위 리스트), thinkingLevel, output, blocking, autoloadSkills, readSummarize, prewalk, advisor
- 발견: `~/.omp/agent/agents/*.md` + `.omp/agents/*.md` + extension packages + Claude marketplace, 내장은 마지막에 append
- **advisor**: 두 번째 모델이 매 턴 읽고 인라인 노트 주입 (quiet aside/concern/blocker)
- **prewalk**: 서브에이전트가 자기 모델로 시작→첫 edit/write에서 smol로 핸드오프
- Agent Hub (Alt+A): 라이브 로스터, transcript 읽기, steer, revive
- `vibe_spawn`: fast→sonic, good→task 티어 라우팅

## codex
- Collaboration modes: `ModeKind::{Default, Plan}`, 템플릿 `collaboration-mode-templates/templates/{default,plan}.md`. Plan만 request_user_input 허용
- Multi-agent V1: spawn_agent, send_input, resume_agent, wait_agent, close_agent (depth 제한)
- Multi-agent V2: spawn_agent, send_message, followup_task, wait_agent, interrupt_agent, list_agents. `multi_agent_v2` Stable이지만 기본 off, `multi_agent`는 기본 on
- Agent roles: `.toml` 재귀 발견 (`$CODEX_HOME/agents/` + `[agents]` 테이블), name/description/nickname_candidates + flattened ConfigToml
- Agent identity: Ed25519 + JWT (`agent-identity/`)
- 인프라: agent-graph-store, agent-message-board-client, core/src/agent/

## claude-code
- 내장: `general-purpose`, `Explore`(Haiku, 세션 모델 상속), `Plan`, `claude`, `fork`(기본 on, 전체 대화+prompt cache 상속)
- 커스텀: `.claude/agents/*.md`, `~/.claude/agents/*.md`, frontmatter(name, description, model, color, tools, disallowedTools, permissionMode, effort, omitClaudeMd, context: fork), `--agents` JSON
- `/agents` 위저드는 제거됨 (Claude에게 물어보는 방식으로)
- 서브에이전트 worktree 격리: `worktree.baseRef`, `worktree.sparsePaths`, `worktree.bgIsolation`
- **Agent teams**: teammates + SendMessage/ListAgents, TaskCreated/TaskCompleted/TeammateIdle 훅, "세션은 단일 implicit team"
- 비대화형 spawn은 기본 백그라운드
- 번들 플러그인 에이전트 16개: code-explorer, code-architect, code-reviewer, agent-creator, plugin-validator, skill-reviewer, conversation-analyzer, agent-sdk-verifier-{ts,py}, pr-review-toolkit 6개

## openclaw
- **Embedded harnesses** (`api.registerAgentHarness`): openclaw(내장), codex, copilot, acpx(ACP), agentsapi
- CLI backends: `agentRuntime.id: "claude-cli"` 등 — 로컬 CLI 프로세스로 실행
- 에이전트 격리: agent별 workspace/agentDir/auth profiles/model registry/`openclaw-agent.sqlite`. 기본 id `main`
- Subagents: `src/agents/subagents/` (registry/spawn/completion/announce/swarm), 툴: subagents, sessions_spawn, agents_wait(swarm 게이트), sessions_yield
- System agent (`src/system-agent/`): assistant.ts, chat-engine.ts, chat-turn-router.ts, approval-intent.ts

## hermes-agent
- **delegate_task**: child AIAgent spawn, isolated context, 자체 terminal session, 제한된 toolset. 단일 goal+context 또는 병렬 tasks 배치(기본 3 동시, 최대 10)
- 역할: `role="leaf"`(기본, delegate_task/clarify/memory/send_message/cronjob 차단), `role="orchestrator"`(delegation toolset 유지)
- `delegation.max_spawn_depth` 기본 1 (flat), `orchestrator_enabled` 킬 스위치
- child는 부모 지식 0으로 시작 (예외: workspace 컨텍스트 파일 임베드, SOUL.md 제외)
- child별 model/provider override: `delegation.model/provider/base_url/api_key/api_mode/fallback_providers`
- `output_schema`: JSON Schema 계약, 실패 시 1회 bounded correction turn
- 라이브 컨트롤 플레인: `delegate_tool_registry.py` (list/steer/stop, spawn pause)
- async/durable: `async_delegation.py`, `async_delegations` 테이블
- **subagent_worktree**: child별 git worktree
- **Kanban swarm**: `hermes kanban` — 카드별 worker 프로세스+세션+workspace, kanban.db 7 테이블, orchestrator 전용 툴은 worker에 숨김
- **Mixture of Agents**: virtual provider, reference 모델이 조언, aggregator가 실행/과금
- **Hosted rooms**: 멀티에이전트 그룹 채팅
