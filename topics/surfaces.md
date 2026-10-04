# AI 코딩 하니스 딜리버리 서피스 / 런타임 진입점 비교

> 8개 AI 코딩 하니스의 실행 진입점(CLI, 기계 인터페이스, IDE, 웹/모바일, 프로토콜, 패키징)을 비교한다.
> 기준 경로: `/home/minkoo/study/repos/`

## 요약 비교

| 하니스 | CLI | 기계 인터페이스 | IDE | 웹/모바일 | 프로토콜 | 패키징 |
|--------|-----|-----------------|-----|-----------|----------|--------|
| **opencode** | TUI + `run` one-shot + REPL | SDK, HTTP API, MCP | VS Code 확장, Desktop 앱 | Web UI, Desktop | HTTP/REST, SSE | npm (bun) |
| **pi-mono** | `packages/tui` (TUI) | `packages/{client,server,protocol,durable}` | 없음 (TUI only) | 없음 | JSON-RPC (protocol) | npm |
| **oh-my-pi** | `main.ts` REPL, `cli.ts` one-shot, `acp-bridge.ts` | RPC, SDK (`sdk.ts`), MCP, LSP | LSP daemon, Cursor bridge | Web UI | JSON-RPC, MCP, LSP | npm |
| **codex** | `tui/`, `exec/`, `cli/` | app-server, app-server-protocol, codex-mcp | VS Code (code-mode-host) | Cloud (cloud-client) | WebSocket, app-server-protocol | Rust 바이너리 릴리즈 |
| **claude-code** | TUI, `-p` headless, `claude mcp serve` | Agent SDK, MCP serve | VS Code 확장, JetBrains | Desktop 앱, Web, Cowork cloud | MCP, SDK | npm |
| **openclaw** | Gateway 서버 (장기 실행) | MCP serve (openclaw-tools-serve 등) | 없음 | Android/iOS/Linux/macOS 앱, Web UI | HTTP, WebSocket, Webhooks | npm |
| **oh-my-openagent** | 어댑터별 (omo-opencode/codex/senpi/native) | MCP client/stdio, LSP daemon | LSP daemon | Web UI | MCP, LSP | bun + Rust 바이너리 |
| **hermes-agent** | `hermes_cli/` (TUI, one-shot, REPL) | ACP adapter (`acp_adapter/server.py`) | 없음 | Web (Vite), Desktop 앱 | Gateway, Relay, Webhooks | Python (pip/uv) |

---

## 1. opencode

**CLI 진입점**
- `cli/cmd/run.ts` — one-shot 실행
- `packages/tui/` — 풀스크린 TUI
- `packages/opencode/src/server/` — 서버 모드 (데몬)

**기계 인터페이스**
- `packages/sdk/` — 공식 SDK (TypeScript)
- `packages/opencode/src/server/` — HTTP API 서버
- `packages/client/`, `packages/protocol/` — 클라이언트/프로토콜 레이어

**IDE 통합**
- `packages/web/` — Web UI
- `packages/desktop/` — Desktop 앱
- VS Code 확장 존재

**웹/데스크톱/모바일**
- `packages/web/`, `packages/desktop/` — 웹/데스크톱 클라이언트
- 모바일 앱 없음

**프로토콜**
- HTTP/REST API + SSE 스트리밍
- SDK를 통한 프로그래밍 접근

**패키징**
- npm 패키지 (bun 기반 빌드)

---

## 2. pi-mono

**CLI 진입점**
- `packages/tui/` — pi-tui (TUI 인터페이스)
- `packages/coding-agent/` — 코딩 에이전트 코어

**기계 인터페이스** (공식 문서 기준 5개 인터페이스)
- `packages/pi-client` → `packages/client/` — 클라이언트 SDK
- `packages/pi-server` → `packages/server/` — 서버
- `packages/pi-protocol` → `packages/protocol/` — 프로토콜 정의
- `packages/pi-durable` → `packages/durable/` — 지속성/상태 관리
- `packages/mcp` — MCP 서버 모드

**IDE 통합**
- 없음 (TUI only)

**웹/데스크톱/모바일**
- 없음

**프로토콜**
- JSON-RPC 기반 (`packages/protocol/`)

**패키징**
- npm 모노레포

---

## 3. oh-my-pi

**CLI 진입점**
- `packages/coding-agent/src/main.ts` — REPL 진입
- `packages/coding-agent/src/cli.ts` — one-shot CLI
- `packages/coding-agent/src/acp-bridge.ts` — ACP 브리지

**기계 인터페이스**
- `packages/coding-agent/src/rpc/` — RPC 인터페이스
- `packages/coding-agent/src/sdk.ts` — SDK
- `packages/coding-agent/src/mcp/` — MCP 서버
- `packages/coding-agent/src/lsp/` — LSP 서버

**IDE 통합**
- `packages/coding-agent/src/lsp/` — LSP daemon
- `packages/coding-agent/src/cursor-bridge-tools.ts` — Cursor 브리지

**웹/데스크톱/모바일**
- `packages/coding-agent/src/web/` — 웹 인터페이스

**프로토콜**
- JSON-RPC, MCP, LSP

**패키징**
- npm 모노레포

---

## 4. codex

**CLI 진입점**
- `codex-rs/tui/` — TUI
- `codex-rs/exec/` — one-shot 실행
- `codex-rs/cli/` — CLI 진입

**기계 인터페이스**
- `codex-rs/app-server/` — 앱 서버
- `codex-rs/app-server-protocol/` — 프로토콜 정의
- `codex-rs/codex-mcp` — MCP 서버
- `codex-rs/app-server-client/` — 클라이언트

**IDE 통합**
- `codex-rs/code-mode-host/`, `codex-rs/code-mode/` — VS Code 통합

**웹/데스크톱/모바일**
- `codex-rs/cloud-client/`, `codex-rs/cloud-tasks/` — 클라우드 연동
- 데스크톱 앱 없음 (TUI + IDE 중심)

**프로토콜**
- app-server-protocol (WebSocket 기반)
- `codex-rs/websocket-client/`, `codex-rs/websocket-auth/`

**패키징**
- Rust 크레이트 릴리즈 (바이너리 배포)

---

## 5. claude-code

**CLI 진입점**
- TUI (대화형)
- `-p` / `--print` — headless one-shot
- `claude mcp serve` — MCP 서버 모드

**기계 인터페이스**
- Agent SDK (`@anthropic-ai/claude-agent-sdk`)
- `claude mcp serve` — MCP 서버
- `mods/types/claude-code.d.ts` — SDK 타입 정의

**IDE 통합**
- VS Code 확장 (CHANGELOG: `[VSCode]`)
- JetBrains 플러그인

**웹/데스크톱/모바일**
- Claude Desktop 앱 (`claude --desktop`)
- Web (claude.ai)
- Cowork 클라우드 세션

**프로토콜**
- MCP (서버/클라이언트)
- SDK (프로그래밍 인터페이스)

**패키징**
- npm (`@anthropic-ai/claude-code`)

---

## 6. openclaw

**CLI 진입점**
- `src/gateway/` — 장기 실행 게이트웨이 서버 (데몬)
- `src/server.ts` — 서버 진입

**기계 인터페이스**
- `src/mcp/openclaw-tools-serve.ts` — MCP 서버 (툴 제공)
- `src/mcp/plugin-tools-serve.ts` — 플러그인 툴 MCP
- `src/mcp/codex-supervision-tools-serve.ts` — Codex 감독 툴
- `src/mcp/channel-server.ts` — 채널 서버

**IDE 통합**
- 없음

**웹/데스크톱/모바일**
- `apps/android/`, `apps/ios/`, `apps/linux/`, `apps/macos/` — 네이티브 앱
- `src/gateway/` — 웹 게이트웨이 UI
- `src/channels/` — 22개 플랫폼 채널 어댑터 (Slack, Discord 등)

**프로토콜**
- HTTP API (`server-http.ts`)
- WebSocket (`websocket-protocol.ts`)
- Webhooks

**패키징**
- npm

---

## 7. oh-my-openagent

**CLI 진입점**
- `packages/omo-opencode/` — opencode 어댑터
- `packages/omo-codex/` — codex 어댑터
- `packages/omo-senpi/` — senpi 어댑터
- `packages/omo-native` — 네이티브 런타임
- 플랫폼 런처: `oh-my-opencode-{platform}` 바이너리들

**기계 인터페이스**
- `packages/mcp-client-core/` — MCP 클라이언트
- `packages/mcp-stdio-core/` — MCP stdio 전송
- `packages/lsp-core/`, `packages/lsp-daemon/` — LSP 서버
- `packages/omo-codex/`, `packages/omo-opencode/` — 하니스별 어댑터

**IDE 통합**
- `packages/lsp-core/`, `packages/lsp-daemon/` — LSP daemon

**웹/데스크톱/모바일**
- `packages/web/` — 웹 UI
- `packages/senpi-desktop-engine/` — 데스크톱 엔진

**프로토콜**
- MCP (client/stdio)
- LSP

**패키징**
- bun (TypeScript) + Rust 바이너리 (플랫폼별)

---

## 8. hermes-agent

**CLI 진입점**
- `hermes_cli/main.py` — 메인 진입
- `hermes_cli/oneshot.py` — one-shot 실행
- `hermes_cli/curses_ui.py` — TUI
- `hermes_cli/commands.py` — 명령 처리

**기계 인터페이스**
- `acp_adapter/server.py` — ACP 서버
- `acp_adapter/entry.py` — ACP 진입점
- `gateway/` — 게이트웨이 (22개 플랫폼 어댑터)

**IDE 통합**
- 없음

**웹/데스크톱/모바일**
- `web/` — Vite 기반 웹 UI
- `apps/desktop/` — 데스크톱 앱
- `apps/bootstrap-installer/` — 부트스트랩 인스톨러

**프로토콜**
- Gateway (멀티 플랫폼 메시징)
- Relay (`gateway/relay/`)
- Webhooks (`hermes_cli/webhook.py`)

**패키징**
- Python (pip/uv)

---

## 관찰 및 시사점

1. **CLI 스펙트럼**: 대부분 TUI + one-shot + REPL 3가지를 제공하나, pi-mono는 TUI에 집중하고 openclaw는 장기 실행 게이트웨이 서버 모델을 채택한다.

2. **기계 인터페이스 다양화**: MCP가 사실상 표준으로 자리잡았다 (opencode, pi-mono, oh-my-pi, codex, claude-code, openclaw, oh-my-openagent 모두 지원). LSP는 opencode·hermes-agent가 내장하고, claude-code(플러그인)·openclaw(번들)·pi-mono(확장 `pi-lsp-client`)가 경유한다(`lsp-lint.md` 참조).

3. **IDE 통합 양극화**: VS Code/JetBrains 확장은 codex·claude-code·opencode가 제공하고, LSP는 opencode·hermes-agent(내장)·oh-my-openagent(MCP)·oh-my-pi·claude-code(플러그인)·openclaw(번들)가 지원한다. 반면 pi-mono는 IDE 확장이 없고 LSP를 외부 확장(`pi-lsp-client`)으로 분리했다. hermes-agent는 IDE 확장 대신 30+ 언어 LSP로 "언어 서버 통합" 축을 택한다(`lsp-lint.md` 참조).

4. **멀티 플랫폼 채널**: openclaw(22개 채널)와 hermes-agent(22개 플랫폼)는 메시징 플랫폼 커버리지에서 압도적이다. 나머지는 터미널/IDE에 집중한다.

5. **패키징 전략**: TypeScript 생태계(opencode, pi-mono, oh-my-pi, claude-code, openclaw)는 npm/bun, Rust 생태계(codex, oh-my-openagent)는 바이너리 릴리즈, Python(hermes-agent)는 pip/uv를 사용한다.

6. **어댑터 패턴**: oh-my-openagent는 별도 하니스(opencode, codex, senpi)를 어댑터로 연결하는 "메타 하니스" 포지셔닝이며, 나머지는 자체 런타임을 가진다.
