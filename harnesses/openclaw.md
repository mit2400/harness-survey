# openclaw

> The AI that really does things. Any OS. Any Platform. (`openclaw/openclaw`, 391k stars)
> **분석 기준**: `openclaw/openclaw` · 커밋 `ee9127b7` (2026-10-04) · 버전 `2026.9.8` — 전체 목록은 [VERSIONS.md](../VERSIONS.md)

## 개요
v2026.9.8, MIT. "Multi-channel AI gateway with extensible messaging integrations". TypeScript 모노레포 (`src/`, `packages/`, `apps/`, `crates/`, `extensions/`, `ui/`). **163 번들 플러그인 + 48 번들 스킬**.

## 디렉토리 구조
```
src/
├── memory/            # 메모리 (root-memory-files, memory-runtime)
├── sessions/          # 세션 (~120 파일, lifecycle/transcripts/state events)
├── agents/            # 에이전트 (tool-catalog, harness, subagents, tools/)
├── skills/            # 스킬 (loading, discovery, library, lifecycle)
├── mcp/               # MCP 브리지
├── plugins/           # 플러그인 (~1000 파일, loader/registry/manifest)
├── plugin-sdk/        # 공개 SDK 경계
├── hooks/             # 내부 훅 (bundled 5개)
├── config/            # 설정 (paths, zod-schema, mcp-config)
├── system-agent/      # 내부 operator 에이전트
├── boards/, claws/, trajectory/, decisions/, link-understanding/, snapshot/, proxy-capture/, session-cards/
└── ...
extensions/            # 163 번들 플러그인
skills/                # 48 번들 스킬
custodian-skills/      # 4 custodian 스킬
docs/                  # 방대한 문서
```

## 메모리 관리
- **계층 모델** (`docs/concepts/memory-architecture.md`):
  | Tier | Surface | 작성자 | 주입 |
  |---|---|---|---|
  | Instructions | AGENTS.md + workspace 파일 | Human | 항상, 세션 시작 |
  | Curated core | MEMORY.md, USER.md | Dreaming/직접 요청 | 세션 시작, budgeted |
  | Episodic | memory/YYYY-MM-date.md, transcripts | Agent + memory flush | recall 시에만 |
  | Prospective | Standing intents(SQLite), cron | intent 툴 | 트리거 시에만 |
  | Review | DREAMS.md, dreaming reports | Dreaming | 절대 (사람 읽기용) |
- **Dreaming** (기본 on): light/deep/REM 3단계 백그라운드 통합, `DREAMS.md` + `memory/dreaming/<phase>/` 기록, MEMORY.md로만 승격
- **Active Memory**: 과거 질문 + deterministic recall 실패 시에만 blocking recall 모델
- 엔진: per-agent SQLite, FTS5 BM25 + vector + hybrid, MMR, CJK trigram, sqlite-vec는 별도 read-only 프로세스
- **Provenance 기반 쓰기 게이팅** (사후 탐지 아님)
- 세션: DMs 공유/그룹 격리, cron=fresh, webhook=isolated, `session.scope: "global"`, per-agent `<agentDir>/openclaw-agent.sqlite`
- 컨텍스트 파일: AGENTS.md, SOUL.md, IDENTITY.md, USER.md, BOOTSTRAP.md, MEMORY.md, TOOLS.md

## 스킬
- **48 번들** (`skills/<name>/SKILL.md`): 1password, apple-notes, apple-reminders, bear-notes, blogwatcher, blucli, camsnap, clawhub, coding-agent, control-ui, diagram-maker, eightctl, gemini, gh-issues, gifgrep, github, gog, goplaces, healthcheck, himalaya, mcporter, meme-maker, model-usage, nano-pdf, node-connect, node-inspect-debugger, notion, obsidian, openai-whisper, openai-whisper-api, openhue, oracle, ordercli, peekaboo, pyproject.toml, python-debugpy, sag, sherpa-onnx-tts, skill-creator, songsee, sonoscli, spike, spotify-player, summarize, things-mac, tmux, trello, visualize, weather, xurl
- Custodian 4: add-model-provider, cloud-image-bake, configure-channel, diagnose-gateway
- **7-tier precedence**: workspace/skills > workspace/.agents/skills > ~/.agents/skills > state-dir/skills > state-dir/agents/<id>/workshop-skills > bundled+custodian > extraDirs+plugin
- Frontmatter: `metadata.openclaw.{always, skillKey, primaryEnv, emoji, homepage, os, requires{bins,anyBins,env,config}, install[]}`, install kinds `brew|node|go|uv|download`
- 레지스트리: **ClawHub** (`src/skills/lifecycle/clawhub*.ts`), Workshop(에이전트 제안)
- 발견: `src/skills/loading/bundled-dir.ts` (`OPENCLAW_BUNDLED_SKILLS_DIR` env → bun sibling → `<packageRoot>/skills`)

## MCP
- **양방향**: client + server
- 설정: `mcp.servers.<name>` — Zod (`src/config/zod-schema.mcp-server.ts`)
  - transports: stdio/sse/streamable-http
  - timeouts: connectionTimeoutMs, requestTimeoutMs, supportsParallelToolCalls
  - OAuth: `auth: "oauth"` + `oauth.{identity: shared|per-requester, authProfileId, scope, redirectUrl, clientMetadataUrl}`
  - TLS: sslVerify, clientCert, clientKey
  - toolFilter.{include[], exclude[]}, codex.{agents[], defaultToolsApprovalMode}
- 런타임: `src/mcp/` — channel-bridge(대화를 MCP로), openclaw-tools-serve, plugin-tools-serve, tools-stdio-server
- 복원력: 서버별 지수 백오프 30s→10min
- CLI: `openclaw mcp add|doctor|probe|status|login|serve`
- MCP 툴도 profile + tool-policy 파이프라인 통과

## 내장 툴
**66개** (`src/agents/tool-catalog.ts`), 11 섹션:
- fs: ls, read, write, edit, apply_patch
- runtime: exec, process, code_execution, secrets
- web: web_search, web_fetch, x_search
- memory: memory_search, memory_get, personal_instructions, presence
- sessions: sessions, sessions_list, sessions_history, sessions_search, conversations_list/send/turn, sessions_send/spawn/yield, github_identity_status/publish, agents_wait, subagents, session_status
- ui: suggest_task, dismiss_task, browser, screen, theme, dashboard, terminal, portal, canvas, show_widget
- messaging: message, heartbeat_respond
- automation: gateway, plugins, openclaw, cron(automations)
- nodes: nodes, computer, mobile_ui
- agents: agents_list, get_goal, create_goal, update_goal, progress_card, ask_user
- media: skill_workshop, skills_search, skills_read, view_image, image_generate, music_generate, video_generate, transcripts, tts, pdf
- 기타: decision_evaluate
- 4 프로필: minimal/coding/messaging/full(`["*"]`), onboarding 기본 full
- alias: bash→exec, apply-patch→apply_patch, cron→automations
- 조건부: agents_wait(swarm), personal_instructions(multi-user)

## 에이전트
- **Embedded harnesses** (`api.registerAgentHarness`): openclaw(내장, `builtin-openclaw.ts`), codex(`extensions/codex/`), copilot(`extensions/copilot/`), acpx(ACP, `extensions/acpx/`), agentsapi(`extensions/agentsapi/`)
- CLI backends: `agentRuntime.id: "claude-cli"` 등
- 에이전트 격리: agent별 workspace/agentDir/auth profiles/model registry/`openclaw-agent.sqlite`, 기본 id `main`
- Subagents: `src/agents/subagents/` (registry/spawn/completion/announce/swarm), 툴: subagents, sessions_spawn, agents_wait(swarm 게이트), sessions_yield
- System agent (`src/system-agent/`): assistant.ts, chat-engine.ts, chat-turn-router.ts, approval-intent.ts, hosted-setup.runtime.ts

## 훅/플러그인
3종 분리:
**(a) Internal hooks** — operator 설치 JS/TS + `HOOK.md`, colon 이벤트명 (`command:new`, `agent:bootstrap`), trusted/unsandboxed/in-Gateway
- 엔진: `src/hooks/{discovery,loader,module-loader,config,configured,workspace,fire-and-forget,policy}.ts`
- 5 번들: boot-md, bootstrap-extra-files, command-logger, compaction-notifier, session-memory
- Gmail transport: `src/hooks/gmail.ts`, `gmail-watcher.ts`
- CLI: `openclaw hooks list|info|enable|disable|check`

**(b) Plugin hooks (typed)** — `api.on("hook_name", handler)`, ~40개:
- Agent/model: before_model_resolve, agent_turn_prepare, before_prompt_build, before_agent_run, before_agent_reply, before_agent_finalize, agent_end, heartbeat_prompt_contribution, model_call_started/ended, llm_input, llm_output
- Tools: before_tool_call, after_tool_call, resolve_exec_env, tool_result_persist, before_message_write
- Messages: inbound_claim, channel_pairing_requested, message_received, message_sending, reply_payload_sending, message_sent, before_dispatch, reply_dispatch
- Sessions: session_start/end, before_compaction, after_compaction, before_reset
- Gateway/plugins: gateway_start/stop, cron_reconciled, cron_changed, before_install, skill_proposal_evaluate, skill_proposal_changed, skill_changed
- 훅별 타임아웃/실패 정책 (before_agent_run/before_tool_call/before_install 15s fail-closed, skill_proposal_evaluate 120s)

**(c) HTTP webhooks** — `hooks.*` config, 기본 `enabled: false`, `path: "/hooks"`, `token` 필수, `allowedAgentIds`, `presets: ["gmail"]`, `mappings[]`

플러그인 시스템: `openclaw.plugin.json` 매니페스트, loader/registry, ClawHub/npm/git/local 설치, 163 번들, capability ladder (`src/plugin-sdk/AGENTS.md`)

## 설정
- `~/.openclaw/openclaw.json` (JSON/JSON5), `$include` 디렉티브
- 경로: `OPENCLAW_CONFIG_PATH`, state dir `OPENCLAW_STATE_DIR`(기본 `~/.openclaw`), `OPENCLAW_HOME`
- Legacy: `clawdbot.json`, legacy state dirs 자동 발견
- 프로필: `OPENCLAW_PROFILE` → `~/.openclaw-<profile>/`, gateway port 18789 기본, 프로필별 20000-40000 해시
- 주요 경로: workspace `~/.openclaw/workspace`, agent dir `~/.openclaw/agents/<agentId>/agent`, sessions DB `<agentDir>/openclaw-agent.sqlite`, credentials `<stateDir>/credentials`, sandboxes `~/.openclaw/sandboxes`, logs `~/.openclaw/logs/`
- Config domains: channels, agents/sessions/messages, worktreeRoot, tools, models, mcp, skills, plugins, browser, ui, desktop, gateway, cloudWorkers, hooks, canvas, discovery, env, secrets, auth storage, audit, logging, diagnostics, telemetry, update, acp, wizard, identity, automations, memory, meta, experimental, desktop.host
- 스키마: `src/config/zod-schema.root-shape.ts`(613줄), `src/config/types.*.ts`, `schema.help.*.ts`
- **정책**: long-lived alias 없음, breaking change는 반드시 `doctor` 마이그레이션

## 독특한 기능
- Dreaming, Provenance-gated memory tiers, SQLite-everything+worker-thread discipline, Code Mode+Tool Search, Agent Attachment/session synchronization, Swarm, Managed worktrees, ACP+A2A, Native hook relay, Write-only credential broker(`secrets` 툴), Decision models(`decision_evaluate`), ClawHub marketplace, Tool-loop detection, 멀티유저/팀 Gateway, Boards/Trajectory/Snapshot/Link-understanding, 레거시 연속성(Warelay→Clawdbot→Moltbot→OpenClaw), 자체 업데이트 철학

## 확장 포인트
- 툴/정책: `src/agents/tool-catalog.ts`(66 ids+profiles), `src/agents/tool-policy-pipeline.ts`
- 설정 스키마: `src/config/zod-schema.root-shape.ts` + `schema.help.*.ts`, `docs/gateway/configuration-reference.md`
- 확장 작성: `extensions/AGENTS.md` + `src/plugin-sdk/AGENTS.md` (capability ladder)
- 메모리: `docs/concepts/memory-architecture.md` + `extensions/memory-core/index.ts`
- 훅 인벤토리: `docs/plugins/hooks/reference.md` + `src/hooks/bundled/`
