# hermes-agent

> The self-improving AI agent built by Nous Research. (`NousResearch/hermes-agent`, 251k stars)

## 개요
Python 모노레포 (`pyproject.toml`, `uv.lock`), 33개 톱레벨 디렉토리. **41 toolset, 번들 스킬 58개 + 옵셔널 152개**, 22개 플랫폼 어댑터를 단일 gateway 프로세스에서 구동. 내장 학습 루프(스킬 자동 생성/개선, FTS5 세션 검색, 주기적 nudge)가 차별점. 공식 문서는 `website/docs/` (getting-started, user-guide, guides, integrations, reference, developer-guide).

## 디렉토리 구조
```
agent/                 # 에이전트 런타임 (context_compressor, shell_hooks, outbound_webhooks, adapters)
hermes_cli/            # CLI (config_defaults, hooks, plugins, kanban, web_server_gateway)
gateway/               # 플랫폼 게이트웨이 (22개 어댑터, delivery, builtin_hooks)
hermes_state*.py       # state.db SQLite 레이어 35+ 모듈 (sessions/messages/wal/fts/...)
tools/                 # 툴 구현 (memory_tool, delegate_tool, kanban_tools, browser_*, mcp_tool_agent)
skills/                # 58 번들 스킬 (카테고리별 디렉토리)
optional-skills/       # 152 옵셔널 스킬
optional-mcps/         # MCP 레시피
plugins/, plugin-catalog/
providers/             # 모델 프로바이더 (Nous Portal, OpenRouter, ...)
cron/                  # 내장 크론 스케줄러
evals/, batch_runner.py, mini_swe_runner.py, trajectory_compressor.py
apps/, ui-tui/, tui_gateway/, acp_adapter/, web/, native/
website/docs/          # 공식 문서 (.mdx)
tests/, tests-js/
```

## 메모리 관리
- **Frozen-snapshot 모델**: `MEMORY.md` (2200자 상한) / `USER.md` (1375자 상한) — 세션 시작 시 스냅샷으로 동결 주입, 세션 중 수정해도 다음 세션까지 반영 안 됨 (`tools/memory_tool.py`, `agent/prompt_builder.py`)
- 외부 메모리 프로바이더 5종: openviking, mem0, holographic, retaindb, byterover
- **state.db**: SQLite 13 테이블 (sessions, messages, FTS5 인덱스 등), `hermes_state_*.py` 35개 모듈이 영역별(messages/sessions/wal/fts/usage/...) 담당
- 세션 검색: FTS5 + LLM summarization (`tools/session_search_tool.py`)
- Compaction: `agent/context_compressor.py`(+`_summary.py`), manual 압축 `agent/conversation_compression_manual.py`, trajectory compressor는 학습용 별도(`trajectory_compressor.py`)

## 스킬
- **번들 58개** (`skills/<category>/<name>/SKILL.md`): apple, autonomous-ai-agents, creative, devops, email, media, note-taking, productivity, research, social-media, software-development, web ... — **옵셔널 152개** 추가 (`optional-skills/`: blockchain, data-science, dogfood, finance, gaming, health, mcp, migration, ml-ops, payments, security, smart-home 등)
- **Skills Hub**: official / clawhub / GitHub / SkillSSH 소스에서 설치, `.no-bundled-skills` 마커로 번들 제외 가능
- **Curator**: 사용 중 스킬을 자동 개선·정돈하는 큐레이터 루프 (내장 학습 루프의 핵심)
- 호환: agentskills.io 오픈 표준

## MCP
- 설정: `config.yaml`의 `mcp_servers` (`cli-config.yaml.example` 참고)
- **65개 Nous recipe** (`optional-mcps/` 등)
- Sampling 기본 on (서버가 LLM 호출 요청 가능)
- **역방향**: `mcp_serve.py`가 Hermes 자신을 MCP 서버로 노출
- 런타임: `tools/mcp_tool_loop.py`, `tools/mcp_tool_agent.py`

## 내장 툴
- **41 toolset** (`toolsets.py`, `toolset_distributions.py`), 기본 `hermes-cli`, 코어 ~60개
- terminal: **8개 백엔드** (local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox 등), Daytona/Modal은 서버리스 퍼시스턴스(hibernation)
- browser: CDP 기반 (`tools/browser_tool*.py`, browser_use, camofox, lightpanda 폴백)
- kanban_*: **12개** kanban 툴 (`tools/kanban_tools.py`)
- 기타: memory, session_search, delegate, execute_code(persistent kernel), mcp, cron, todo 등

## 에이전트
- **delegate_task** (`tools/delegate_tool.py`): leaf/orchestrator 모드, `output_schema`로 구조화 결과 강제, `tools/async_delegation.py`로 비동기 위임
- **Kanban swarm**: kanban 보드 기반 멀티에이전트 분업 (`tools/kanban_*.py`, gateway builtin_hooks, tests/hermes_cli/test_kanban_lifecycle_hooks.py)
- **MoA (Mixture-of-Agents)**: 다중 모델 응답 합성
- **Hosted rooms**: 원격 호스팅 대화방
- Subagent worktree 격리

## 훅/플러그인
4종 분리:
- **(a) Gateway HOOK.yaml** 플러그인 훅
- **(b) Plugin** (`plugins/`, `plugin-catalog/`)
- **(c) Shell subprocess 훅** (`agent/shell_hooks.py`)
- **(d) Outbound webhook** (`agent/outbound_webhooks.py`) — HMAC 서명
- `VALID_HOOKS` 약 **50개** 이벤트 (`hermes_cli/hooks.py`, `agent/shell_hooks.py`, `gateway/builtin_hooks/`)

## 설정
- `~/.hermes/config.yaml` + `.env` (`hermes_cli/config_defaults.py`)
- 톱레벨 키 **99개**, `UPPER_SNAKE` 환경변수 → `.env` 라우팅
- **Profiles**: 프로필별 상태 디렉토리 분리, `hermes_state_profile_repair.py`
- 경로: `~/.hermes/` 아래 workspace, state.db, credentials, logs

## 독특한 기능
- **/goal (Ralph loop)**: 목표 달성까지 반복 실행 루프, `/loop` 명령
- **Tool Search**: 툴 카탈로그 검색으로 컨텍스트 절약
- Shell hooks + outbound webhooks (HMAC)
- `execute_code` + persistent kernel (RPC로 툴 호출, 멀티스텝 파이프라인을 0-context 턴으로 압축)
- Subagent git worktree 격리
- Skill curator (자동 개선)
- Kanban, MoA, hosted rooms
- **22개 플랫폼 어댑터** (Telegram, Discord, Slack, WhatsApp, Signal, CLI ...)
- Petdex (pet 상호작용), skin engine (TUI 테마)
- 7 terminal backends + Daytona/Modal hibernation
- Trajectory 생성/압축으로 학습 데이터 파이프라인 (`batch_runner.py`, `trajectory_compressor.py`)

## 확장 포인트
- Toolset: `toolsets.py` + `toolset_distributions.py`
- 툴 추가: `tools/` + toolset 등록
- 스킬: `skills/`, Hub 소스(official/clawhub/GitHub/SkillSSH)
- 플랫폼: `gateway/` 어댑터 추가
- 훅: `VALID_HOOKS` (`hermes_cli/hooks.py`) + `gateway/builtin_hooks/`
- MCP 레시피: `optional-mcps/`
- 문서: `website/docs/` (developer-guide, reference)
