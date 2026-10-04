# MCP 지원 비교

## 요약

| 하니스 | 기본 MCP 서버 | Transport | Exposure 모델 | OAuth | 리버스 (자기 자신을 MCP로) |
|---|---|---|---|---|---|
| opencode | 없음 (hosted Exa/Parallel만) | local stdio / remote HTTP | 전부 노출, `server_tool` 네이밍 | RFC 7591 DCR 자동 | 없음 |
| oh-my-openagent | **4개** (websearch/context7/grep_app/lsp) | stdio + HTTP | 3-tier (내장/`.mcp.json`/스킬 내장) | OAuth 2.0 + PKCE + DCR | lsp-tools-mcp |
| pi-mono | 없음 | stdio / streamable HTTP (SSE 거부) | **기본 `codemode`** (codemode 스크립트에서만 접근) | RFC 9728/8414/9207/8252 + cimd | 없음 |
| oh-my-pi | 없음 | stdio / http / sse | essential/discoverable/declared 3-tier | managed OAuth + auth-broker | 없음 |
| codex | 없음 (Codex Apps 커넥터) | Stdio / StreamableHttp | Direct/DirectModelOnly/Deferred + tool_search | ema_auth | codex-mcp serve |
| claude-code | 없음 (claude.ai 커넥터) | stdio/sse/http/ws | alwaysLoad:false로 deferred | OAuth + headersHelper | `claude mcp serve` |
| openclaw | 없음 | stdio/sse/streamable-http | profile + tool-policy 파이프라인 | per-requester OAuth | `openclaw mcp serve` |
| hermes-agent | 없음 (65 recipe) | stdio / HTTP | Tool Search로 deferred | 66KB OAuth 모듈 | `mcp_serve.py` |

## opencode
- 구현: `packages/opencode/src/mcp/` (index.ts 37KB, catalog.ts, auth.ts)
- 설정: `mcp: Record<string, Info>` in opencode.json, local(`command`)/remote(`url`)
- 툴 네이밍: `sanitize(server)_sanitize(tool)`, `{ "tools": { "myserver_*": false } }`로 비활성화
- 페이지네이션 1000페이지, outputSchema 검증 실패 시 tolerant fallback
- OAuth: 401 시 자동, RFC 7591 dynamic client registration, 토큰은 `~/.local/share/opencode/mcp-auth.json`
- MCP prompts → 슬래시 커맨드로 변환
- **기본 서버 없음**. hosted websearch만 (`mcp.exa.ai`, `search.parallel.ai`) — OpenCode/Zen provider이거나 `OPENCODE_ENABLE_EXA/PARALLEL`일 때

## oh-my-openagent
- **3-tier 아키텍처**:
  | Tier | 소스 | 로더 |
  |---|---|---|
  | 1. 내장 | `packages/omo-opencode/src/mcp/` | `createBuiltinMcps()` |
  | 2. Claude Code | `.mcp.json` | `claude-code-mcp-loader` (`${VAR}` 확장) |
  | 3. 스킬 내장 | SKILL.md frontmatter | `skill-mcp-manager` (세션별 stdio+HTTP) |
- Tier-1 기본 4개: `websearch`(Exa/Tavily), `context7`, `grep_app`, `lsp`(로컬 stdio, `lsp-tools-mcp`)
- 추가 MCP 패키지: `lsp-daemon`(유저당 공유 LSP 데몬, unix socket), `git-bash-mcp`(Windows), `ast-grep-mcp`(Senpi 전용)
- 보안: Tier-3 클라이언트는 `${sessionID}:${skillName}:${serverName}` 키로 세션 격리, `mcp_env_allowlist`는 user 레이어만
- Codex edition: `.mcp.json`에 lsp/git-bash/grep_app/context7 선언

## pi-mono
- **replaceable 내장 확장** (`src/extensions/mcp/`) — MCP 구현 자체를 다른 확장으로 교체 가능
- 설정: `~/.pi/agent/mcp.json` (user), `<cwd>/.pi/mcp.json` (project, trust-gated)
- 툴 네이밍: `mcp__<server>__<tool>`
- **exposure 기본값 `codemode`** — MCP 툴이 모델에 직접 선언되지 않고 codemode 스크립트의 `searchTools()`/`ALL_TOOLS`로만 접근. 대안: `deferred`(tool_search 경유), `direct`, `hidden`
- startup은 `direct` 선언 서버만 10초 대기, 나머지는 lazy connect
- SSE 명시적 거부, streamable HTTP만
- OAuth: RFC 9728 PRM discovery, RFC 8414, RFC 9207, RFC 8252 loopback, `cimd` 모드(`https://pi.dev/oauth/client.json`)
- 리소스 툴 `list_mcp_resources/list_mcp_resource_templates/read_mcp_resource` 자동 추가

## oh-my-pi
- 설정: `.omp/mcp.json` (project), `~/.omp/agent/mcp.json` (user), profile별 `~/.omp/profiles/<name>/agent/mcp.json`
- **10개 외부 포맷 네이티브 인식**: Claude Code, Codex, Gemini CLI, OpenCode, Cursor, Windsurf, VS Code, Claude marketplace, OMP extension, Agent Plugins
- Transport: stdio(기본)/http(Streamable HTTP)/sse(compat)
- 스키마: `{ $schema?, mcpServers, disabledServers?, enabledServers? }`, 번들 JSON Schema 제공
- OAuth: managed(`auth.credentialId`, deterministic id), explicit oauth block, auth-broker gateway
- 시크릿: `${VAR:-default}`, `!shell-command`(10초 타임아웃, 캐시)
- 기본값: `mcp.enableProjectConfig=true`, `startupTimeoutMs=250`, `notifications=false`
- **브라우저 자동화 MCP 서버 자동 드롭** (playwright/puppeteer 등, 네이티브 브라우저가 활성일 때)
- 툴 네이밍: `mcp__<server>_<tool>`

## codex
- 설정: `[mcp_servers.<name>]` in config.toml
- Transport: Stdio(`command/args/env/cwd`), StreamableHttp(`url/bearer_token_env_var/http_headers`)
- `McpServerConfig`: `enabled`, `required`, `startup_readiness`(live vs cached), `supports_parallel_tool_calls`, `tool_input_schema_max_bytes`(기본 5000), `omit_tools_from`
- **Codex Apps**: 커넥터 기반 내장 MCP (`codex-mcp/src/codex_apps.rs`), `server__connector` 네임스페이스
- Tool exposure: `ToolExposure::{Direct, DirectModelOnly, Deferred}`, `tool_search`로 deferred 디스커버리
- Governance: `mcp_requirements.rs`(admin allowlist), `mcp_edit.rs`
- OAuth: `ema_auth.rs`

## claude-code
- 설정 위치: project `.mcp.json`, settings.json `mcpServers`, plugin `.mcp.json`/`plugin.json`, agent 파일 inline, `.mcpb` 번들, managed `managedMcpServers`
- Transport: stdio, sse, http, ws
- **claude.ai 커넥터**: first-party MCP, trusted heading, OAuth, `disableClaudeAiConnectors`/`allowAllClaudeAiMcps`
- Deferral: `alwaysLoad: false`로 서버 툴 전부 tool search 뒤로
- 타임아웃: `MCP_CONNECT_TIMEOUT_MS`, `CLAUDE_CODE_MCP_STARTUP_WAIT_MS`, description 2048자 cap, requestTimeout 60s
- Enterprise: `allowManagedMcpServersOnly`, `deniedMcpServers`, `--strict-mcp-config`, `--bare`
- `claude mcp serve`: Claude Code 자체를 MCP 서버로 (Agent 툴 포함)

## openclaw
- **양방향**: client + server
- 설정: `mcp.servers.<name>` — Zod 스키마 (`src/config/zod-schema.mcp-server.ts`)
  - transports: stdio/sse/streamable-http
  - `connectionTimeoutMs`, `requestTimeoutMs`, `supportsParallelToolCalls`
  - OAuth: `auth: "oauth"` + `oauth.{identity: shared|per-requester, authProfileId, scope, redirectUrl}`
  - TLS: `sslVerify`, `clientCert`, `clientKey`
  - `toolFilter.{include[], exclude[]}`, `codex.{agents[], defaultToolsAppromsionMode}`
- 런타임: `src/mcp/` — channel-bridge(대화를 MCP로 노출), openclaw-tools-serve, plugin-tools-serve, tools-stdio-server
- 복원력: 서버별 지수 백오프 30s→10min
- CLI: `openclaw mcp add|doctor|probe|status|login|serve`
- MCP 툴도 profile + tool-policy 파이프라인 통과 (우회 불가)

## hermes-agent
- 설정: `mcp_servers:` in `~/.hermes/config.yaml`, `${VAR}` 보간, npx 캐시
- 기본값: `auto_reload_on_config_change: True`, `discovery_concurrency: 4`
- 서버별: `command/args/env`(stdio) 또는 `url`(HTTP), `enabled`, `sampling.{enabled:true, model, max_tokens_cap, ...}`, tool filtering, `lazy`(스키마 캐시, 첫 호출 시 연결)
- **65개 Nous 승인 recipe** (`optional-mcps/`), `hermes mcp install <name>`
- OAuth: `mcp_tool_oauth.py`(66KB), device flow, dashboard OAuth
- 안정성: death supervisor, liveness, health checks, schema cache, node ABI
- 동적 toolset `mcp-<server>`, Tool Search로 deferred 가능
- `mcp_serve.py`: Hermes 자체를 MCP 서버로
