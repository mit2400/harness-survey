# LSP / 린트 / 정적 분석 지원 비교

## 요약

| 하니스 | LSP 지원 | 제공 형태 | 진단(diagnostics) | AST 검색 | 린트/포맷 연동 |
|---|---|---|---|---|---|
| opencode | 내장 | `lsp` 툴(9연산) + 편집 시 자동 진단 | 편집/쓰기 후속 자동 수집 | ast-grep 외부 스킬 | LSP 포맷 포맷터(`src/format/formatter.ts`) |
| oh-my-openagent | 내장 (가장完备) | `lsp-tools-mcp` MCP 서버 + 공유 `lsp-daemon` | MCP 툴 + 디렉터리 단위 진단 | `ast-grep-mcp` 별도 MCP | `lsp-setup` 스킬(25개 서버 설치 가이드) |
| pi-mono | 코어 없음 | 외부 확장(`pi-lsp-client`) | `lsp_diagnostics` | 외부 확장(`pi-ast-grep`) | 없음(모델이 `bash`로 직접 실행) |
| oh-my-pi | 내장 (기본 off) | `lsp`, `ast_grep`, `ast_edit` 툴 | lsp 툴 | `ast_grep`/`ast_edit` 내장 | edit→ast_edit 자동 승격 |
| codex | **없음** | — (Cargo.lock의 `lsp-types`는 starlark의 전이 의존성) | — | — | 없음(전적으로 `apply_patch` 단일 편집) |
| claude-code | 내장 툴 + **플러그인 경유 서버** | `lspServers` in plugin manifest(`.lsp.json`) | LSP 진단 요약(클릭 확장) | 스킬/플러그인 | 플러그인 경유 |
| openclaw | 번들 경유 | `bundle-lsp.ts` + `agent-bundle-lsp-runtime.ts` | 번들 제공 | code-mode 결합 | 번들 제공 |
| hermes-agent | 내장 (가장 넓은 언어 파생) | `agent/lsp/` 8모듈 | `reporter.py` + CLI | 자체 `search_policy.py` | 서버 설치 자동화 |

---

## opencode

- **LSP 코어**: `packages/opencode/src/lsp/lsp.ts`(연결 관리), `lsp/client.ts`, `lsp/server.ts`
- **툴**: `src/tool/lsp.ts` + 프롬프트 `src/tool/lsp.txt`. 연산 9종:
  `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls`
  - `operation` / `filePath` / `line`(1-based) / `character`(1-based) / `query`(workspaceSymbol용) 스키마
- **진단 수집**: 전용 연산이 아니라 **편집 흐름에 후속** — `src/tool/edit.ts`, `src/tool/write.ts`, `src/tool/apply_patch.ts`가 LSP를 참조
- **포맷**: `src/format/formatter.ts` — LSP 포맷 capabilities 사용
- 그 외 `cli/cmd/debug/`, `server/routes/.../file.ts`에서 LSP 상태/구동 노출
- **ast-grep**: 내장 아님. 스킬 경유(`ast-grep` shared skill)

## oh-my-openagent

- **MCP形态**: `packages/lsp-tools-mcp/` — `src/lsp/client.ts`, `config-loader.ts`, `server-resolution.ts`, `server-installation.ts`, `server-install-state.ts`, `process.ts`, `workspace-edit.ts`, `infer-extension.ts`
- **공유 데몬**: `packages/lsp-daemon/` — 여러 세션이 한 LSP 프로세스를 공유
- **테스트가 곧 기능 목록**(`packages/lsp-tools-mcp/test/`):
  - `directory-diagnostics.test.ts` → **디렉터리 단위 진단**(pi生态 대비 우위)
  - `workspace-edit.test.ts` → rename 등 워크스페이스 편집
  - `transport-security.test.ts`, `server-installation-security.test.ts` → **서버 설치/전송 보안 검증**
  - `initialize-timeout.test.ts`, `startup-failure.test.ts` → 초기화 실패 복원
- **ast-grep**: 별도 MCP(`packages/ast-grep-mcp/`). OpenCode에선 스킬(sg CLI), Senpi에선 MCP
- **스킬**: `lsp-setup`(references/rust|typescript|python|go|… 서버 설치 절차), `programming`(ruff/pyright/clippy 가이드)

## pi-mono

- **코어에 LSP 없음**: `packages/coding-agent/src/core/tools/`는 read/bash/edit/write/grep/glob/ls. LSP·ast-grep 미포함(의도된 최소주의)
- **확장 레포로 분리**:
  - `repos/pi-lsp-client/` — 6개 툴 자동 등록
    - `lsp_diagnostics`(파일 **또는 디렉터리**, `severity` 필터, "build **전**에 실행"이 전제)
    - `lsp_goto_definition`, `lsp_find_references`(`includeDeclaration`), `lsp_symbols`(document/workspace), `lsp_prepare_rename`, `lsp_rename`
  - `repos/pi-ast-grep/` — AST 규칙 검색 확장
- **아키텍처 규약**(`pi-lsp-client/AGENTS.md`):
  - 서버 프로세스는 `LspManager` 단독 관리, `withLspClient(...)`로 획득·refCount·재시도
  - `lsp_rename`은 워크스페이스 편집이므로 `executionMode: "sequential"` 강제
  - 설치는 **감사 가능**: `/lsp install <id>`는 `AUTO_INSTALLABLE_SERVERS`의 문서화된 레시피만 실행, 그 외엔 수동 설치 안내
  - TUI는 `details` 타입을 읽고 포매터 문자열을 파싱하지 않음
- **grep**: 코어에 내장 grep(find) 툴(`core/tools/grep.ts`) + `truncate.ts` 출력 제한

## oh-my-pi

- **내강 툴 3종**(`packages/coding-agent/src/tools/builtin-names.ts`):
  - `:5` `"ast_grep"`, `:15` `"lsp"` (+ `tools/ast-grep.ts`, `tools/ast-edit.ts`)
- **ast_edit**: preview → `xd://resolve` → accept 3단계(`tools/ast-edit.ts`), read 없이 앵커 쓰면 거부
- **기본 off**: essential 14에 lsp/ast 미포함 → 필요 시 `+lsp` 형태로 활성화 후 `/reload`
- **정렬**: `ts-*` 규칙 12개, `go-*` 9개 등 26개 내장 rules가 언어별 지침 제공(린트/테스트 관례를 프롬프트로)
- **탐색**: Rust `ripgrep`/`glob`/`find` 크레이트 내장 → grep/find 툴이 네이티브 속도

## codex

- **LSP 지원 없음**. `codex-rs/` 전체 grep 결과 `lsp`는 `Cargo.lock`의 `lsp-types`와 `MODULE.bazel.lock`의 `starlark_syntax` 의존성뿐
- **원인**: 코드 편집을 `apply_patch` freeform 하나로 통합 설계(`core/src/tools/handlers/apply_patch_spec.rs`) → 편집 경로에 LSP 훅을 붙일 지점이 없음
- **대신**: 샌드박스 + `unified exec` + 테스트 실행으로 검증 루프 구성. 정적분석은 모델이 `shell`로 직접 구동
- **ast-grep도 없음** → `rg`/`grep` 셸 도구 사용

## claude-code

- **LSP 툴은 내장, 서버는 플러그인 경유** (`mods/`에는 LSP 관련 심볼 없음 → CLI 내부 구현)
- 서버 정의: 플러그인 manifest의 `lspServers`(`claude-code/CHANGELOG.md:559` — `claude plugin validate`가 `lspServers` 경로 검증, `:4309`/`:4421` — `/plugin` 화면에 LSP 서버 표시)
- CHANGELOG에서 확인되는 LSP 사용성 이력:
  - `:59` language server가 dynamic capability registration을 쓰거나 응답이 끊기면 무한 대기 → **서버별 `requestTimeout` 60초** 적용
  - `:3989` `workspaceSymbol`이 결과를 못 돌려주던 버그 → `query` 파라미터 추가
  - `:4842` LSP 진단 요약이 클릭/ctrl+o로 확장되도록 개선
  - `:2570` language server 재연결이 **전체 프롬프트 캐시를 무효화**하던 버그 수정(캐시 민감)
  - `:3281` LSP 열린 문서가 무제한 유지되던 메모리 누수 → **LRU 50개 문서로 제한**
  - `:945` 백그라운드 서브에이전트가 LSP 툴을 못 쓰던 버그 수정
  - `:1235` 프로젝트 전체 진단 발행으로 턴이 느려지던 문제 해결
- **강점**: 진단 UX(요약/확장), 캐시 무효화 방지, 플러그인 생애주기 관리(shutdown 실패 시에도 exit 전송 `:1528`)

## openclaw

- **번들(플러그인) 경유**: `src/plugins/bundle-lsp.ts`, `src/agents/agent-bundle-lsp-runtime.ts`(Windows spawn 대응 테스트 포함)
- **매니페스트 스키마**: `src/plugins/bundle-manifest.ts:370-372` — 인라인 `lspServers` 값 또는 `.lsp.json` 해석 결과가 있으면 `lspServers` capability로 등록
- **code-mode 결합**: `src/agents/code-mode.guest-source.test.ts` 등 — 코드 실행 샌드박스에서 LSP 접근 경로 존재
- **레지스트리/검증**: `src/cli/plugins-inspect-command.ts`, `src/plugins/installed-plugin-components.ts`에서 LSP 컴포넌트 노출

## hermes-agent

- **가장自成된 LSP 서브시스템**: `agent/lsp/`
  - `servers.py` — **30개 이상 언어 레지스트리**(python, typescript, tsx, js, jsx, vue, svelte, astro, go, rust, ruby, c, cpp, csharp, fsharp, swift, java, kotlin, yaml, json, lua, php, blade, prisma, dart, ocaml, shellscript, terraform, latex, gleam, clojure, nix, typst, haskell, julia, elixir, zig, dockerfile, powershell) + 확장자→언어 매핑
  - `manager.py`(수명 관리), `client.py`(JSON-RPC), `install.py`(설치), `reporter.py`(진단 리포팅), `workspace.py`, `cli.py`
- **진단 → 빌드 순서**: `lsp_diagnostics` 계열이 **build 실행 전** 진단을 전제로 설계(pi-lsp-client와 동일 철학)
- **탐색 정책**: `agent/search_policy.py`로 검색 도구 사용 정책 관리
- **검증 레시피**: `agent/verify/recipes.py`가 검증 단계별 명령 레시피 제공(린트/테스트 Runner)

---

## 관찰 / 시사점

1. **LSP 지원 세 갈래**
   - **내장 풀스택**: hermes(30+ 언어, 자체 설치/CLI), oh-my-openagent(MCP + 공유 데몬 + 디렉터리 진단), opencode(9연산 + 편집 후속 자동 진단)
   - **플러그인/확장 위임**: claude-code(플러그인 `.lsp.json`), openclaw(번들), pi-mono(외부 `pi-lsp-client`)
   - **없음**: codex(`apply_patch` 단일 편집 설계), oh-my-pi/pi-mono는 opt-in
2. **진단 수집 방식이 갈림**: 전용 조회 툴(opencode는 연산에 diagnostics가 **없고** 편집 후속으로 수집, hermes/pi-lsp-client는 `lsp_diagnostics` 명시적 호출, OmO는 MCP로 통합). 편집 직후 자동으로 붙는 쪽이 모델이 검증 루프를 지키기 쉽다
3. **ast-grep은 LSP와 별개 트랙**: LSP는 "언어 서버가 이해하는 심볼/타입", ast-grep은 "구문 패턴". oh-my-pi만 둘을 내장하고(`ast_grep`+`ast_edit`), 나머지는 스킬/외부 확장(`pi-ast-grep`, `ast-grep-mcp`, `ast-grep` 스킬)
4. **MCP를 LSP 전달자로 쓴 것은 OmO만의 선택**: `lsp-tools-mcp` 덕분에 세션 격리·권한 게이트(3-tier)를 그대로 재사용. 반면 opencode/hermes는 in-process 직접 구현
5. **린트 실행은 대부분 모델에 위임**: 전용 린트/포맷 툴을 둔 하니스가 없음. 차이는 "언제 붙이느냐"(opencode=편집 후속, hermes/pi=build 전 명시 호출)뿐이며, 실제 실행은 `bash`/`shell`로 돌아감
6. **캐시·성능 관리가 성숙도 지표**: claude-code의 LSP 재연결 시 프롬프트 캐시 무효화 방지(`CHANGELOG:2570`)와 문서 LRU(`:3281`)는 실사용에서摩擦이 없다는 뜻