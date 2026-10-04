# 하니스별 독특한 기능

## opencode
1. **Code Mode** (`packages/codemode/` + `tool/code-mode.ts`, id `execute`) — 모델이 MCP 툴을 오케스트레이션하는 스크립트를 작성, 컨텍스트 절약
2. **References** (`packages/core/src/reference.ts`) — 로컬 디렉토리 + git repo를 alias로 첨부, repo는 `data/repos/`에 자동 clone/branch pin
3. **17개 provider별 시스템 프롬프트** (`session/prompt/*.txt`) — anthropic/gpt/codex/gemini/kimi/beast/trinity/meta 등
4. **System Context algebra** (`packages/core/src/system-context/`) — baseline/update diff 가능한 컨텍스트 소스
5. **Git-backed snapshots** — 매 턴을 shadow gitdir(`data/snapshot/<project>/<hash>`)에 커밋, 메시지별 revert
6. **Durable prompt inbox + event-sourced sessions (V2)** — `session_input` 테이블, steer-vs-queue, context epoch
7. **Native OpenAI Codex WebSocket transport** (`plugin/openai/ws.ts`)
8. **Hosted websearch** — Exa/Parallel MCP를 키 없이 프록시
9. **Experimental policies** — permissions 위의 allow/deny 레이어
10. **V1/V2 이중 아키텍처** — Effect-native, 명시적 마이그레이션 계약

## oh-my-openagent (OmO)
1. **`ulw`/`mass ulw` 키워드 시스템** — IntentGate가 감지, mass ulw는 dependency-frontier DAG(`packages/senpi-task/src/dag/`, ~48파일)로 실행
2. **Boulder** — work-tracking 상태머신, todo continuation enforcer가 구동
3. **Kibitzer 메모리 사이드카** — 작은 모델이 BM25 recall로 메인 에이전트 옆에서 넛지
4. **prompt-async-gate** — 내부 session.prompt의 유일한 공식 경로, 정적 audit 테스트로 강제
5. **Team Mode** — lead + 8 멤버, mailbox, 공유 task list, 멤버별 worktree, tmux 레이아웃
6. **Hashline edit** — LINE#ID 해시 앵커, stale patch 거부
7. **Goal** (ralph-loop 대체) — session.idle마다 continuation 재주입, `create_goal/update_goal/get_goal`
8. **2개 독립 fallback** — model-fallback(proactive) vs runtime-fallback(reactive), 의도적 비통합
9. **Session notification** — OS별 알림 (linux/macos/windows)
10. **btw-side** — 임시 사이드 대화, **Herdr** — DAG 시각화, **omo thread** — 스크립트/커넥터 SDK
11. **cmux 감지** — 실제 tmux 없이 tmux pane 동작

## pi-mono
1. **codemode** — QuickJS WASM 샌드박스에서 JS 실행, `tools.<name>()` 호출, 출력만 모델에 도달. `store()`/`load()`는 세션 엔트리로 영속 (branch-aware)
2. **tool_search** — BM25로 미선언 툴 검색, 다음 모델 호출에만 선언, transcript에 기록
3. **권한 시스템 없음** — 명시적 설계 입장, 대신 3가지 컨테이너화 패턴 문서화 (gondolin/docker/openshell)
4. **Prompt-cache-aware 아키텍처** — mcp_servers 섹션 append(변이 아님), 툴/프롬프트 변경도 transcript delta로, summarization은 cache write 비활성화
5. **세션 트리 + branch-aware 상태** — `/tree` `/fork` `/cross` 구분, store() 값/로드된 툴/선언이 전부 branch-scoped
6. **Contiguity-aware compaction** — tool call과 result 사이 안 자름, recovery의 context_edit 생략 추적
7. **Runtime-replaceable 코어** — codemode/tool_search/mcp가 `replaceable: true`
8. **5개 인터페이스** — TUI/print/JSON/RPC(JSONL stdio)/in-process SDK + pi-client/pi-protocol (CBOR 원격 세션)
9. **AGENTS.override.md + worktree shadowing** — 중첩 worktree에서 메인 repo 컨텍스트 파일 억제

## oh-my-pi
1. **Snapcompact** — 히스토리를 모델별 PNG 프레임으로 아카이브 (빌링 수식 기반 프레임 크기, 로컬 전용)
2. **TTSR (Time-Traveling Stream Rules)** — regex 매치 시 스트림 중간 토큰에서 abort, 규칙 주입, 같은 지점부터 재시도. compaction 생존
3. **Hashline editing** — 콘텐츠 해시 앵커, stale patch 거부 (Grok 4 Fast에서 출력 토큰 -61% 주장)
4. **16개 내부 URL 스킴** — `pr://`, `issue://`, `agent://`, `skill://`, `ssh://`, `mcp://`, `memory://`, `history://`, `conflict://`, `rule://`, `vault://`, `security://`, `proc://`, `artifact://`, `cfg://`, `omp://`, `xd://`. `write conflict://*`로 머지 일괄 해결
5. **eval 툴 역호출** — Python/JS 커널이 루프백 브리지로 에이전트 툴 호출
6. **16개 외부 config 포맷 네이티브 인식** — Cursor MDC, Cline, Codex, Copilot, Claude, Gemini, OpenCode, VS Code, Windsurf (importer 없음)
7. **Advisor** — 두 번째 모델이 매 턴 리뷰, 인라인 노트 주입
8. **Prevision-threshold speculative compaction** — threshold 직전에 백그라운드 요약, 교차 시 즉시 splice
9. **~80k LoC Rust 내장** — ripgrep, glob, find, brush bash, 46+ coreutils, fork/exec 없음
10. **60+ provider / ~1000 모델**, KDL 모델 정책 (`packages/catalog/src/compat/rules/`)
11. **Collab** — `/collab`으로 라이브 세션 릴레이, QR 링크, 클라이언트 측 프레임 암호화
12. **Vibe mode** — director가 fast/good 워커 구동
13. **Magic keywords** — ultrathink/orchestrate/workflowz (코드 스팬/XML/경로에선 스킵)

## codex
1. **Plugins + marketplaces** (`codex-rs/core-plugins/`) — git/npm/remote marketplace, curated OpenAI, plugin.json, plugin-sourced skills/MCP/hooks/apps
2. **Code Mode** (`codex-rs/code-mode/`, `v8-poc/`) — JS/TS로 툴 호출, namespaces + tool_search
3. **Remote/containerized execution** (`codex-rs/exec-server/`) — 원격 FS/프로세스 스트리밍, sandboxed FS/process, cloud tasks
4. **Guardian** (`core/src/guardian/`) — LLM 기반 승인 리뷰어가 rule-based 승인 대체/보강
5. **Worktrees** (`codex-rs/worktree/`) — git worktree 격리 에이전트 실행
6. **External agent migration** (`codex-rs/external-agent-migration/`) — Claude Code/Cursor config/hooks/subagents/memory 임포트
7. **Shell snapshot** — 턴 간 shell env/venv 보존
8. **Unified exec** — resumable PTY 스타일 실행, zsh fork/execve-wrapper 프로세스 재사용
9. **Sandbox matrix** — Seatbelt(macOS)/Landlock+seccomp(Linux)/Windows sandbox service/mxc-sandbox/bwrap
10. **Requirements/policy plane** — `requirements.toml` + execpolicy로 admin이 모델/승인/sandbox/MCP/hook 제약 강제
11. **157 feature flag** — stage(Stable/Experimental/...) + default_enabled 명시
12. **Realtime** — WebRTC/WebSocket 대화 모드
13. **Memories 2-phase 파이프라인** — rollout 추출→전역 통합, secret redaction

## claude-code
1. **Mods** — 바이너리 내장 TS 훅 모듈 4개 (sec-default, diff, telemetry, agents-md), noun contract, `claude plugin test`
2. **Auto 권한 모드 기본** — 로컬 LLM 안전 분류기가 툴 호출 리뷰, 못 정하면 프롬프트로 degrade. bypassPermissions는 project/local에서 무시
3. **풍부한 네이티브 샌드박스** — bubblewrap/socat, unix-socket+Mach lookup+local-binding+domain allowlist, credential file injection, nested sandbox weakening, cgroups 메모리 제한
4. **Agent teams** — teammates + worktree 격리 + team-aware 훅
5. **Checkpoints / `/rewind`** — 코드+대화 복원, 디스크 바운드 백업
6. **배포 서피스** — cloud sessions, Cowork, Remote Control(폰/Drive 릴레이), self-hosted runner, VS Code/JetBrains/Desktop/Chrome/Gateway
7. **`--bare`/`--safe-mode`/`--restricted`/`CLAUDE_CODE_SIMPLE`** — 훅/LSP/플러그인/스킬 walk/auto-memory 스킵 (~14% 빠름)
8. **Effort levels + `ultracode`** — 모델별 orthogonal 토글
9. **플러그인 생태계** — marketplace(git/SSH/HTTPS), `plugin install --config`, `plugin validate`, `plugin test`, `plugin dev`, hot reload, `.mcpb` 번들, `${user_config.*}` 치환
10. **UI 확장** — `ui.render` JSX-like, Raster/Image blits(60-120fps), panes, dialogs, inputs
11. **Hookify** — 평문 마크다운 룰에서 실제 훅 생성 (대화 transcript 분석)
12. **Ralph Wiggum** — Stop 훅이 같은 프롬프트 재주입, 자기참조 반복
13. **Gateway 헤더** — `x-claude-code-request-class`, `-agent-type`, `-compaction`, `x-claude-code-context-compacted`

## openclaw
1. **Dreaming** — light/deep/REM 3단계 백그라운드 메모리 통합, `DREAMS.md` 일기, 기본 on
2. **Provenance-gated memory tiers** — 쓰기 시점 구조적 게이팅 (사후 탐지 아님)
3. **SQLite-everything + worker-thread discipline** — 새 JSON/JSONL sidecar 금지, DB는 worker thread, Gateway 메인 스레드 금지
4. **Code Mode + Tool Search** — 숨겨진 툴 카탈로그에 대한 compact JS 프로그램, `tool_search`/`tool_describe`/`tool_call`
5. **Agent Attachment / session synchronization** — Gateway 소유 세션을 Control UI/터미널/외부 하니스에서 이어가기
6. **Swarm** — 동시 에이전트 오케스트레이션, `agents_wait` 게이트
7. **Managed worktrees** — git worktree 기반 per-task 워크스페이스, 스킬 소스 핀닝
8. **ACP + A2A** — 외부 에이전트 제어 플레인 (`runtime: "acp"`, `agentId: "codex"`)
9. **Native hook relay** — 네이티브 하니스 승인 대기를 OpenClaw 승인/UI로 브리지
10. **Write-only credential broker** — `secrets` 툴, 에이전트가 값 없이 자격증명 사용
11. **Decision models** — 별도 저렴 모델이 명시적 증거로 판정 (`decision_evaluate`)
12. **ClawHub marketplace** — 플러그인+스킬 레지스트리, provenance/trust policy
13. **Tool-loop detection** — runaway/stalled 에이전트 루프 감지 서브시스템
14. **멀티유저/팀 Gateway** — per-requester verified identity, personal USER.md, team.openclaw.ai
15. **Boards/Trajectory/Snapshot/Link-understanding** — 추가 서브시스템
16. **레거시 연속성** — Warelay→Clawdbot→Moltbot→OpenClaw, `clawdbot.json` 자동 발견+doctor 마이그레이션

## hermes-agent
1. **Frozen-snapshot 메모리** — 세션 시작 시 1회 렌더링, prefix cache 보존, 절대 auto-compact 안 함
2. **프롬프트 인젝션 스캐너** — repo-planted AGENTS.md 차단, SOUL.md는 경고만, 비-git 디렉토리는 부모 미참조
3. **`/goal`** — Ralph-loop 엔진, 매 턴 후 judge 모델이 done/continue/blocked 판정, `goals.max_turns: 20`
4. **`/loop`** — 타이머 기반 자기주도 재실행, 응답 변화 없으면 지수 백오프
5. **Tool Search** — `tool_search`/`tool_describe`/`tool_call` 3 브리지로 MCP+비코어 툴 지연 로딩
6. **Shell hooks** — 임의 언어 subprocess 훅, 툴 호출 차단 + 다음 LLM 턴에 컨텍스트 주입, JSON stdin/stdout, Cursor/Claude Code 호환 `failClosed`
7. **Outbound webhooks** — HMAC 서명 + `profile` 필드, 다른 Hermes 인스턴스 깨우기
8. **`execute_code` + 영속 Python 커널** — 멀티스텝 워크플로를 1 LLM 턴으로, 커널 상태 유지
9. **`output_schema` on delegation** — 1회 bounded correction, 실패 시 raw 답변 보존
10. **Subagent git worktrees** + background-process ownership (child 프로세스는 child와 함께 사망)
11. **Skill curator** — active→stale(14d)→archived(30d), LLM umbrella 통합은 opt-in, content-addressed ledger + single-edit rollback, 삭제 대신 아카이브
12. **`.no-bundled-skills` 마커** — `hermes update` 생존, `--remove`는 byte-identical 미수정 스킬만 삭제
13. **Kanban swarm** — 카드별 worker 프로세스+세션+workspace, kanban.db 7 테이블
14. **Mixture of Agents** — virtual provider, reference 모델 조언 + aggregator 실행/과금
15. **Hosted rooms** — 멀티에이전트 그룹 채팅
16. **22개 플랫폼 어댑터** — telegram/discord/slack/feishu/wecom/line/matrix/sms/irc/a2a 등
17. **Petdex mascot engine** + CLI skin/theme engine
