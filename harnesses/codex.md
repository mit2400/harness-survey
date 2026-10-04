# codex

> Lightweight coding agent that runs in your terminal (`openai/codex`, 127k stars)
> **분석 기준**: `openai/codex` · 커밋 `b741e48` (2026-10-03) · 버전 `0.0.0-dev` — 전체 목록은 [VERSIONS.md](../VERSIONS.md)

## 개요
Rust 워크스페이스 (`codex-rs/`, ~122 crates) + 얇은 TS `codex-cli/` 래퍼.

## 디렉토리 구조
```
codex-rs/
├── core/              # 메인 (tools, compact, memories, guardian, realtime)
├── config/            # 설정 (config_toml, loader, mcp_types, hook_config)
├── rollout/           # 세션 (sessions, state_db, compression, search)
├── memories/          # 메모리 (read/write)
├── skills/            # 스킬 (assets/samples, parser, loading, selection)
├── codex-mcp/         # MCP (connection_manager, catalog, codex_apps, ema_auth)
├── tools/             # ToolSpec, tool_config
├── hooks/             # 훅 엔진 (engine, events, schema)
├── agent-roles/       # 에이전트 역할 발견
├── agent-identity/    # Ed25519/JWT
├── agent-graph-store/ # 에이전트 그래프
├── agent-message-board-client/
├── collaboration-mode-templates/
├── core-plugins/      # 플러그인 로더
├── code-mode/         # Code Mode
├── v8-poc/            # V8 샌드박스 PoC
├── exec-server/       # 원격 실행
├── external-agent-migration/
├── worktree/          # git worktree
├── features/          # 157 feature flag
├── protocol/          # ModeKind, TUI_VISIBLE_COLLABORATION_MODES
├── prompts/           # compact 프롬프트 등
├── history/           # compaction checkpoint
├── state/             # SQLite 마이그레이션
├── sandboxing/, linux-sandbox/, windows-sandbox-rs/, mxc-sandbox/, bwrap/
├── realtime-webrtc/, otel/, analytics/, diagnostics/, feedback/
└── ...
```

## 메모리 관리
- **Memories 기능** (`codex-rs/memories/`, 기본 off): 2-phase 비동기 파이프라인
  - Phase 1: 스레드별 rollout 추출 (병렬, leased job, secret redaction) → `raw_memory`, `rollout_summary`, `rollout_slug`
  - Phase 2: 전역 통합 → `~/.codex/memories/raw_memories.md`
  - 설정 `[memories]`: version, dual_write, disable_on_external_context, generate_memories, use_memories, dedicated_tools, max_raw_memories_for_consolidation(256), max_unused_days(30), max_rollout_age_days(10), max_rollouts_per_startup(2), min_rollout_idle_hours(6), min_rate_limit_remaining_percent(25), extract_model, consolidation_model
- 세션: `~/.codex/sessions/YYYY/MM/DD/rollout-<ts>-<uuid>.jsonl`, archived_sessions/, SQLite state DB(주 인덱스), zstd 압축, `codex resume`
- Compaction: `core/src/compact.rs` (run_compact_task, run_inline_auto_compact_task, COMPACT_USER_MESSAGE_MAX_TOKENS=20_000), remote V2 (`compact_remote_v2.rs`), checkpoint (`history/src/compaction_checkpoint.rs`), 설정 `model_auto_compact_token_limit`, `model_post_turn_compact_threshold_percent`, `compact_prompt`
- AGENTS.md: `core/src/agents_md.rs` — project root(`.git`)→cwd walk, `AGENTS.override.md` 우선, `project_doc_max_bytes`, `project_doc_fallback_filenames`, `project_root_markers`

## 스킬
- 5개 번들 (`skills/src/assets/samples/`, `include_dir!` 컴파일 타임 임베드): imagegen, openai-docs, review-agent, skill-creator, skill-installer
- 설치: `$CODEX_HOME/skills/.system` (fingerprint marker)
- 발견 루트: `<config_folder>/skills`(Repo), `$CODEX_HOME/skills`(deprecated), `~/.agents/skills`(User), `$CODEX_HOME/skills/.system`(System), `/etc/codex/skills`(Admin), plugin roots
- 메커니즘: frontmatter 파싱, `$skill` 명시 멘션, implicit invocation, `@` 툴 멘션, snapshot 캐시
- 설정 `[skills]`: bundled.enabled, config(이름/경로별 enable/disable/rename), catalog budget = 컨텍스트 윈도우 2%

## MCP
- 설정: `[mcp_servers.<name>]` in config.toml
- Transport: Stdio(command/args/env/cwd), StreamableHttp(url/bearer_token_env_var/http_headers)
- `McpServerConfig`: enabled, required, startup_readiness(live vs cached), supports_parallel_tool_calls, tool_input_schema_max_bytes(기본 5000), omit_tools_from
- **Codex Apps**: 커넥터 기반 내장 MCP (`codex-mcp/src/codex_apps.rs`), `server__connector` 네임스페이스
- Tool exposure: Direct/DirectModelOnly/Deferred, tool_search
- Governance: mcp_requirements.rs(admin allowlist), mcp_edit.rs
- OAuth: ema_auth.rs

## 내장 툴
`core/src/tools/spec_plan.rs`가 레지스트리 빌더:
- shell/exec_command(+write_stdin), apply_patch(freeform), update_plan, view_image, tool_search, web_search(hosted), request_permissions, new_context_window/get_context_remaining, current_time, sleep, list_mcp_resources/list_mcp_resource_templates/read_mcp_resource, request_user_input, send_message_to_user_async, request_plugin_install/list_available_plugins_to_install, wait_for_environment, test_sync, multi-agent V1/V2
- ToolSpec: Function/Namespace/ToolSearch/WebSearch/Freeform
- 3 exposure 모드, trusted vs external 등록, per-tool hooks, network approval deferral, sandbox auto-escalation

## 에이전트
- Collaboration modes: `ModeKind::{Default, Plan}`, 템플릿 `collaboration-mode-templates/templates/{default,plan}.md`. Plan만 request_user_input 허용
- Multi-agent V1: spawn_agent, send_input, resume_agent, wait_agent, close_agent (depth 제한)
- Multi-agent V2: spawn_agent, send_message, followup_task, wait_agent, interrupt_agent, list_agents. `multi_agent_v2` Stable 기본 off, `multi_agent` 기본 on
- Agent roles: `.toml` 재귀 발견 (`$CODEX_HOME/agents/` + `[agents]` 테이블), name/description/nickname_candidates + flattened ConfigToml
- Agent identity: Ed25519 + JWT (`agent-identity/`)
- 인프라: agent-graph-store, agent-message-board-client, core/src/agent/

## 훅/플러그인
- **12 이벤트** (`config/src/hook_config.rs`): PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, SessionStart, SessionEnd, UserPromptSubmit, SubagentStart, SubagentStop, Stop, Interrupt
- 설정: `[hooks.<Event>]` matcher 그룹 + `hooks.json` + plugin hooks
- **MCP 기반 훅**: `hooks/src/engine/mcp_runner.rs`
- 이벤트별 JSON Schema 24개 (`hooks/schema/generated/`)
- Trust: `HookStateToml { enabled, trusted_hash }`
- Admin: `requirements.toml`의 `allow_managed_hooks_only = true`
- SessionStart/SubagentStart stdout이 additionalContext 주입
- Feature: `CodexHooks` Stable + 기본 on
- Plugins: `core-plugins/` (git/npm/remote marketplace, curated OpenAI, plugin.json, plugin-sourced skills/MCP/hooks/apps)

## 설정
- `config.toml` (TOML), `hooks.json`, `[profiles.*]`, `[permissions]`, `requirements.toml`, `auth.json`
- 계층: `/etc/codex/config.toml` → requirements.toml → macOS MDM → Windows `%ProgramData%\OpenAI\Codex` → managed_config.toml → cloud bundle → `$CODEX_HOME/config.toml` → `<project>/.codex/config.toml` → CLI
- **Project-local denylist**: repo config가 openai_base_url, chatgpt_base_url, model_provider(s), notify, profile(s), otel 등 설정 불가
- strict_config.rs: unknown 키 경고

## 독특한 기능
- Plugins + marketplaces, Code Mode(`code-mode/`, `v8-poc/`), exec-server 원격 실행, Guardian(LLM 승인 리뷰어), worktree 격리, external-agent-migration(Claude/Cursor config/hooks/subagents/memory 임포트), shell snapshot, unified exec(zsh fork), sandbox matrix(Seatbelt/Landlock/Windows/mxc/bwrap), requirements/policy plane, 157 feature flag, Realtime(WebRTC/WebSocket), Memories 2-phase 파이프라인

## 확장 포인트
- 툴: `core/src/tools/spec_plan.rs` + `handlers/`
- 설정 표면: `config/src/config_toml.rs` (~2000줄) + `config-schema/`
- 플러그인: `core-plugins/src/loader.rs`
- 스킬 발견: `ext/skills/src/host_roots.rs`
- Feature flag: `features/src/lib.rs` (key로 stage+default 조회)
- 메모리: `memories/README.md` → `memories/write/src/phase1.rs`/`phase2.rs`
