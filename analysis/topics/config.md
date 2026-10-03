# 설정 파일 비교

## 요약

| 하니스 | 메인 설정 | 포맷 | 계층/우선순위 | 프로필 |
|---|---|---|---|---|
| opencode | `opencode.json(c)` | JSON/JSONC | remote→global→OPENCODE_CONFIG→project→.opencode→env→managed→MDM | 없음 |
| oh-my-openagent | `~/.omo/omo.jsonc` | JSONC (Zod v4) | user→project(조상 walk)→harness→profile | `OMO_PROFILE` |
| pi-mono | `~/.pi/agent/settings.json` | JSON | user→project(trust-gated) | 없음 |
| oh-my-pi | `~/.omp/agent/config.yml` | YAML | env→runtime→overlay→project→global→default (6-layer) | `OMP_PROFILE` |
| codex | `config.toml` | TOML | /etc→MDM→user→project→CLI | `[profiles.*]` |
| claude-code | `settings.json` | JSON | user→project→local→flag→policy (5단계) | 없음 |
| openclaw | `~/.openclaw/openclaw.json` | JSON/JSON5 | OPENCLAW_CONFIG_PATH→state dir→profile | `OPENCLAW_PROFILE` |
| hermes-agent | `~/.hermes/config.yaml` + `.env` | YAML + env | CLI→config.yaml→.env→defaults | profile별 재귀 홈 |

## opencode
- 파일: `opencode.json`/`opencode.jsonc`/`config.json`, `tui.json`/`tui.jsonc`, `.well-known/opencode`, macOS MDM `.mobileconfig`
- 포맷: JSON/JSONC, `$schema: "https://opencode.ai/config.json"`
- 우선순위 (later wins): remote well-known → global(`~/.config/opencode`) → `OPENCODE_CONFIG` → project → `.opencode/` dirs → `OPENCODE_CONFIG_CONTENT` → managed → MDM. **머지됨 (대체 아님)**
- `.opencode/` 디렉토리 레이아웃: `agent(s)/`, `command(s)/`, `mode(s)/`, `plugin(s)/`, `skill(s)/`, `tool(s)/`, `themes/`, `glossary/`
- V1 top-level 키: `$schema, shell, server, command, skills, references, watcher, snapshot, plugin, share, autoshare, autoupdate, disabled_providers, enabled_providers, model, small_model, default_agent, subagent_depth, username, mode, agent, provider, mcp, formatter, lsp, instructions, layout, permission, tools, attachment, enterprise, tool_output, compaction, experimental{...}`
- env/flags: `OPENCODE_CONFIG`, `OPENCODE_CONFIG_CONTENT`, `OPENCODE_CONFIG_DIR`, `OPENCODE_DB`, `OPENCODE_TUI_CONFIG`, `OPENCODE_DISABLE_PROJECT_CONFIG`, `OPENCODE_EXPERIMENTAL*`, `OPENCODE_DISABLE_*`, `OPENCODE_ENABLE_EXA/PARALLEL` 등

## oh-my-openagent
- 파일: `~/.omo/omo.jsonc` (user), `.omo/omo.jsonc` (project, cwd→$HOME walk, nearest wins)
- 포맷: JSONC (주석+trailing comma), Zod v4, snake_case
- harness별 resolution: shared base → `[opencode]`/`[native]`/`[codex]` → `profiles.<P>` → `profiles.<P>.[harness]` → Zod defaults
- 프로필 활성화: `OMO_PROFILE` > `OCX_PROFILE` > `OPENCODE_CONFIG_DIR` tail > none
- 머지: plain object deep-merge, scalar/array 교체, `__proto__`/`prototype`/`constructor` 제거 (prototype pollution 방지)
- 견고성: malformed 값은 최소 실패 subtree만 drop, `invalid-value` diagnostic, unknown keys 리포트, symlink 스킵, `omo doctor`
- top-level 키: `$schema, categories, agents, task, teams, models, model_profiles, model_profile, memory, git_master, telemetry, disabled_skills, [opencode], [native], [codex], profiles, _migrations, legacy_migrations`
- 마이그레이션: lock+journal+no-clobber, 백업 `~/.omo/migration-backup-<ts>/`, `config migrate --dry-run`
- 상태 디렉토리: `~/.omo/agent`(canonical), `~/.omo/teams/`, `~/.omo/rules/`, `~/.omo/plans/`, `~/.omo/goal/`, `~/.omo/lsp-daemon/`, project `.omo/{plans,tasks,teams,ulw-loop,notepads,evidence}`
- CLI: install/run/doctor/config migrate/cleanup/version/refresh-model-capabilities/boulder/ulw-loop/worktree-sweep/mcp oauth
- 5 bin alias: oh-my-opencode, oh-my-openagent, omo, lazycodex, lazycodex-ai

## pi-mono
- 파일: `<agent-dir>/settings.json` (~70키), `keybindings.json`, `mcp.json`, `models.json`, `auth.json`, `mcp-auth.json`, `AGENTS.*.md`/`CLAUDE.md`, `SYSTEM.md`, `APPEND_SYSTEM.md`, `{extensions,skills,prompts,themes}/`, `sessions/`, `bin/`
- agent dir: `~/.pi/agent` (`PI_CODING_AGENT_DIR`로 변경), project: `<cwd>/.pi` (trust 후 로드)
- settings writes는 lockfile 보호 (`proper-lockfile`, 10 retries) — 같은 cwd에서 다중 pi 세션 대응
- 기본값: `theme: "system"`, `tuiMode: "fullscreen"`, `transport: "auto"`, `steeringMode/followUpMode: all|one-at-a-time`, `doubleEscapeAction: "tree"`, `defaultProjectTrust: "ask"`, `codemode.mode: "on"`, `codemode.inlineBudget: 3000`, `markdown.mermaid: "streaming"`, `httpIdleTimeoutMs: 300000`

## oh-my-pi
- 파일: `~/.omp/agent/config.yml` (YAML canonical, `config.yaml` compat), project `<cwd>/.omp/config.yml` + `.omp/settings.json`(legacy)
- **6-layer precedence** (높은 순): env var → runtime overrides → overlays(`PI_CONFIG_FILES`, `--config`) → project → global → definition default
- Registry 패턴: 각 setting이 `register({id, type, default, env?, protocolDefault?, validate?, pathScoped?, credential?, ui?})` → typed handle, `provenance()` 반환
- Config roots: `.omp` → `.claude` → `.codex` → `.gemini` (user→project), `.omp`는 generic source discovery에서 제외
- 프로필: `omp --profile <name>` / `OMP_PROFILE` / `PI_PROFILE` → `~/.omp/profiles/<name>/agent/`
- Path-scoped arrays: `enabledModels`, `disabledProviders`만 `path:` 형식 허용
- 로드 실패: invalid YAML은 `.broken-<ts>-<pid>-<uuid>`로 이동 후 startup 실패
- XDG: `XDG_DATA_HOME`/`XDG_STATE_HOME`/`XDG_CACHE_HOME`
- CLI: `omp config set|get|reset|path|list|init-xdg`

## codex
- 파일: `config.toml` (TOML), `hooks.json`, `[profiles.*]`, `[permissions]`, `requirements.toml`, `auth.json`, `config-schema/`(생성된 JSON Schema)
- 계층 (낮→높): `/etc/codex/config.toml` → `/etc/codex/requirements.toml` → macOS MDM(`com.openai.codex`, `config_toml_base64`) → Windows `%ProgramData%\OpenAI\Codex` → `/etc/codex/managed_config.toml` → cloud bundle → `$CODEX_HOME/config.toml`(기본 `~/.codex`) → `<project>/.codex/config.toml` → CLI overrides
- **Project-local denylist**: repo config가 `openai_base_url`, `chatgpt_base_url`, `model_provider(s)`, `notify`, `profile(s)`, `otel` 등 설정 불가
- `strict_config.rs`: unknown/ignored TOML 키 경고
- 주요 키: `model_auto_compact_token_limit`, `model_post_turn_compact_threshold_percent`, `compact_prompt`, `project_root_markers`, `project_doc_max_bytes`, `project_doc_fallback_filenames`, `[memories]`, `[mcp_servers]`, `[skills]`, `multi_agent_v2.*`

## claude-code
- 파일: `~/.claude/settings.json`(user), `.claude/settings.json`(project), `.claude/settings.local.json`(local), `--settings`/SDK inline(flag), managed policy(policy)
- **5단계 precedence** (낮→높): user → project → local → flag → policy (모든 managed tier 머지)
- Managed/policy 소스: `managed-settings.json`(+drop-ins), MDM `.mobileconfig`/plist, Windows HKLM+ADMX, WSL `/etc/claude-code`, server-managed, `policyHelper`
- 파싱 불가 policy는 startup 거부 (조용한 unenforce 대신)
- 기타: `~/.claude.json`(legacy, `allowedTools`/`ignorePatterns`/`env`/`todoFeatureEnabled` 제거됨), `keybindings.json`, `.mcp.json`, `.claude-plugin/marketplace.json`, `~/.claude/skills/`, `~/.claude/workflows/`, `~/.claude/rules/`
- `CLAUDE_CONFIG_DIR`로 전체 재배치
- 주요 키: `permissions.{defaultMode,ask,allow,deny,additionalDirectories,disableBypassPermissionsMode,blockReadsOutsideWorkingDirectories}`, `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`, `strictKnownMarketplaces`, `sandbox.{enabled,autoAllowBashIfSandboxed,allowUnsandboxedCommands,excludedCommands,network.*,filesystem.*,credentials.*}`, `worktree.*`, `effort.level`, `voice.enabled`, `thinking.enabled`, `attribution`, `autoMemoryDirectory`, `disableBundledSkills`, `disableWorkflows`, `bashEditDiffEnabled`

## openclaw
- 파일: `~/.openclaw/openclaw.json` (JSON/JSON5), `$include` 디렉티브, `OPENCLAW_INCLUDE_ROOTS`
- 경로: `OPENCLAW_CONFIG_PATH`로 override, state dir `OPENCLAW_STATE_DIR`(기본 `~/.openclaw`), `OPENCLAW_HOME`
- Legacy: `clawdbot.json`, legacy state dirs 자동 발견
- 프로필: `OPENCLAW_PROFILE` → `~/.openclaw-<profile>/`, gateway port 18789 기본, 프로필별 20000-40000 해시
- 주요 경로: workspace `~/.openclaw/workspace`, agent dir `~/.openclaw/agents/<agentId>/agent`, sessions DB `<agentDir>/openclaw-agent.sqlite`, credentials `<stateDir>/credentials`, sandboxes `~/.openclaw/sandboxes`, logs `~/.openclaw/logs/`
- Config domains: `channels, agents/sessions/messages, worktreeRoot, tools, models, mcp, skills, plugins, browser, ui, desktop, gateway, cloudWorkers, hooks, canvas, discovery, env, secrets, auth storage, audit, logging, diagnostics, telemetry, update, acp, wizard, identity, automations, memory, meta, experimental, desktop.host`
- 스키마: `src/config/zod-schema.root-shape.ts`(613줄), `src/config/types.*.ts`, `schema.help.*.ts`
- **정책**: long-lived alias 없음, breaking change는 반드시 `doctor` 마이그레이션 (`openclaw doctor --fix`)

## hermes-agent
- 파일: `<HERMES_HOME>/config.yaml`(YAML, 비시크릿), `.env`(시크릿), `auth.json`, `profile.yaml`, `SOUL.md`, `memories/{MEMORY.md,USER.md}`, `skills/`, `plugins/<name>/`, `hooks/<name>/`, `agent-hooks/`, `cron/`, `sessions/`, `pets/`, `skins/`, `logs/`, `state.db`, `kanban.db`, `projects.db`, `attachments/`, `profiles/<name>/`
- HERMES_HOME: `~/.hermes` 기본, `$HERMES_HOME` override, ContextVar override
- Precedence: CLI args → config.yaml → .env → built-in defaults
- 템플릿: `cli-config.yaml.example`(124KB, 주석 포함), `.env.example`(26KB), `_config_version`으로 마이그레이션
- **99 top-level 키**: model, providers, fallback_providers, fallback, credential_pool_strategies, toolsets, database, runtime, max_concurrent_sessions, max_live_sessions(16), session, attachments, agent, terminal, web, browser, checkpoints, context_file_max_chars, mcp, tool_output, tool_loop_guardrails, compression, prompt_caching, openrouter, bedrock, auxiliary, display, dashboard, privacy, tts, stt, voice, vision, wake_word, human_delay, context, memory, delegation, goals, loops, moa, skills, curator, honcho, timezone, slack, discord, whatsapp, telegram, mattermost, matrix, approvals, command_allowlist, quick_commands, platform_hints, plugins, hooks, hooks_auto_accept, personalities, auth, security, cron, kanban, bot_mode, code_execution, tools, logging, model_catalog, model_overrides, models_dev, network, monitoring, gateway, streaming, sessions, onboarding, telemetry, doctor, updates, lsp, x_search, vault, secrets, bot_desktop, computer_use, proxy, desktop, nous, vertex, local_runtime, _config_version
- 주요 기본값: `runtime.nofile_soft_limit: 4096`, `database.journal_mode: "wal"`, `agent.max_turns: None`, `agent.gateway_timeout: 1800`, `tool_output.max_bytes: 50000`, `display.skin: "default"`, `cron.catch_up_missed: True`, `cron.allow_agent_scheduling: False`, `goals.max_turns: 20`, `loops.{min_interval_seconds:30, max_ticks:100}`
- `hermes config get|set|unset|edit|check|migrate`, **UPPER_SNAKE 키는 자동으로 .env로 라우팅** (`config_env_routing.py`)
- Profile identity markers: config.yaml, .env, SOUL.md, profile.yaml, auth.json, state.db
