# claude-code

> Claude Code is an agentic coding tool that lives in your terminal (`anthropics/claude-code`, 149k stars)
> **주의**: 이 레포는 CLI 소스가 아니라 공개 배포 레포 (plugins/mods/examples/CHANGELOG). 엔진은 컴파일된 네이티브 바이너리.

## 개요
공개 레포 구성: `plugins/`(13 번들 플러그인), `mods/`(바이너리 내장 TS 훅 모듈 4개), `examples/`(settings/mdm/hooks/gateway), `CHANGELOG.md`(900KB, 410 릴리스, 0.2.21→2.1.288), `.claude-plugin/marketplace.json`, `.github/workflows/`

## 핵심 파일
- `mods/types/claude-code.d.ts` — **13,186줄 엔진 타입 선언**. EngineInterface, CoreEngineInterface, OpEventOf, 32개 `*HookInput`, SettingsSource, TargetTier. 가장 가치 있는 파일
- `mods/README.md` — mod 시스템 설명 (hooks modules, noun contract, `claude plugin test mods/<name>`)
- `CHANGELOG.md` — 모든 기능의 1차 증거
- `plugins/README.md` — 플러그인별 내용 + 디렉토리 레이아웃
- `plugins/plugin-dev/skills/*/SKILL.md` — plugin-structure, skill-development, agent-development, hook-development, mcp-integration

## 메모리 관리
- **MEMORY.md 자동 메모리**: 인덱스 파일, 25KB/200줄 제한 (초과 시 에러), `/memory` 다이얼로그 on/off, `autoMemoryDirectory` 설정, 프롬프트 인젝션 하드닝
- CLAUDE.md: enterprise/user/project/local 스코프, `@` import, HTML 주석 숨김, nested CLAUDE.md는 Read 시 lazy load
- `.claude/rules/*.md`: `paths:` glob frontmatter로 path-scoped
- AGENTS.md: 내장 mod(`mods/agents-md/`), 4 모드 (claude-md/claude-md-or-agents-md/claude-md-and-agents-md/managed-only)
- Compaction: `/compact`, `/autocompact <tokens>`(모델별 저장), 재귀 복구, Sonnet 5는 1M 윈도우 ~967K에 발동, 게이트웨이 헤더 `x-claude-code-compaction`

## 스킬
- 바이너리 내장 (`disableBundledSkills`/`CLAUDE_CODE_DISABLE_BUNDLED_SKILLS`로 해제)
- 발견: `~/.claude/skills/`, `.claude/skills/`, `--add-dir`의 `.claude/skills`, nested `.claude/skills`(lazy, 충돌은 `<dir>:<name>`), plugin `skills/*/SKILL.md` + plugin root `SKILL.md`
- **Hot reload**: `~/.claude/skills`/`.claude/skills` 변경 시 재시작 없이, `SessionStart` 훅이 `reloadSkills: true` 반환 가능
- `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PLUGIN_ROOT}` 변수
- 번들 스킬 텍스트는 바이너리에 압축 저장 (~2MB 절약)

## MCP
- 설정 위치: project `.mcp.json`, settings.json `mcpServers`, plugin `.mcp.json`/`plugin.json`, agent 파일 inline, `.mcpb` 번들, managed `managedMcpServers`
- Transport: stdio, sse, http, ws
- **claude.ai 커넥터**: first-party MCP, trusted heading, OAuth, `disableClaudeAiConnectors`/`allowAllClaudeAiMcps`
- Deferral: `alwaysLoad: false`로 서버 툴 전부 tool search 뒤로
- 타임아웃: `MCP_CONNECT_TIMEOUT_MS`, `CLAUDE_CODE_MCP_STARTUP_WAIT_MS`, description 2048자 cap, requestTimeout 60s
- Enterprise: `allowManagedMcpServersOnly`, `deniedMcpServers`, `--strict-mcp-config`, `--bare`
- `claude mcp serve`: Claude Code 자체를 MCP 서버로 (Agent 툴 포함)
- 기본 활성 서버 없음 (claude.ai 커넥터만 로그인 시)

## 내장 툴
Bash(+PowerShell), Read, Write, Edit, NotebookEdit, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, Skill, Workflow, Artifact, LSP, Monitor, PushNotification, SlashCommand, EnterWorktree/ExitWorktree, TaskCreate/Get/Update/List, TodoWrite(모델 게이트), SendMessage, ListAgents, CronCreate/CronList, ScheduleWakeup, StructuredOutput, SendFeedback, ToolSearch
- TaskOutput은 제거됨 (출력 파일 직접 read)
- 툴셋 게이트: permission mode, agent frontmatter `tools:`, `--tools`, `--restricted`, `CLAUDE_CODE_SIMPLE`

## 에이전트
- 내장: `general-purpose`, `Explore`(Haiku, 세션 모델 상속), `Plan`, `claude`, `fork`(기본 on, 전체 대화+prompt cache 상속)
- 커스텀: `.claude/agents/*.md`, `~/.claude/agents/*.md`, frontmatter(name, description, model, color, tools, disallowedTools, permissionMode, effort, omitClaudeMd, context: fork), `--agents` JSON
- `/agents` 위저드 제거됨
- 서브에이전트 worktree 격리: `worktree.baseRef`, `worktree.sparsePaths`, `worktree.bgIsolation`
- **Agent teams**: teammates + SendMessage/ListAgents, TaskCreated/TaskCompleted/TeammateIdle 훅, "세션은 단일 implicit team"
- 비대화형 spawn은 기본 백그라운드
- 번들 플러그인 에이전트 16개: code-explorer, code-architect, code-reviewer, agent-creator, plugin-validator, skill-reviewer, conversation-analyzer, agent-sdk-verifier-{ts,py}, pr-review-toolkit 6개

## 훅/플러그인
- **클래식 훅 32개** (`mods/types/claude-code.d.ts:4288`): PreToolUse, PostToolUse, PostToolUseFailure, PostToolBatch, PermissionRequest, PermissionDenied, Notification, UserPromptSubmit, UserPromptExpansion, SessionStart, SessionEnd, Stop, StopFailure, SubagentStart, SubagentStop, PreCompact, PostCompact, PreModelSwitch, PostModelSwitch, Setup, TeammateIdle, TaskCreated, TaskCompleted, Elicitation, ElicitationResult, ConfigChange, InstructionsLoaded, WorktreeCreate, WorktreeRemove, CwdChanged, FileChanged, DirectoryAdded, MessageDisplay
- 설정: `hooks/hooks.json` (command, prompt, agent, function/module 타입)
- 출력: `permissionDecision`, `hookSpecificOutput.additionalContext`, `.updatedToolOutput`, `.sessionTitle`, `.worktreePath`, `continueOnBlock`, `args: string[]` exec form, per-hook timeout
- 정책: `disableAllHooks`, `allowManagedHooksOnly`
- **신형 function hooks** (`CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`): TS 타입, `register(on, options)` 단일 엔트리, ~90개 dot-name 이벤트 (`session.*`, `turn.*`, `tool.{register,list,check,describe,call}`, `command.*`, `agent.*`, `prompt.*`, `ui.*`, `model.complete`, `mcp.call`, `fs.*`, `store.*`, `clock.*`, `env.*`, `http.fetch`, `process.run`, `settings.read`, `plugin.register`, `telemetry.log`)
- **Mods**: 바이너리 내장 4개 (sec-default, diff, telemetry, agents-md), noun contract, `claude plugin test mods/<name>`

## 설정
- **5단계 precedence** (낮→높): user(`~/.claude/settings.json`) → project(`.claude/settings.json`) → local(`.claude/settings.local.json`) → flag(`--settings`/SDK) → policy(모든 managed tier 머지)
- Managed/policy: `managed-settings.json`(+drop-ins), MDM `.mobileconfig`/plist, Windows HKLM+ADMX, WSL `/etc/claude-code`, server-managed, `policyHelper`
- 파싱 불가 policy는 startup 거부
- 기타: `~/.claude.json`(legacy, `allowedTools`/`ignorePatterns`/`env`/`todoFeatureEnabled` 제거됨), `keybindings.json`, `.mcp.json`, `.claude-plugin/marketplace.json`, `~/.claude/skills/`, `~/.claude/workflows/`, `~/.claude/rules/`
- `CLAUDE_CONFIG_DIR`로 전체 재배치
- 주요 키: `permissions.{defaultMode,ask,allow,deny,additionalDirectories,disableBypassPermissionsMode,blockReadsOutsideWorkingDirectories}`, `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`, `strictKnownMarketplaces`, `sandbox.{enabled,autoAllowBashIfSandboxed,allowUnsandboxedCommands,excludedCommands,network.*,filesystem.*,credentials.*}`, `worktree.*`, `effort.level`, `voice.enabled`, `thinking.enabled`, `attribution`, `autoMemoryDirectory`, `disableBundledSkills`, `disableWorkflows`, `bashEditDiffEnabled`

## 독특한 기능
- Mods(바이너리 내장 TS 훅 모듈), Auto 권한 모드 기본(로컬 LLM 안전 분류기), 풍부한 네이티브 샌드박스(bubblewrap/socat, unix-socket+Mach lookup+domain allowlist, credential injection, nested weakening, cgroups), Agent teams, Checkpoints/`/rewind`, 배포 서피스(cloud/Cowork/Remote Control/self-hosted runner/VS Code/JetBrains/Desktop/Chrome/Gateway), `--bare`/`--safe-mode`/`--restricted`/`CLAUDE_CODE_SIMPLE`, Effort levels+`ultracode`, 플러그인 생태계(marketplace/validate/test/dev/hot reload/`.mcpb`/`${user_config.*}`), UI 확장(`ui.render` JSX-like, Raster/Image blits, panes), Hookify(평문 룰→훅 생성), Ralph Wiggum(Stop 훅 자기참조 반복), Gateway 헤더

## 확장 포인트
- 플러그인 작성: `plugins/plugin-dev/skills/plugin-structure/SKILL.md` (디렉토리 레이아웃 + plugin.json 스키마)
- 스킬 작성: `plugins/plugin-dev/skills/skill-development/SKILL.md` (SKILL.md 해부, 3단계 progressive disclosure, scripts/references/assets)
- 에이전트 작성: `plugins/plugin-dev/skills/agent-development/SKILL.md` (frontmatter 포맷)
- 훅 작성: `plugins/plugin-dev/skills/hook-development/SKILL.md` + `scripts/validate-hook-schema.sh`
- MCP 통합: `plugins/plugin-dev/skills/mcp-integration/references/server-types.md` (stdio/sse/http/ws, OAuth)
- 엔진 내부: `mods/types/claude-code.d.ts` 전체 정독
- 구현체가 필요하면: `npm i -g @anthropic-ai/claude-code` 후 `cli.js`/네이티브 바이너리 strings 분석
