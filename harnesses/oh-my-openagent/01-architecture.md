# oh-my-openagent (OmO) — 전체 아키텍처 맵

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13`
> 이 문서는 [oh-my-openagent.md](oh-my-openagent.md)의 "기능별" 문서를 companion으로 하는 "구조/레이어" 문서다.
> 기능 디테일(훅 목록·에이전트·MCP 툴 이름 등)은 원본 문서를 볼 것. 여기서는 **무엇이 어디에 있고 왜 그 모양인지**만 다룬다.

---

## 1. 패키지 지형

**50개 디렉터리**, 그중 12개가 플랫폼 바이너리 패키지라 실질 패키지는 **38개**다. 루트 `package.json`의 `workspaces` 배열에는 34개가 등록돼 있고(`packages/rules-engine` … `packages/omo-native`), `web`·`get-worker`·`lsp-tools-mcp`·`lsp-daemon`은 워크스페이스가 아니라 별도 빌드 대상(`bunfig.toml`의 `pathIgnorePatterns`에서 분리)이다.

LoC는 **테스트 파일 제외 `src/` 기준**이다(아래 표의 수치는 이 기준). 전체 38개 중 16개만 측정되어 있으므로 나머지는 `—`로 두었다(측정 기준 불일치로 임의 수치를 채우지 않음).

### 1.1 어댑터 (4개)

| 패키지 | LoC | 역할 | 분류 |
|---|---:|---|---|
| `packages/omo-opencode/` | **126,239** | OpenCode Ultimate 에디션. 루트 npm dist(`dist/index.js`)의 **빌드 엔트리** | 어댑터 |
| `packages/omo-senpi/` | **122,133** | Senpi 네이티브 TypeScript 확장 어댑터 (`packages/omo-senpi/plugin/` 로컬-path Pi 패키지) | 어댑터 |
| `packages/omo-codex/` | **8,237** | Codex CLI Light 에디션. npm alias `lazycodex-ai`, 저장소/bin 아이덴티티 `lazycodex` | 어댑터 |
| `packages/omo-native/` | 2,633 | `omo-ai` 배포 런처(npm bin `omo`). 유일한 의존성은 외부 `@code-yeongyu/senpi`. `omo-senpi` 플러그인 페이로드를 스테이징하고 supervision — 엔진 자체는 아님 | 어댑터(런처) |

### 1.2 공유 코어 (`*-core`, 18개)

`packages/AGENTS.md`의 ROLE MAP이 "Core packages 20"으로 세지만, 여기서 18개는 **`-core` 접미사를 가진 것 + `rules-engine`** 이다(`packages/AGENTS.md`: "renamed from `rules-core`" — 접미사 하나를 버린 게 이 패턴의 유일한 예외다).

| 패키지 | LoC | 역할 | 분류 |
|---|---:|---|---|
| `packages/memory-core/` | **14,784** | git MemFS 메모리 엔진, memory 툴, 커밋-HEAD 컴파일러, reflection 상태머신, FTS-lite 검색, 락 | 공유 코어 |
| `packages/skills-loader-core/` | **7,560** | 스킬 로딩·내장 스킬·런타임 스킬·매칭 프리미티브 | 공유 코어 |
| `packages/omo-config-core/` | **4,190** | harness-neutral `omo.json` 스키마, walk-up 로더, 주석 보존 원자적 writer | 공유 코어 |
| `packages/team-core/` | **4,210** | 팀 모드 레지스트리·mailbox·tasklist·state·worktree·tmux 레이아웃 도메인 | 공유 코어 |
| `packages/agents-md-core/` | **160** | AGENTS.md walk-up 탐색 + 인젝션 로직 | 공유 코어 |
| `packages/rules-engine/` | **3,052** | 룰 탐색 + 매칭 엔진 (구 `rules-core`) | 공유 코어 |
| `packages/hashline-core/` | **1,407** | Hashline edit 프리미티브 + diff 헬퍼 | 공유 코어 |
| `packages/boulder-state/` | **1,656** | 워크 추적 상태머신, 분리 스토리지 | 공유 코어 |
| `packages/isolation-core/` | **2,397** | 격리/worktree 실행 도메인 | 공유 코어 |
| `packages/delegate-core/` | **417** | delegate 태스크 선택 + 재시도 | 공유 코어 |
| `packages/utils/` | — | deep-merge, snake-case, frontmatter, file-utils, jsonc-parser | 공유 코어 |
| `packages/model-core/` | — | 모델 해석 파이프라인 (ProviderCache DI 주입) | 공유 코어 |
| `packages/lsp-core/` | — | harness-neutral LSP 엔진, 요청 컨텍스트, 툴 정의, MCP 진입 헬퍼 | 공유 코어 |
| `packages/tmux-core/` | — | tmux 세션·pane·레이아웃·러너 | 공유 코어 |
| `packages/mcp-stdio-core/` | — | MCP 서버용 JSON-RPC stdio 프레이밍 + 디스패치 | 공유 코어 |
| `packages/mcp-client-core/` | — | MCP 클라이언트 라이프사이클, 스킬 내장 MCP 매니저, OAuth | 공유 코어 |
| `packages/telemetry-core/` | — | 텔레메트리 프리미티브 + PostHog 래퍼 | 공유 코어 |
| `packages/claude-code-compat-core/` | — | Claude Code 호환 로더 (플러그인/MCP/커맨드/에이전트) | 공유 코어 |
| `packages/comment-checker-core/` | — | apply-patch 파서 + 바이너리 러너(spawn 주입형) | 공유 코어 |
| `packages/openclaw-core/` | — | OpenClaw 게이트웨이, reply-listener 데몬, 세션 레지스트리 | 공유 코어 |

### 1.3 MCP 패키지

`packages/AGENTS.md`의 ROLE MAP은 MCP 패키지를 **4개**로 센다. 여기에 클라이언트/트랜스포트 반쪽인 `mcp-stdio-core`·`mcp-client-core`까지 세면 MCP 관련 코어는 6개가 된다(둘은 §1.2에 분류).

| 패키지 | 역할 | 빌드 대상 |
|---|---|---|
| `packages/lsp-tools-mcp/` | stdio MCP. `lsp_status` 등 **8개 alias** 제공. `.github/`·`vitest.config.ts`를 갖춘 벤더링 독립 프로젝트 | Node + vitest (`npm --prefix`) |
| `packages/lsp-daemon/` | per-user LSP **데몬**(unix socket / Windows named pipe) + stdio MCP 프록시 + 툴 클라이언트. bin `omo-lsp-daemon` | Node + vitest |
| `packages/ast-grep-mcp/` | stdio MCP `ast_grep`(`search`/`rewrite`/`scan`), `sg` CLI 래핑. bin `omo-ast-grep` | Bun |
| `packages/git-bash-mcp/` | Windows 전용 `git_bash` 툴용 stdio MCP (Codex 에디션 전용) | Bun |

### 1.4 그 밖의 패키지

| 패키지 | LoC | 역할 | 분류 |
|---|---:|---|---|
| `packages/senpi-task/` | **53,551** | DAG 실행 엔진(`src/dag/`), 격리, kernel-tools. **`omo-senpi`만 소비** — harness-neutral이 아니라서 `*-core`가 아니다(`packages/AGENTS.md` 명시) | 어댑터 지원 |
| `packages/shared-skills/` | **39,139** | 18개 `SKILL.md` 번들. §6에서 다룸 | 콘텐츠 |
| `packages/senpi-desktop-tool/` | — | 데스크톱 computer-use 툴层 | Rust 브리지 |
| `packages/senpi-desktop-service/` | — | 데스크톱 서비스 런타임 | Rust 브리지 |
| `packages/senpi-desktop-engine/` | — | 데스크톱 엔진 | Rust 브리지 |
| `packages/senpi-desktop-protocol/` | — | 데스크톱 프로토콜 타입 | Rust 브리지 |
| `packages/senpi-desktop-prelude/` | — | 프리루드 | Rust 브리지 |
| `packages/get-worker/` | — | `src/downloads-rollup.ts` 등 다운로드 통계 집계 | 보조(독립) |
| `packages/web/` | — | `app/`·`components/`·`e2e/`를 가진 프론트엔드(`wrangler.jsonc`) | 보조(독립) |

> 위 5개 `senpi-desktop-*`는 이름이 `packages/` 아래에 있지만 실제 구현은 `crates/`에 있다(§5). `packages/senpi-desktop-*`는 TS ↔ Rust FFI 껍데기다.

### 1.5 플랫폼 바이너리 패키지 (12개)

`oh-my-opencode-{darwin,linux,windows}-{arm64,x64,x64-baseline,musl}` 조합. 전부 `bin/oh-my-opencode.js` + `package.json` **두 파일뿐**이다.

| 패키지 | 설명 (`script/build-binaries.ts`) |
|---|---|
| `oh-my-opencode-darwin-arm64` / `darwin-x64` / `darwin-x64-baseline` | macOS (baseline = AVX2 없음) |
| `oh-my-opencode-linux-arm64` / `linux-arm64-musl` | Linux ARM64 |
| `oh-my-opencode-linux-x64` / `linux-x64-baseline` / `linux-x64-musl` / `linux-x64-musl-baseline` | Linux x64, musl 변형 |
| `oh-my-opencode-windows-arm64` / `windows-x64` / `windows-x64-baseline` | Windows (arm64은 x64 에뮬레이션 경유) |

**중요**: `packages/AGENTS.md`가 명시하듯 "these are not distinct native binaries, and the build does not perform native compilation" — `script/build-binaries.ts`의 `createPlatformLauncherSource()`가 **동일한 Node 스크립트 페이로드**를 12개 패키지에 복사한다. `bun build --target bun-<platform>` 타깃 문자열과 `-baseline`/`-musl` 접미사는 **패키지 선택 + 호환 메타데이터**로 남아 있다.

---

## 2. 레이어링: 왜 `*-core`가 패턴인가

```
            ┌─────────────────────────────────────────┐
 어댑터 층   │ omo-opencode · omo-senpi · omo-codex     │   하니스 API를 직접 만짐
            │ omo-native (런처)                       │
            └─────────────────────────────────────────┘
                              ▲ 의존
            ┌─────────────────────────────────────────┐
 코어 층     │ *-core 18개 (memory/config/rules/team/   │   하니스-neutral 도메인
            │ tmux/skills/model/hashline/…)           │
            └─────────────────────────────────────────┘
                              ▲ 의존
            ┌─────────────────────────────────────────┐
 리프 코어   │ utils · model-core · memory-core ·        │  다른 core에 의존 없음
            │ hashline-core · boulder-state ·           │
            │ isolation-core                           │
            └─────────────────────────────────────────┘
```

핵심 규칙 3가지:

1. **`plugin-interface.ts`만 OpenCode `Plugin` API를 만진다.** `packages/omo-opencode/src/AGENTS.md:133` — "is the **only** layer that talks to OpenCode's `Plugin` API. Every other file goes through it." 하니스 종속 코드는 이 파일 경계 뒤에 가둔다.
2. **코어는 하니스를 모른다.** `packages/memory-core/AGENTS.md`는 "Harness-neutral Letta-Code-parity memory engine"로 자기 정의를 서술한다. 하니스 이름이 코어 소스에 나오면 그건 설계 위반이다.
3. **일부 어댑터 디렉터리는 shim이다.** `packages/omo-opencode/src/AGENTS.md:24` — "Several former implementation directories now act partly as OpenCode-facing shims over extracted Core packages." 즉 `*-core` 분리는 리팩터링 결과물이지 처음부터의 설계가 아니다.

### 왜 굳이 나눴나 — 근거

- 루트 `AGENTS.md` 최상단 배너가 정면으로 밝힌다: *"A MASSIVE MULTI-HARNESS AGENT OS REFACTOR IS IN PROGRESS — WE ARE RESTRUCTURING EVERYTHING TO SUPPORT MULTIPLE AGENT HARNESSES (OPENCODE, CODEX, PI, AND OTHERS)."* 즉 `*-core`는 **다중 하니스 지원이라는 목표를 부과하기 위해 이미 존재하던 단일 하니스 코드를 쪼개서 만든 것**이다.
- 같은 리팩터링의 산물로, 하니스별 QA 게이트도 둘 이상이다: 루트 `AGENTS.md`는 `packages/omo-opencode/`에 `opencode-qa`, `packages/omo-codex/`에 `codex-qa`, `packages/omo-senpi/` + `packages/senpi-task/`에 `senpi-qa` 스킬을 요구한다. 셋 다 "실제 하니스를 구동하고 증거를 디스크에 남겨라"를 요구하므로, 공유 로직은 하니스 밖으로 뺄 수밖에 없었다.
- 단, 경계를 엄격히 지킨 것은 아니다. `packages/omo-opencode/package.json`의 `dependencies`에 **`@oh-my-opencode/omo-codex`가 들어 있다** — 어댑터가 어댑터에 의존하는 예외가 실제로 존재한다(§4 참고).

---

## 3. 부트 플로우 — `omo-opencode`

### 3.1 엔트리 체인

루트 `package.json`의 `main`은 `./dist/index.js`이고, 이 번들은 `packages/omo-opencode/src/index.ts`에서 나온다(`packages/AGENTS.md`: "`omo-opencode` is the **build entry** for the main npm dist (`packages/omo-opencode/src/index.ts` → bundled into root `dist/`)").

`packages/omo-opencode/src/index.ts`는 4줄짜리 shim이다:

```ts
const pluginModule: PluginModule = createPluginModule()
export const omoPlugin = pluginModule.server
export default pluginModule
```

실제 부트 로직은 `createPluginModule()` → **`packages/omo-opencode/src/testing/create-plugin-module.ts`**. 여기가 이번 문서에서 가장 중요한 발견이다: 부트 로직이 `testing/` 디렉터리 밑에 있다. `create-plugin-module.ts`는 `PluginModuleDeps` 인터페이스에 `loadPluginConfig`·`createManagers`·`createTools`·`createHooks`·`createPluginInterface`·`runOpenCodeStartupMigration`·`initializeOpenClaw` 등 **의존성 주입 슬롯**을 정의하고(74–87행), `createPluginModule(overrides: Partial<PluginModuleDeps> = {})`(159행)로 오버라이드를 받는다. 즉 **테스트 프레임워크가 아니라 부트 DI 컨테이너**이고, `src/testing/`은 테스트 전용 디렉터리가 아니다.

### 3.2 부트 시퀀스 (`create-plugin-module.ts`, 159행 이후 순서)

| # | 단계 | 근거 위치 |
|---|---|---|
| 1 | `runOpenCodeStartupMigration({ cwd })` — 레거시 설정 저널 재개. 실패 시 warn 후 계속 | `startup-migration.ts`, 170–186행 |
| 2 | omo 프로세스 스윕 | 190–195행 |
| 3 | `detectDuplicateOmoPlugin` — 중복 등록이면 **빈 객체 `{}`를 반환하고 즉시 종료** | 196–210행 |
| 4 | `detectExternalSkillPlugin` 충돌 경고 | 211–215행 |
| 5 | `injectServerAuthIntoClient` | 216행 |
| 6 | **`loadPluginConfig`** → `pluginConfig` + `startupValidation` | `plugin-config.ts`, 220행 |
| 7 | 텔레메트리 기록, `tui.json` self-heal | 222–235행 |
| 8 | `initLiveServerRoute`, 라이브 부모 wake 라우팅, `warmLiveServerProbe` | 236–238행 |
| 9 | 런타임 보안 스킬 소스 서버 (실패해도 계속) | 240–246행 |
| 10 | `initI18n`, `setAgentSortOrder` | 247–248행 |
| 11 | `initializeOpenClaw` — **동적 `import`** | 251–253행 |
| 12 | 팀 모드 `checkTeamModeDependencies` — `../features/team-mode/*` **동적 import** | 255–262행 |
| 13 | tmux 통합 활성화 검사, `startTmuxCheck` | 274–277행 |
| 14 | `disabled_hooks` → `isHookEnabled`, `safe_hook_creation` 게이트 | 278–281행 |
| 15 | `createFirstMessageVariantGate`, `createRuntimeTmuxConfig`, `createModelCacheState` | 283–288행 |
| 16 | `createManagers({...})` | 289행 |
| 17 | `createTools({...})` (await) | 298행 |
| 18 | `createHooks({...})` | 304행 |
| 19 | `createPluginInterface({ ctx, pluginConfig, firstMessageVariantGate, managers, hooks, tools })` | 317행 |
| 20 | `createPluginDispose({...})` — 해제 경로 | 326행 |

**특징**: 부트 순서가 실패 허용(fail-soft)으로 설계돼 있다. 설정 마이그레이션 실패·텔레메트리 실패·보안 스킬 서버 실패·OpenClaw 실패는 전부 `warn` 후 진행한다. 플러그인 로딩 자체를 막는 유일한 경로는 **3번 중복 플러그인 감지**다.

### 3.3 툴 레지스트리 조립

`packages/omo-opencode/src/create-tools.ts`가 조립한다:

1. `createSkillContext({...})` — 스킬 컨텍스트 결정
2. `createAvailableCategories(pluginConfig)` → `availableCategories` (`src/plugin/available-categories.ts`)
3. `createToolRegistry({...})` → `{ filteredTools, taskSystemEnabled }` (`src/plugin/tool-registry.ts`, `createToolRegistry` 29행)
4. 결과에 `skillContext`의 `mergedSkills`/`availableSkills`/`browserProvider`/`disabledSkills`를 머지

`src/plugin/tool-registry.ts`는 단일 파일이 아니라 **역할별 분할**이다:

| 파일 | 역할 |
|---|---|
| `src/plugin/tool-registry-core-tools.ts` | 항상 켜지는 코어 툴 |
| `src/plugin/tool-registry-factories.ts` | 툴 팩토리 |
| `src/plugin/tool-registry-gated-tools.ts` | 조건부 툴 |
| `src/plugin/tool-registry-team-tools.ts` | 팀 모드 툴 |
| `src/plugin/tool-registry-trimming.ts` | `trimToolsToCap()` — **컨텍스트 상한에 맞춰 툴을 자른다** (84행에서 호출) |

`trimToolsToCap(filteredTools, maxTools)`의 존재 자체가 정보다 — 툴이 많아져서 그냥 다 넘기면 컨텍스트가 터진다는 전제를 코드가 인정한다. 실제 툴 구현은 `src/tools/` 아래 `grep/`, `glob/`, `task/`, `skill/`, `skill-mcp/`, `session-manager/`, `call-omo-agent/`, `background-task/`, `delegate-task/`, `hashline-edit/`, `interactive-bash/`, `look-at/`, `monitor/`, `slashcommand/` 14개 디렉터리.

### 3.4 훅 티어 합성

`packages/omo-opencode/src/create-hooks.ts`는 3개 티어 컴포저를 합친다:

```ts
const core         = createCoreHooks({...})          // src/plugin/hooks/create-core-hooks.ts
const continuation = createContinuationHooks({...})  // src/plugin/hooks/create-continuation-hooks.ts
const skill        = createSkillHooks({...})         // src/plugin/hooks/create-skill-hooks.ts
return { core, continuation, skill, ... }            // CreatedHooks
```

`CreatedHooks` 타입은 `ReturnType<typeof createHooks>`(14행)로 파생되고, `disposeCreatedHooks()`(28행)가 해제 전담이다. 기능 문서의 "5-tier 54~62개"는 `src/hooks/`의 실제 훅 디렉터리 기준이고, 이 3-컴포저는 그 tier들을 **훅 생성 단위**로 다시 묶은 것이다.

### 3.5 플러그인 인터페이스 조립

`packages/omo-opencode/src/plugin-interface.ts`의 `createPluginInterface({ ctx, pluginConfig, firstMessageVariantGate, managers, hooks, tools })`가 반환하는 `PluginInterface`에 하니스 핸들러가 배선된다. 각 핸들러는 `src/plugin/`의 개별 팩토리다:

| 핸들러 키 | 팩토리 |
|---|---|
| `chat.params` | `createChatParamsHandler` (내부에 `applyAgentVariant`) |
| `chat.headers` | `createChatHeadersHandler` |
| `command.execute.before` | `createCommandExecuteBeforeHandler` |
| `chat.message` | `createChatMessageHandler` |
| `experimental.chat.messages.transform` | `createMessagesTransformHandler` |
| `experimental.chat.system.transform` | `createSystemTransformHandler` |
| 이벤트 | `createEventHandler` |
| 툴 정의/실행 | `createToolDefinitionHandler`, `createToolExecuteBeforeHandler`, `createToolExecuteAfterHandler` |

`packages/omo-opencode/src/AGENTS.md` 기준 총 14개 OpenCode 훅 핸들러가 배선된다. 12개가 `plugin-interface.ts`에 있고, 나머지 2개(`experimental.session.compacting`, `experimental.compaction.autocontinue`)는 **`create-plugin-module.ts` 안에서 직접 배선**된다. 즉 "테스트 디렉터리"가 아니라 부트 컨테이너라는 사실의 또 다른 증거다.

### 3.6 설정 해석

- 스키마·로더·writer: `packages/omo-config-core/src/` (harness-neutral)
- 어댑터 쪽 래퍼: `packages/omo-opencode/src/plugin-config.ts`, 검증은 `plugin-config.prototype-pollution.test.ts` / `plugin-config.test.ts`로 분리
- 해석 결과는 부트 시 **한 번만** 계산되어 `pluginConfig`로 고정되고, 이후 `isHookEnabled(hookName)` 클로저(279–280행)로 훅 게이트에만 쓰인다
- 프로필/머지/마이그레이션 규칙은 기능 문서 §"설정" 참조

---

## 4. 의존 방향 — 어댑터 → 코어, 역방향 없음

### 4.1 `omo-opencode` → 코어 (실측 import 카운트)

`packages/omo-opencode/src/` 전체에서 `from "@oh-my-opencode/*"` 패턴 집계:

| 코어 | import 수 | 코어 | import 수 |
|---|---:|---|---:|
| `utils` | 49 | `delegate-core` | 4 |
| `model-core` | 42 | `boulder-state` | 4 |
| `tmux-core` | 32 | `agents-md-core` | 3 |
| `hashline-core` | 21 | `team-core` | 1 |
| `omo-config-core` | 20 | `shared-skills` | 1 |
| `rules-engine` | 19 | `openclaw-core` | 1 |
| `telemetry-core` | 13 | | |
| `prompts-core` | 9 | | |
| `comment-checker-core` | 6 | | |

`packages/omo-opencode/package.json`의 `dependencies`도 정확히 일치한다(`@opencode-ai/plugin`·`@opencode-ai/sdk`는 peer가 아니라 **직접 의존** — 어댑터가 하니스를 깔고 들어간다).

### 4.2 역방향은 없다

`packages/*/src/` 전체에서 `from "../omo-opencode"` 계열(상대 경로로 어댑터를 역참조)을 grep한 결과 **0건**이다. 어떤 코어도 어댑터를 모른다.

### 4.3 코어 → 코어 (내부 계층)

| 코어 | 의존하는 코어 | 성격 |
|---|---|---|
| `utils` | (없음) | 리프 |
| `memory-core` | (없음) | 리프 — 최대 규모(14,784 LoC)인데 의존이 0 |
| `hashline-core` | (없음) | 리프 |
| `model-core` | (없음) | 리프 |
| `boulder-state` | (없음) | 리프 |
| `isolation-core` | (없음) | 리프 |
| `omo-config-core` | `utils` | 1단계 |
| `agents-md-core` | `rules-engine` | 1단계 |
| `team-core` | `tmux-core` | 1단계 |
| `skills-loader-core` | `model-core`, `shared-skills`, `utils` | 1단계 |
| `senpi-task` (어댑터 지원) | `delegate-core`, `isolation-core`, `omo-config-core`, `utils` | 어댑터보조 |

**해석**: 코어 내부 그래프는 깊이 1~2에 깔린다. `memory-core`가 리프라는 게 특히 의미있다 — 메모리 엔진 14.8k LoC가 어떤 공유 유틸리티도 빌려가지 않고 독립이라는 뜻이고, 이게 없으면 Senpi/Codex 포팅 때 메모리 엔진을 통째로 다시 써야 한다.

### 4.4 다만 경계는 순수하지 않다

`packages/omo-opencode/package.json`의 `dependencies`에 **`@oh-my-opencode/omo-codex`** 가 들어 있다. 즉 어댑터가 어댑터에 의존하는 실제 사례가 있다(추정: 공유 커맨드/설정 마이그레이션 재사용). `senpi-task`도 어댑터(("Senpi-coupled")이면서 `omo-senpi`만 소비한다. 이 둘이 `*-core`가 아닌 이유를 설명해 준다 — **하니스 중립이 아니라면 코어로 승격시키지 않는다**는 규칙이 실제로 작동하고 있다는 증거다.

---

## 5. 빌드 & 툴체인

### 5.1 TypeScript/Bun

| 항목 | 값 | 근거 |
|---|---|---|
| 런타임/번들러 | **Bun 1.4.2** | `.github/workflows/ci.yml:181` `oven-sh/setup-bun@v2` + `bun-version: "1.4.2"`, `bun-types: "1.4.2"` |
| 타입체커 | **`tsgo`** (`@typescript/native-preview` 7.0.0-dev) | 루트 `typecheck` 스크립트 전부 `tsgo --noEmit` |
| `typescript` | `^7.0.2` (dev) | 루트 `devDependencies` |
| 테스트 | `bun test` + `bunfig.toml` preload(`./test-env.ts`, `./test-setup.ts`) | 루트 `test` 스크립트 |
| lint | Biome (`packages/web/biome.json`, 벤더링된 `lsp-tools-mcp`/`lsp-daemon`) | 개별 프로젝트 |

`typecheck:packages`가 **34개 tsconfig를 순차 `&&`로 나열**돼 있다(각 워크스페이스 + `omo-codex/plugin/shared`). 별도 프로젝트 그래프 도구 없이 수동으로 관리한다.

### 5.2 오케스트레이터: turbo/bazel 없음, 자체 빌드 노드 그래프

`turbo.json`, `BUILD*`, `nx.json` **없다**. 대신 `script/build.ts` + `script/build-nodes.ts`가 커스텀 DAG다.

- `script/build.ts`는 각 빌드 노드를 `spawn`하고, POSIX에선 자식 프로세스 그룹을 만들어 실패 시 `killTree()`로 손자 프로세스까지 `SIGKILL`(Windows는 `taskkill /T /F` fallback)
- 노드 간 실제 의존 간선은 **딱 3개**다(`script/build.ts` 주석): `index → node-require-shim`(dist/index.js 패치), `materialize → skills-assets`, `materialize → codex-plugin`(둘 다 material화된 `packages/shared-skills/skills`를 읽음). `materialize`는 정확히 한 번 먼저 실행되고, `OMO_SKIP_MATERIALIZE=1`로 건너뛸 수 있다
- 그래프와 consumer profile은 `script/build-nodes.ts`에 있어 **빌드를 돌리지 않고도** 검사 가능

### 5.3 Rust 크레이트 — `crates/` 10개

`Cargo.toml`의 workspace members (root `Cargo.toml`):

| 크레이트 | 역할 |
|---|---|
| `senpi-desktop-core` | 크로스 플랫폼 데스크톱 추상화 |
| `senpi-desktop-safety` | 안전 정책(허용/차단 판정) |
| `senpi-desktop-session` | 세션 상태 |
| `senpi-desktop-engine` | 엔진 |
| `senpi-desktop-backend-fake` | 테스트용 백엔드 |
| `senpi-desktop-backend-macos` | macOS 백엔드 |
| `senpi-desktop-backend-x11` | X11 백엔드 |
| `senpi-desktop-backend-wayland` | Wayland 백엔드 |
| `senpi-desktop-backend-atspi` | **Linux 접근성(AT-SPI) 백엔드** |
| `senpi-desktop-backend-win32` | **Win32 백엔드** |

모두 `crate-type = ["rlib"]`(공유 라이브러리). `edition = "2021"`, `version = "0.0.0"`(워크스페이스 비공개 버전), `license = "MIT"`. 워크스페이스 `Cargo.lock`이 루트에 있어 TS `bun.lock`과 나란히(**polyglot 락 파일 2개**) 관리된다. `bun run test:desktop`가 `script/check-cargo-pinned-deps.mjs`로 **고정(deps pin) 검증**부터 돌린다 — 워크스페이스 `[workspace.dependencies]`에 `image = "=0.25.10"`처럼 `=` 정밀 고정된 항목이 있다.

**왜 Rust인가**: 백엔드 5종(macOS/X11/Wayland/AT-SPI/Win32)을 다뤄야 한다. Node로는 Win32 UI Automation과 Linux AT-SPI 접근성 브리지를 안정적으로 만들 수 없고, 크로스 컴파일 매트릭스는 Rust 쪽이 훨씬 싸다. 반면 **프레젠테이션/오케스트레이션은 TS에 남겼다** — 그래서 `packages/senpi-desktop-*` 5개(FFI 껍데기)와 `crates/` 10개(실제 구현)가 나뉜다.

### 5.4 플랫폼 바이너리 12개가 말해주는 릴리스 엔지니어링

12개 패키지가 함의하는 것:

1. **설치 시점 런타임 선택**. `postinstall.mjs`가 `process.platform`/`process.arch`로 후보 패키지명을 만들어 순차 시도하고(`"No platform binary package installed. Tried: ..."`), `bin/platform.js`의 `getBinaryPath(pkg, platform)`로 해석한다.
2. **버전 스큐 감지**. `postinstall.mjs`는 메인 패키지 버전과 선택된 플랫폼 패키지 버전을 비교해 불일치하면 배너가 stale임을 경고하고 `npm i -g oh-my-opencode@X oh-my-opencode-<plat>@X` 복구 명령을 제시한다. **12개 슬롯의 버전을 메인 버전(5.1.13)과 수동으로 동기화해야 한다**는 운영 부담이 붙는다. 루트 `optionalDependencies` 12줄이 그 동기화 대상이다.
3. **이중 툴체인 빌드 스크립트**. `build:lsp-tools-mcp`·`build:lsp-daemon`은 `npm ci`(= Node/npm), `build:ast-grep-mcp`·`build:git-bash-mcp`은 `bun run --cwd`(= Bun). 같은 레포에 npm 락과 bun 락이 공존한다.
4. **`-baseline`(AVX2 없음) / `-musl`(libc) 분기는 CPU·libc 호환 라우팅 메타데이터**다. `packages/AGENTS.md`가 "not distinct native binaries, the build does not perform native compilation"이라 명시하므로, 현재는 **분기를 유지한 채 컴파일 부재**인 상태 — 즉 네이티브 전환을 대비한 **확장 슬롯**이다.
5. **플랫폼별 릴리스 워크플로 분리**. `packages/AGENTS.md`: "Published by the `publish-platform.yml` workflow." 메인 npm 배포와 플랫폼 배포가 별도 파이프라인이다.
6. **교차 fallback**: `windows-arm64`은 x64 에뮬레이션을 타고, Bun이 못 돌면 Node fallback이 있다.

---

## 6. `shared-skills` (39k LoC) — 코드가 아니라 콘텐츠

`packages/shared-skills/`는 **가장 큰 패키지 중 하나인데 `src/` 디렉터리가 없다**(테스트 제외 39,139 LoC는 `skills/**/*.md`와 `upstreams/` 산출물에서 나온다).

```
packages/shared-skills/
├── skills/          # 18개 SKILL.md 번들
├── upstreams/       # designpowers, open-design, taste-skill, ui-ux-pro-max
├── scripts/
├── index.mjs · index.d.ts
├── skill-source-filter.mjs / .d.ts
├── depersonalization-gate.d.ts
└── stage-omowright-runtime.mjs
```

`skills/` 18개: `ast-grep`, `browser`, `coding-agent-sessions`, `data-scientist`, `debugging`, `frontend`, `git-master`, `init-deep`, `lsp-setup`, `programming`, `refactor`, `remove-ai-slops`, `review-work`, `ultimate-browsing`, `ulw-execute`, `ulw-plan`, `ulw-research`, `visual-qa`.

왜 이게 아키텍처상 특별한가:

1. **의미론이 다른 유일한 패키지다.** 다른 37개는 실행되는 로직이지만, 이건 **프롬프트 아티팩트**다. `packages/AGENTS.md`는 role map에서 이걸 "Skills 1"으로 별도 분류하고, "cross-harness SKILL.md bundle shared between OMO and Codex"라고 설명한다. 즉 **두 어댑터가 같은 콘텐츠를 공유하는 유일한 물건**이고, 그래서 `skills-loader-core`가 이 패키지(`skills-loader-core` → `shared-skills` import)에 의존한다.
2. **빌드 그래프의 병목이다.** `script/build.ts` 주석에 따르면 `materialize` 노드가 정확히 한 번 실행되고, **두 소비자(`skills-assets`, `codex-plugin`)가 그 materialization 결과를 읽는다**. 콘텐츠 패키지 하나가 TS 빌드 그래프의 중심 허브다. `build:materialize-frontend` = `materialize-shared-upstreams.mjs --strict`.
3. **npm `files` 배열에 통째로 실린다.** 루트 `package.json`의 `files`에 `packages/shared-skills/index.mjs`·`skill-source-filter.mjs`·`skills`가 명시되고, `build:shared-skills-assets`는 `rm -rf dist/skills && cp -R packages/shared-skills/skills dist/skills`로 루트 dist에 복사한다.
4. **테스트는 있다.** `browser-skill-engine.test.ts`, `frontend-stylegallery-routing.test.ts`, `provenance-gate.test.ts`, `ultimate-browsing-runtime-pins.test.ts` 등 — 콘텐츠에도 provenance 게이트와 라우팅 검증이 붙는다. `bunfig.toml`은 `packages/shared-skills/upstreams/**`만 `pathIgnorePatterns`로 테스트 대상에서 뺀다.
5. **이중 스킬 전략의 절반.** 기능 문서의 "TS 내장 14개 + shared-skills 18개" 중 이 패키지가 뒤쪽 18개를 담당한다.

---

## 7. 핵심 통찰

**① 공유 코어의 동기적 짝은 "증거 파일"이다.**
`*-core` 분리로 로직을 중립화했지만, 그 반대편 비용으로 **하니스별 QA 하네스 3개**(`opencode-qa`, `codex-qa`, `senpi-qa`)와 `.omo/evidence/` 규제가 생겼다. 루트 `AGENTS.md`는 이를 "NO EVIDENCE FILE == NO QA == NO COMMIT == NO PUSH"로 강제하고, `script/tracked-evidence-paths-audit.test.ts`가 증거 파일이 git에 들어가면 빌드를 실패시킨다(#8703). **일반적인 단일 하니스 레포에는 이 계층이 아예 없다.** 추상화 계층 하나를 더할 때마다 검증해야 할 대상 면(surface)이 하나 더 생기는데, OmO는 그 면을 하니스별로 명시적으로 나눠 관리한다.

**② 12개 플랫폼 패키지는 현재 0바이트 추상화다.**
`packages/AGENTS.md`가 스스로 "these are not distinct native binaries, and the build does not perform native compilation"이라고 적고 있다. 12개 디렉터리는 **미래 이주용 슬롯**이고, `-baseline`(AVX2)·`-musl`(libc) 접미사는 그 이주 경로를 미리 정의한 것이다. 릴리스 비용(`optionalDependencies` 12줄 수동 버전 동기화 + 별도 `publish-platform.yml`)은 이미 지불하고 있고, native 컴파일 도입 여부는 아직 결정되지 않았다. **빌드 시스템이 배포 대상 행렬을 먼저 확정하고 구현을 나중에 채우는 설계**다.

**③ 코어 그래프는 얕고 넓다 — 얕은 위상이 의도적이다.**
내부 의존이 깊이 1~2에서 끝나고 리프 6개(`utils`, `memory-core`, `hashline-core`, `model-core`, `boulder-state`, `isolation-core`)가 바닥을 받친다. 14.8k LoC인 `memory-core`가 의존 0인 것은 우연이 아니다 — ** Senpi/Codex/OpenCode 3개 포팅에서 그대로 재사용돼야 하는 코드가 다른 공유 모듈에 얽혀 있으면 안 된다**는 제약이다. 반대로 어댑터 쪽은 `omo-opencode` 126k LoC에 `tmux-core`만 32회 import하는 등 얇게 여러 코어를 긁는 구조다.

**④ 부트 로직이 `src/testing/`에 산다 — 테스트 격리보다 DI가 먼저다.**
`packages/omo-opencode/src/testing/create-plugin-module.ts`가 진짜 엔트리工厂이고, `PluginModuleDeps` 슬롯으로 `loadPluginConfig`·`createManagers`·`createTools`·`createHooks`·`createPluginInterface`·`runOpenCodeStartupMigration`을 주입받는다. 두 개의 compaction 훅 핸들러도 여기가 아니라 이 파일에서 배선된다. 126k LoC 규모에서 부트 20단계를 각각 테스트하려면 **부트 컨테이너 자체가 테스트 대역 불가능한 클래스여야 한다**는 판단이고, 그 결과 디렉터리 이름이 의도와 어긋나게 되었다. 구조를 읽을 때 `testing/`을 "테스트 전용"으로 건너뛰면 부트 플로우 전체를 놓친다.

**⑤ `omo-codex`가 `omo-opencode`의 의존이다 — 규칙의 예외가 남는다.**
`packages/omo-opencode/package.json`이 `@oh-my-opencode/omo-codex`를 직접 의존한다. `senpi-task`(53,551 LoC, `omo-senpi`만 소비)도 같은 이유로 `*-core`가 아니다. **"하니스 중립이면 core, 아니면 adapter"라는 규칙이 문서가 아니라 `package.json`에 강제되고 있으며, `omo-codex`만 그 규칙을 어긴 사례가 남아 있다.** 리팩터링이 진행 중이라는 루트 `AGENTS.md` 배너("DO NOT TRUST THE STRUCTURE BELOW AS STABLE")의 이유가 여기에 있다.

---

## 부록 — 경로 색인

| 관심사 | 경로 |
|---|---|
| 모노레포 역할 맵 | `packages/AGENTS.md` |
| 어댑터 내부 구조 | `packages/omo-opencode/src/AGENTS.md` |
| 부트 DI 컨테이너 | `packages/omo-opencode/src/testing/create-plugin-module.ts` |
| OpenCode 훅 배선 | `packages/omo-opencode/src/plugin-interface.ts` |
| 툴 레지스트리 | `packages/omo-opencode/src/plugin/tool-registry.ts` + `tool-registry-{core-tools,factories,gated-tools,team-tools,trimming}.ts` |
| 훅 티어 컴포저 | `packages/omo-opencode/src/create-hooks.ts` → `src/plugin/hooks/create-{core,continuation,skill}-hooks.ts` |
| 툴 구현 | `packages/omo-opencode/src/tools/` (14 디렉터리) |
| 설정 해석 | `packages/omo-config-core/src/` · `packages/omo-opencode/src/plugin-config.ts` |
| 빌드 DAG | `script/build.ts` · `script/build-nodes.ts` |
| 플랫폼 바이너리 생성 | `script/build-binaries.ts` (`createPlatformLauncherSource`) |
| 설치 시 런타임 선택 | `postinstall.mjs` · `bin/platform.js` |
| Rust 워크스페이스 | `Cargo.toml` · `crates/` (10 크레이트) |
| 스킬 콘텐츠 | `packages/shared-skills/skills/` (18) · `packages/shared-skills/upstreams/` |
| QA 게이트 규칙 | 루트 `AGENTS.md` |
