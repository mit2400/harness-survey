# 훅/플러그인 시스템 비교

## 요약

| 하니스 | 훅 이벤트 수 | 플러그인 형태 | 특이점 |
|---|---|---|---|
| opencode | 20 hook 포인트 + 27 이벤트 | TS 모듈, npm/로컬 | 내부 플러그인 12개 (auth/provider) |
| oh-my-openagent | **54~62개 (5-tier)** | OpenCode 플러그인 + Codex/Senpi 어댑터 | boulder, keyword-detector, prompt-async-gate |
| pi-mono | 32개 | TS 모듈 (jiti) | codemode/tool_search/mcp가 replaceable |
| oh-my-pi | 40+ | Extension = Hook 통합 | marketplace, 16 외부 포맷 인식 |
| codex | 12개 | config.toml/hooks.json/plugin | MCP 기반 훅, trust hash |
| claude-code | **32 클래식 + ~90 function** | hooks.json + TS 모듈 | mods(바이너리 내장), function hooks |
| openclaw | 3종 분리 (~40 typed) | openclaw.plugin.json | internal/plugin/HTTP webhook |
| hermes-agent | 4종 (~50 VALID_HOOKS) | plugin.yaml + register(ctx) | shell subprocess 훅, outbound webhook |

## opencode
- **20 hook 포인트** (`packages/plugin/src/index.ts`): dispose, event, config, tool, auth, provider, chat.message, chat.params, chat.headers, permission.ask, command.execute.before, tool.execute.before, shell.env, tool.execute.after, experimental.chat.messages.transform, experimental.chat.system.transform, experimental.provider.small_model, experimental.session.compacting, experimental.compaction.autocontinue, experimental.text.complete, tool.definition
- 모두 `(input, output) => mutate output` 프로토콜
- **27 이벤트** (event 훅으로 구독): command.executed, file.edited, session.created/compacted/deleted/idle/error, tool.execute.before/after, permission.asked/replied, tui.* 등
- 플러그인 로딩: `{plugin,plugins}/*.{ts,js}` 자동 발견, npm spec은 Bun으로 `~/.cache/opencode/node_modules`에 설치, `.opencode/package.json` deps
- 내부 플러그인 12개: Codex auth(WebSocket transport), GitHub Copilot, Modal, GitLab, Poe, Cloudflare, Azure, DigitalOcean, Snowflake, xAI, Cerebras
- TUI 플러그인: 별도 런타임 (`packages/plugin/src/tui.ts`) — commands, dialogs, toasts, themes, keybinds, routes, sidebar

## oh-my-openagent
- **5-tier 훅 구성** (`packages/omo-opencode/src/hooks/AGENTS.md`):
  | Tier | Composer | 기본 | +team-mode |
  |---|---|---|---|
  | Session | create-session-hooks.ts | 24 | 24 |
  | Tool Guard | create-tool-guard-hooks.ts | 17 | 18 |
  | Transform | create-transform-hooks.ts | 4 | 6~7 |
  | Continuation | create-continuation-hooks.ts | 7 | 7 |
  | Skill | create-skill-hooks.ts | 2 | 2 |
  | 합계 | | 54 | 61~62 |
- 주요 훅: comment-checker(AI slop), write-existing-file-guard, bash-file-read-guard, prometheus-md-only, plan-format-validator, rules-injector, keyword-detector(IntentGate: ulw/search/analyze/team), todo-continuation-enforcer(**boulder**), compaction-context-injector, compaction-todo-preserver, preemptive-compaction, model-fallback(proactive) vs runtime-fallback(reactive, 의도적 분리), anthropic-context-window-limit-recovery, json-error-recovery, edit-error-recovery, delegate-task-retry, tool-pair-validator, webfetch-redirect-guard, session-notification(OS별)
- 14개 OpenCode hook handler 배선 (`plugin-interface.ts`)
- Claude Code 호환 로더: `claude-code-{agent,command,mcp}-loader`, `hooks/claude-code-hooks/`
- Codex edition: 10개 컴포넌트 vendored 플러그인, Codex 이벤트(SessionStart/UserPromptSubmit/PreToolUse/PostToolUse/PostCompact/Stop/SubagentStop)에 배선
- OpenClaw 연동: Discord/Telegram/HTTP/shell 양방향 (`src/openclaw/`)
- 빌트인 커맨드 7개: /goal, /refactor, /ulw-execute, /stop-continuation, /remove-ai-slops, /handoff, /hyperplan

## pi-mono
- **32 이벤트** (`src/core/extensions/types.ts` ExtensionEvent):
  - Resource: project_trust, resources_discover, mcp_servers_change
  - Session: session_start, session_before_compact, session_shutdown
  - Provider: before_provider_request, before_provider_headers, after_provider_response, provider_stream_event, model_select, thinking_level_select
  - Agent: before_agent_start, agent_start, agent_end, agent_before_settle, agent_settled
  - Turn/Message: turn_start/end, message_start/update/end
  - Tool: tool_call, tool_result, tool_execution_start/update/end, prepareLoadout, user_bash
  - Context/UI: context, context_with_system, cache_warming_decision, input, ui_prompt_start/end
- 3가지 이벤트 의미: notify-only / transform(수정된 값 반환) / gate(`{block:true, reason}`)
- 확장 API: registerTool, registerCommand, registerShortcut, registerFlag, registerProvider, registerMcpServer, registerVirtualModel, registerToolRenderer, registerEntryRenderer, registerMessageRenderer, appendEntry, sendMessage, sendUserMessage, setActiveTools, pi.events
- **Tool exposure 모델**: direct/model-only/codemode/deferred/hidden — MCP와 동일 추상화
- 로드 순서: project `.pi/extensions/` → global `~/.pi/agent/extensions/`
- **replaceable**: codemode/tool_search/mcp는 `replaceable: true` — 같은 이름의 확장이 완전히 대체

## oh-my-pi
- **Hooks = Extensions 통합**: `--hook`은 `--extension`의 alias, JS/TS hook factory가 ExtensionRunner로 로드
- **40+ 이벤트** (`extensibility/hooks/types.ts`): session(session_start, session_before_compact→cancel/CompactionResult 공급, session.compacting, session_compact, session_before_tree, session_tree, session_shutdown), agent(context, before_agent_start, agent_start/end, turn_start/end, auto_compaction_start/end, auto_retry_start/end, ttsr_triggered, todo_reminder), tool(tool_call→block/reason/input 교체/additionalContext, tool_result→필드별 머지 오버라이드)
- 확장 모듈: TS/JS, default export factory `(pi: ExtensionAPI) => void`, 이벤트 핸들러+툴+커맨드+키바인딩+플래그+메시지 렌더러+sendMessage/sendUserMessage/appendEntry
- 로드 순서: native `.omp/extensions/`(cwd만, ancestor walk 없음) → `~/.omp/agent/extensions/` → JS/TS hooks `.omp/hooks/pre|post/*.{ts,js}` → plugin `omp.extensions` → `--extension`
- Marketplace: `.omp-plugin/marketplace.json` (또는 Claude 호환 `.claude-plugin/marketplace.json`), `omp-plugins.lock.json`
- 제약: 로드 중 action 메서드 호출 시 `ExtensionRuntimeNotInitializedError`

## codex
- **12 이벤트** (`config/src/hook_config.rs` HookEventsToml): PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, SessionStart, SessionEnd, UserPromptSubmit, SubagentStart, SubagentStop, Stop, Interrupt
- 설정 소스: `[hooks.<Event>]` matcher 그룹 (config.toml) + `hooks.json` (config folder별) + plugin hooks
- **MCP 기반 훅**: `hooks/src/engine/mcp_runner.rs` — 훅이 MCP 서버 호출 가능
- 이벤트별 input/output JSON Schema 24개 생성 (`hooks/schema/generated/`)
- Trust: `HookStateToml { enabled, trusted_hash }` — 개별 비활성화/신뢰
- Admin: `requirements.toml`의 `allow_managed_hooks_only = true`
- SessionStart/SubagentStart stdout이 `additionalContext` 주입, SubagentStart는 컨텍스트 주입 전용
- Feature: `CodexHooks` Stable + 기본 on

## claude-code
- **클래식 훅 32개** (`mods/types/claude-code.d.ts:4288` HookInput):
  PreToolUse, PostToolUse, PostToolUseFailure, PostToolBatch, PermissionRequest, PermissionDenied, Notification, UserPromptSubmit, UserPromptExpansion, SessionStart, SessionEnd, Stop, StopFailure, SubagentStart, SubagentStop, PreCompact, PostCompact, PreModelSwitch, PostModelSwitch, Setup, TeammateIdle, TaskCreated, TaskCompleted, Elicitation, ElicitationResult, ConfigChange, InstructionsLoaded, WorktreeCreate, WorktreeRemove, CwdChanged, FileChanged, DirectoryAdded, MessageDisplay
- 설정: `hooks/hooks.json` (command, prompt, agent, function/module 타입)
- 출력: `permissionDecision`, `hookSpecificOutput.additionalContext`, `.updatedToolOutput`, `.sessionTitle`, `.worktreePath`, `continueOnBlock`, `args: string[]` exec form, per-hook timeout
- 정책: `disableAllHooks`, `allowManagedHooksOnly`
- **신형 function hooks** (`CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`): TS 타입, `register(on, options)` 단일 엔트리, ~90개 dot-name 이벤트 (`session.*`, `turn.*`, `tool.{register,list,check,describe,call}`, `command.*`, `agent.*`, `prompt.*`, `ui.*`, `model.complete`, `mcp.call`, `fs.*`, `store.*`, `clock.*`, `env.*`, `http.fetch`, `process.run`, `settings.read`, `plugin.register`, `telemetry.log`)
- **Mods**: 바이너리 내장 4개 (sec-default, diff, telemetry, agents-md), noun contract (`types/index.d.ts`), `claude plugin test mods/<name>`

## openclaw
3종 분리 (`docs/automation/hooks.md`, `docs/plugins/hooks.md`):
**(a) Internal hooks** — operator 설치 JS/TS + `HOOK.md`, colon 이벤트명 (`command:new`, `agent:bootstrap`), trusted/unsandboxed/in-Gateway
- 엔진: `src/hooks/{discovery,loader,module-loader,config,configured,workspace,fire-and-forget,policy}.ts`
- **5개 번들**: boot-md, bootstrap-extra-files, command-logger, compaction-notifier, session-memory
- Gmail transport: `src/hooks/gmail.ts`, `gmail-watcher.ts`
- CLI: `openclaw hooks list|info|enable|disable|check`

**(b) Plugin hooks (typed)** — `api.on("hook_name", handler)`, ~40개 (`docs/plugins/hooks/reference.md`):
- Agent/model: before_model_resolve, agent_turn_prepare, before_prompt_build, before_agent_run, before_agent_reply, before_agent_finalize, agent_end, heartbeat_prompt_contribution, model_call_started/ended, llm_input, llm_output
- Tools: before_tool_call, after_tool_call, resolve_exec_env, tool_result_persist, before_message_write
- Messages: inbound_claim, channel_pairing_requested, message_received, message_sending, reply_payload_sending, message_sent, before_dispatch, reply_dispatch
- Sessions: session_start/end, before_compaction, after_compaction, before_reset
- Gateway/plugins: gateway_start/stop, cron_reconciled, cron_changed, before_install, skill_proposal_evaluate, skill_proposal_changed, skill_changed
- 훅별 타임아웃/실패 정책 (before_agent_run/before_tool_call/before_install은 15s fail-closed, skill_proposal_evaluate 120s)

**(c) HTTP webhooks** — `hooks.*` config, 기본 `enabled: false`, `path: "/hooks"`, `token` 필수, `allowedAgentIds`, `presets: ["gmail"]`, `mappings[]`

플러그인 시스템: `openclaw.plugin.json` 매니페스트, loader/registry, ClawHub/npm/git/local 설치, 163개 번들 (`extensions/`), capability ladder (`src/plugin-sdk/AGENTS.md`)

## hermes-agent
4종 (`website/docs/user-guide/features/hooks.md`):
| 시스템 | 등록 | 실행 위치 | 격리 |
|---|---|---|---|
| Gateway hooks | `HOOK.yaml` + `handler.py` in `<profile>/hooks/<name>/` | Gateway만 | in-process, dir-trust |
| Plugin hooks | `ctx.register_hook()` in `~/.hermes/plugins/<name>/` | CLI + Gateway | in-process |
| Shell hooks | `hooks:` block in config.yaml → `~/.hermes/agent-hooks/` | CLI + Gateway + Desktop/TUI/dashboard | subprocess |
| Outbound webhooks | `hooks.outbound:` list | CLI + Gateway | HTTP POST, HMAC 서명 |
- Gateway 이벤트: gateway:startup, session:start/end/reset/compress, agent:start/step/end, reaction:added/removed, command:* (와일드카드)
- **VALID_HOOKS ~50개** (`hermes_cli/plugins.py:109`): pre_tool_call, post_tool_call, transform_terminal_output, transform_tool_result, transform_llm_output, pre_llm_call, post_llm_call, on_stream_start/delta/end, on_interim_message, pre_verify, pre/post_api_request, api_request_error, pre/post_auxiliary_call, transform_api_error_classification, on_session_start/end/finalize/reset, on_skill_lifecycle, subagent_start/stop, pre_gateway_dispatch, agent_loop_stopped, pre/post_approval_request/post_approval_response, on_room_member_activity, pre_transcription, kanban_task_claimed/completed/blocked, on_kanban_worker_spawned/exited/stale_claim, on_kanban_task_updated, on_kanban_dispatch_tick, gateway_platform_event, pre_command
- Shell hook 스키마: `hooks.<event>: [{matcher: regex, command, timeout: 60(clamp 300), fail_closed}]`, JSON stdin/stdout, Cursor/Claude Code 호환 `failClosed`
- Outbound: `hooks.outbound: [{name, url, events, secret_env, timeout, matcher}]`, payload에 `profile` 필드
- 플러그인 발견 4소스 (later wins): bundled → user → project(`HERMES_ENABLE_PROJECT_PLUGINS=1`) → pip entry point. 번들 플러그인은 **기본 disabled**
- 번들 플러그인: disk-cleanup, security-guidance(25 write_file/patch 룰), spotify(7 툴), google_meet, teams_pipeline + memory/context_engine/cron_providers/web 네임스페이스
- 프로바이더 컬렉션: model-providers 36, platforms 22, web 13, image_gen 8, browser 3, dashboard_auth 4, video_gen
- 플러그인 카탈로그: 407개 커뮤니티 매니페스트 (`plugin-catalog/*.yaml`)
