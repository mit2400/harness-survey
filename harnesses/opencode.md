# opencode

> The open source coding agent. (`anomalyco/opencode`, 211k stars)

## 개요
모노레포, 33 패키지. V1(`packages/opencode/src/`)과 V2(`packages/core/src/`) 병행 구현. V2는 Effect-native, Location/Application 스코프 아키텍처.

## 디렉토리 구조
```
packages/
├── opencode/          # V1 메인 (Effect-ified)
├── core/              # V2 Session Core
├── schema/            # 공유 스키마
├── plugin/            # 플러그인 API (Hooks 인터페이스)
├── protocol/          # 프로토콜
├── server/            # 서버
├── sdk/, sdk-next/    # SDK
├── codemode/          # CodeMode 패키지
├── function/, identity/, llm/
├── slack/             # Slack 연동
└── web/               # 웹 앱 + docs
```

## 메모리 관리
- 전용 메모리 서브시스템 **없음** (서드파티 `opencode-supern` 플러그인만)
- 세션: SQLite (`<XDG_DATA>/opencode/opencode-<channel>.db`), 테이블 `session/message/part/todo/session_input/session_context_epoch` (`packages/core/src/session/sql.ts`)
- Compaction: V1 `packages/opencode/src/session/compaction.ts` (PRUNE_MINIMUM=20_000, PRUNE_PROTECT=40_000), V2 `packages/core/src/session/compaction.ts`
- AGENTS.md: `packages/opencode/src/session/instruction.ts` — config-dir → `~/.claude/CLAUDE.md` → walk-up(AGENTS.md→CLAUDE.md→CONTEXT.md). read 대상 기준 상위 AGENTS.md를 메시지당 1회 자동 첨부
- System Context: `packages/core/src/system-context/` (env, date 블록 diff 주입)
- Snapshot: `packages/opencode/src/snapshot/index.ts` — 매 턴 shadow gitdir에 커밋

## 스킬
- 내장 1개: `customize-opencode` (`packages/core/src/plugin/skill/customize-opencode.md`)
- 발견: `packages/opencode/src/skill/index.ts` — `{skill,skills}/**/SKILL.md`, `~/.claude`, `~/.agents`, `config.skills.paths/urls`
- Remote: `discovery.ts` — `<url>/index.json` → `Global.Path.cache/skills/`, atomic swap
- Frontmatter: name(dir명 일치, kebab), description(필수), license, compatibility, metadata

## MCP
- 구현: `packages/opencode/src/mcp/` (index.ts 37KB, catalog.ts, auth.ts, oauth-callback.ts)
- Transport: local stdio / remote HTTP
- 네이밍: `sanitize(server)_sanitize(tool)`, `{ "tools": { "myserver_*": false } }`
- OAuth: 401 시 자동, RFC 7591 DCR, 토큰 `~/.local/share/opencode/mcp-auth.json`
- MCP prompts → 슬래시 커맨드
- 기본 서버 없음. hosted websearch만 (`mcp.exa.ai`, `search.parallel.ai`)

## 내장 툴
V1 (`packages/opencode/src/tool/registry.ts`): bash(shell), read, edit, write, glob, grep, task, todowrite, webfetch, websearch, skill, question, apply_patch(gpt-* 전용), execute(code-mode), lsp(실험적), plan(실험적)
V2 (`packages/core/src/tool/builtins.ts`): apply-patch, bash, edit, glob, grep, question, read, skill, todowrite, webfetch, websearch, write
커스텀: `{tool,tools}/*.{js,ts}` → `tool()`/`tool.schema`

## 에이전트
`packages/opencode/src/agent/agent.ts`: build(기본), plan, general, explore, compaction(hidden), title(hidden), summary(hidden)
커스텀: `{agent,agents}/**/*.md` YAML frontmatter

## 훅/플러그인
- 20 hook 포인트 (`packages/plugin/src/index.ts`): dispose, event, config, tool, auth, provider, chat.message/params/headers, permission.ask, command.execute.before, tool.execute.before/after, shell.env, experimental.chat.messages/system.transform, experimental.provider.small_model, experimental.session.compacting, experimental.compaction.autocontinue, experimental.text.complete, tool.definition
- 27 이벤트 (event 훅)
- 내부 플러그인 12개: Codex auth(WebSocket), GitHub Copilot, Modal, GitLab, Poe, Cloudflare, Azure, DigitalOcean, Snowflake, xAI, Cerebras
- TUI 플러그인: `packages/plugin/src/tui.ts`

## 설정
- `opencode.json(c)`, `tui.json(c)`, `.well-known/opencode`, MDM
- 우선순위: remote→global→OPENCODE_CONFIG→project→.opencode→env→managed→MDM (머지)
- `.opencode/` 레이아웃: agent(s)/, command(s)/, mode(s)/, plugin(s)/, skill(s)/, tool(s)/, themes/, glossary/
- env: `OPENCODE_CONFIG*`, `OPENCODE_DB`, `OPENCODE_EXPERIMENTAL*`, `OPENCODE_DISABLE_*`, `OPENCODE_ENABLE_EXA/PARALLEL`

## 독특한 기능
- Code Mode (`packages/codemode/`), References (`core/src/reference.ts`), 17 provider별 프롬프트, System Context algebra, git snapshot revert, durable prompt inbox(V2), Codex WebSocket transport, hosted websearch, experimental policies, V1/V2 이중 아키텍처

## 확장 포인트
- 툴 추가: V1 `packages/opencode/src/tool/<name>.ts` + registry.ts, V2 `packages/core/src/tool/` + builtins.ts
- 에이전트: `packages/core/src/plugin/agent.ts`(V2) 또는 `agent/agent.ts`(V1)
- 스킬: `.opencode/skills/<name>/SKILL.md`, 번들은 `packages/core/src/plugin/skill/`
- 훅: `packages/plugin/src/index.ts` Hooks 인터페이스 확장
- 설정 키: `packages/core/src/v1/config/config.ts`(V1) / `packages/core/src/config.ts`(V2)
