# pi-mono (pi.dev)

> Pi ships with powerful defaults but skips features like sub-agents and plan mode. (`badlogic/pi-mono`)
> **분석 기준**: `badlogic/pi-mono` · 커밋 `8369268` (2026-10-03) · 버전 `0.0.3` — 전체 목록은 [VERSIONS.md](../VERSIONS.md)

## 개요
Mario Zechner의 pi coding agent. Bun+TypeScript 모노레포 13 패키지. "강력한 기본값 + 기능 최소화" 철학.

## 디렉토리 구조
```
packages/            # 13 패키지 (디렉터리명 = 짧은 이름, npm 패키지명 = @earendil-works/pi-*)
├── coding-agent/     # CLI (메인)
├── agent/            # 에이전트 루프 (pkg: pi-agent)
├── ai/               # 멀티프로바이더 LLM API (pkg: pi-ai)
├── tui/              # 터미널 UI (pi-tui)
├── durable/          # durable 런타임 (pi-durable)
├── mcp/              # MCP (pi-mcp)
├── codemode/         # QuickJS 샌드박스 (pi-codemode)
├── protocol/         # CBOR 프로토콜 (pi-protocol)
├── client/           # 클라이언트 (pi-client)
├── server/           # 서버 (pi-server)
├── evals/            # evals (pi-evals)
├── chord/            # 앱 컴포지션
└── telemetry/        # 텔레메트리
```
> 참고: 소스 디렉터리명은 `tui/`, `client/` 등 짧은 형태이고, `package.json`의 이름은 `@earendil-works/pi-tui`, `@earendil-works/pi-client` … 이다.

## 메모리 관리
- 메모리 툴 없음
- 세션: JSONL 트리 `~/.pi/agent/sessions/--<path>--/<ts>_<id>.jsonl`, v3, `/tree` `/fork` `/clone`
- Compaction: `contextTokens > contextWindow - reserveTokens` (reserve 16384, keepRecent 20000). tool call/result 사이 안 자름. branch summarization, overflow 시 compact+retry 1회
- 컨텍스트 파일: agent dir → cwd 조상 walk, 각 디렉토리에서 `AGENTS.override.md > AGENTS.md > CLAUDE.md` first-match. worktree 시 메인 repo 파일 억제. `--no-context-files`로 해제

## 스킬
- **내장 0개** (의도적)
- 발견: `~/.pi/agent/skills/`, `<cwd>/.pi/skills/`, `~/.agents/skills/`, `.agents/skills/`(조상 walk), `--skill`/`settings.skills[]`
- Agent Skills spec 준수, SKILL.md 디렉토리가 루트 (재귀 X), bare `.md`도 로드
- 프롬프트엔 name+description+`<location>`만, 본문은 read로 on-demand
- `.agents/skills`는 trust gating

## MCP
- replaceable 내장 확장 (`src/extensions/mcp/`)
- 설정: `~/.pi/agent/mcp.json`, `<cwd>/.pi/mcp.json`(trust-gated)
- 네이밍: `mcp__<server>__<tool>`
- **exposure 기본 `codemode`** — MCP 툴이 codemode 스크립트의 `searchTools()`/`ALL_TOOLS`로만 접근. 대안: deferred/direct/hidden
- stdio / streamable HTTP (SSE 거부), startup은 direct만 10초 대기, 나머지 lazy
- OAuth: RFC 9728/8414/9207/8252 + cimd
- 리소스 툴 자동 추가: list_mcp_resources/list_mcp_resource_templates/read_mcp_resource

## 내장 툴
8개: read, bash, powershell, edit, write, grep, find, ls
**기본 활성 4개**: read, bash, edit, write (`DEFAULT_TOOL_NAMES`)
codemode, tool_search는 내장 확장 (기본 off, MCP 필요시 자동 활성화)
`defaultTools`에 `+name`/`-name` 수식자, `/reload`로 추가만 활성화
bash 결과: 모델엔 50KB/2000줄, codemode엔 최대 1MiB

## 에이전트
- **내장 서브에이전트 없음** (명시적 설계)
- experimental: `src/experimental/durable/subagent.ts` (child에서 자기 제거, replay:"safe"), `vacation.ts`
- 예제: `examples/extensions/subagent/`, `plan-mode/`

## 훅/플러그인
- 32 이벤트 (`src/core/extensions/types.ts`): resource/session/provider/agent/turn/message/tool/context/ui
- 3 의미: notify-only / transform / gate(`{block:true, reason}`)
- 확장 API: registerTool/Command/Shortcut/Flag/Provider/McpServer/VirtualModel/ToolRenderer/EntryRenderer/MessageRenderer, appendEntry, sendMessage, sendUserMessage, setActiveTools, pi.events
- Tool exposure: direct/model-only/codemode/deferred/hidden (MCP와 동일 추상화)
- 로드: project `.pi/extensions/` → global `~/.pi/agent/extensions/`
- **replaceable**: codemode/tool_search/mcp는 `replaceable: true`

## 설정
- `<agent-dir>/settings.json`(~70키), keybindings.json, mcp.json, models.json, auth.json, mcp-auth.json, SYSTEM.md, APPEND_SYSTEM.md, {extensions,skills,prompts,themes}/, sessions/, bin/
- agent dir: `~/.pi/agent` (`PI_CODING_AGENT_DIR`), project: `<cwd>/.pi`(trust 후)
- settings writes lockfile 보호 (`proper-lockfile`)
- 기본값: theme:"system", tuiMode:"fullscreen", transport:"auto", defaultProjectTrust:"ask", codemode.mode:"on", codemode.inlineBudget:3000

## 독특한 기능
- **codemode**: QuickJS WASM 샌드박스, `tools.<name>()` 호출, 출력만 모델에, `store()`/`load()` 세션 엔트리 영속(branch-aware), `searchTools()` BM25, `models` 글로벌
- **tool_search**: BM25 미선언 툴 검색, 다음 호출에만 선언, transcript 기록
- **권한 시스템 없음** (명시적 입장), 3 컨테이너화 패턴 문서화
- **Prompt-cache-aware**: mcp_servers append, 툴/프롬프트 변경도 delta, summarization cache write off
- **세션 트리 + branch-aware 상태**: store()/로드된 툴/선언이 branch-scoped
- **Contiguity-aware compaction**: tool call/result 사이 안 자름
- **Runtime-replaceable 코어**: codemode/tool_search/mcp
- **5 인터페이스**: TUI/print/JSON/RPC/SDK + pi-client/pi-protocol(CBOR)
- AGENTS.override.md + worktree shadowing

## 확장 포인트
- 확장 작성: `packages/coding-agent/examples/extensions/README.md` 참고, `pi --extension ./x.ts`
- 서브에이전트/plan mode: `examples/extensions/subagent/`, `plan-mode/` 패턴
- MCP exposure: `exposure: "direct"`(소규모)/`"deferred"`(대규모)
- 코어 이해: `agent-session.ts`(158KB), `extensions/runner.ts`(52KB), `resource-loader.ts`(48KB)
