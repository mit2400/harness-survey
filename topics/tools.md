# 내장 툴 비교

## 요약

| 하니스 | 툴 수 | 기본 활성 | 특이 툴 |
|---|---|---|---|
| opencode | ~16 (V1) / 12 (V2) | 전부 | code-mode(`execute`), plan, lsp, question |
| oh-my-openagent | 12 상시 + 26 조건부 | 12개 | session_*, background_*, team_* 12, monitor 4, goal 3, hashline edit |
| pi-mono | 8 + codemode/tool_search | **4개** (read/bash/edit/write) | codemode(QuickJS), tool_search |
| oh-my-pi | 31 (+hidden 3) | essential 14 | eval, debug(DAP), ida, ast_edit, github, browser/computer |
| codex | ~20 | 대부분 | apply_patch(freeform), tool_search, view_image, multi-agent V1/V2 |
| claude-code | ~25 | 대부분 | Workflow, Artifact, LSP, Monitor, Cron*, SendMessage/ListAgents |
| openclaw | **66** | profile별 | decision_evaluate, secrets, transcripts, tts, pdf, canvas, portal |
| hermes-agent | 41 toolset (~60 코어) | `hermes-cli` | terminal(8 백엔드), kanban_* 12, browser_vault_*, execute_code |

## opencode
V1 (`packages/opencode/src/tool/registry.ts`): invalid, question, shell(bash), read, glob, grep, edit, write, task, fetch(webfetch), todo(todowrite), search(websearch), skill, patch(apply_patch, gpt-* 전용), execute(code-mode), lsp(실험적), plan(실험적)
V2 (`packages/core/src/tool/builtins.ts`): apply-patch, bash, edit, glob, grep, question, read, skill, todowrite, webfetch, websearch, write (12개)
- 커스텀 툴: `{tool,tools}/*.{js,ts}` → `tool()`/`tool.schema` (`@opencode-ai/plugin`)
- code-mode: MCP 툴을 스크립트로 오케스트레이션 (`packages/codemode/`)

## oh-my-openagent
항상 켜짐 (12): grep, glob, session_list, session_read, session_search, session_info, background_output, background_cancel, call_omo_agent, task, skill, skill_mcp
조건부 (+26): look_at, interactive_bash(tmux), monitor_start/stop/list/output(4), task_create/get/list/update(4), edit(hashline), create_goal/update_goal/get_goal(3), team_*(12)
- LSP 툴은 레지스트리 툴이 아니라 내장 `lsp` MCP로 제공: lsp_status/diagnostics/goto_definition/find_references/symbols/prepare_rename/rename/install_decision/format
- ast-grep은 OpenCode에선 스킬(`sg` CLI), Senpi에선 MCP
- **Hashline edit**: Read 출력에 `LINE#ID` 해시 태그, edit이 해시 검증 후 stale patch 거부 (xxHash32)

## pi-mono
8개: `read, bash, powershell, edit, write, grep, find, ls`
- **기본 활성 4개만**: `DEFAULT_TOOL_NAMES = ["read","bash","edit","write"]`
- grep/find/ls/powershell은 opt-in
- codemode, tool_search는 내장 확장 (기본 off, MCP가 필요시 자동 활성화)
- `defaultTools`에 `+name`/`-name` 수식자, `/reload`로 추가만 활성화
- bash 결과: 모델엔 50KB/2000줄, codemode엔 최대 1MiB

## oh-my-pi
`BUILTIN_TOOL_NAMES` 31: read, bash, edit, ast_grep, ast_edit, ask, debug, ida, eval, github, glob, grep, find, lsp, checkpoint, rewind, context_notes, new_context, security_scan, task, wait, todo, web_search, write, memory_edit, retain, recall, reflect, learn, manage_skill
Hidden 3: yield, goal, think
Essential 14 (항상 top-level): read, write, bash, edit, glob, find, eval, task, wait, learn, manage_skill, context_notes, new_context
- 3-tier: essential(top-level) / discoverable(`xd://` 뒤) / declared
- 기본 off: github, security_scan, generate_image, tts, checkpoint, rewind, memory 툴, debug, ida, glob, grep, web_search, todo, ask, AST 툴, lsp, goal
- **eval**: 영속 Python+Bun JS 셀, 루프백 브리지로 에이전트 툴 역호출
- **debug**: DAP (lldb/dlv/debugpy), **ida**: 바이너리 분석
- **ast_edit**: preview→`xd://resolve`→accept
- **github**: PR-as-paths (gh_* 툴 난립 방지)
- checkpoint↔rewind 자동 페어링, grep→ast_grep, edit→ast_edit 자동 포함

## codex
`codex-rs/core/src/tools/spec_plan.rs`가 레지스트리 빌더:
- shell/exec_command(+write_stdin), apply_patch(freeform custom), update_plan, view_image, tool_search, web_search(hosted), request_permissions, new_context_window/get_context_remaining, current_time, sleep, list_mcp_resources/list_mcp_resource_templates/read_mcp_resource, request_user_input, send_message_to_user_async, request_plugin_install/list_available_plugins_to_install, wait_for_environment, test_sync, multi-agent V1/V2
- ToolSpec: Function/Namespace/ToolSearch/WebSearch/Freeform
- 3 exposure 모드: Direct/DirectModelOnly/Deferred
- Code Mode: 툴을 JS로 호출, namespaces + tool_search 인디렉션

## claude-code
Bash(+PowerShell), Read, Write, Edit, NotebookEdit, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, Skill, Workflow, Artifact, LSP, Monitor, PushNotification, SlashCommand, EnterWorktree/ExitWorktree, TaskCreate/Get/Update/List, TodoWrite(모델 게이트, Opus 4.8/Sonnet 5/Fable 5/Mythos 5에선 off), SendMessage, ListAgents, CronCreate/CronList, ScheduleWakeup, StructuredOutput, SendFeedback, ToolSearch
- TaskOutput은 제거됨 (출력 파일 직접 read)
- 툴셋은 permission mode, agent frontmatter `tools:`, `--tools`, `--restricted`, `CLAUDE_CODE_SIMPLE`로 게이트

## openclaw
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

## hermes-agent
기본 toolset: `hermes-cli`. 41개 named toolset (web, search, x_search, vision, video, image_gen, video_gen, computer_use, terminal, skills, browser, cronjob, file, tts, todo, memory, context_engine, session_search, connections, project, bot_room, desktop_ui, setup, clarify, code_execution, delegation, homeassistant, kanban, discord, discord_admin, yuanbao, feishu_doc, feishu_drive, spotify, debugging, safe, coding, hermes-acp, hermes-api-server, hermes-webhook, hermes-gateway)
`_HERMES_CORE_TOOLS` (~60): web_search, web_extract, terminal, process_manage, read_file, write_file, patch, search_files, vision_analyze, image_generate, skills_list/view/manage, browser_*(20+), text_to_speech, todo_list, memory, session_search, clarify, execute_code, delegate_task, cronjob_manage, ha_*(4), kanban_*(12), computer_use, manage_connections
- terminal: 8 백엔드 (local/docker/ssh/singularity/modal/daytona/vercel_sandbox) + sudo/background/heredoc
- browser: CDP/camofox/Lightpanda/cloud/browser-use/vault
- `disabled_toolsets`는 파이프라인 끝에서 strict 차감
- Tool Search: `tool_search`/`tool_describe`/`tool_call` 3개 브리지로 MCP+비코어 플러그인 툴 지연 로딩
