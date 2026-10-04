# oh-my-openagent (OmO) 4개 어댑터 타깃 비교

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13`
> 상위 문서: [oh-my-openagent.md](../oh-my-openagent.md) — 이 문서는 **"무엇이 실제로 다른가"**만 다룬다.

## 한 줄 요약

| 패키지 | npm 이름 | 실체 | src LoC |
|---|---|---|---|
| `packages/omo-opencode/` | `@oh-my-opencode/omo-opencode` (private) | OpenCode **플러그인** (`PluginModule`) | 126,239 |
| `packages/omo-senpi/` | `@oh-my-opencode/omo-senpi` (private) | **Senpi 네이티브 확장**(TypeScript Extension) | 76,079 |
| `packages/omo-codex/` | `@oh-my-opencode/omo-codex` (private) | Codex **설치러 + 벤더링된 플러그인 namespace** | 8,237 |
| `packages/omo-native/` | **`omo-ai`** (공개, bin `omo`) | 위 Senpi 플러그인을 **구동하는 런처** | 0 |

`docs/guide/installation.md`가 이 구분을 "**three editions** of the same product: two plugins that load into a host you already run, plus one standalone edition"로 규정한다. 즉 **플러그인 2개(Ultimate/Light) + 스탠드얼론 1개(Native)**.

---

## 1. 각 타깃은 실제로 무엇인가

### `omo-opencode` — OpenCode 하니스에 로드되는 플러그인

`packages/omo-opencode/src/index.ts` 전체가 이렇다:

```ts
import type { PluginModule } from "@opencode-ai/plugin"
const pluginModule: PluginModule = createPluginModule()
export const omoPlugin = pluginModule.server
export default pluginModule
```

- `@opencode-ai/plugin` 타입의 `PluginModule`을 default export 한다. 런타임 코드가 아니라 **호스트가 불러가는 플러그인 모듈**.
- `src/plugin-interface.ts`가 OpenCode 훅 이벤트를 그대로 구현한다: `chat.params`, `chat.headers`, `chat.message`, `command.execute.before`, `experimental.chat.messages.transform`, system transform, `event`, tool definition, `tool.execute.before/after`.
- 초기화는 `src/AGENTS.md`에 7단계로 문서화돼 있다: `installAgentSortShim` → `initConfigContext` → `detectExternalSkillPlugin` → `injectServerAuthIntoClient` → `loadPluginConfig` → `initializeOpenClaw`/`checkTeamModeDependencies` → `createManagers/Tools/Hooks/PluginInterface`.
- **빌드 엔트리포인트다**: `packages/AGENTS.md` — "`omo-opencode` is the **build entry** for the main npm dist (`packages/omo-opencode/src/index.ts` → bundled into root `dist/`)". 루트 `package.json`의 `main`은 `./dist/index.js`.
- Codex 에디션 문서도 이걸 명시한다: Light edition "has **no OpenCode agent registry or `team_*` tool family**" — 즉 그 레지스트리는 이 타깃에만 있다.

### `omo-senpi` — OmO 자체 네이티브 런타임("Senpi")

`packages/omo-senpi/src/index.ts`는 2줄이다:

```ts
export const omoSenpiAdapterPackageName = "@oh-my-opencode/omo-senpi"
export * from "./install"
```

진짜 진입점은 두 곳이다:

| 경로 | 역할 |
|---|---|
| `src/extension/` | Senpi `ExtensionAPI` 합성 계층. `compose.ts`, `bundled-index.ts`, `component-list.ts`가 컴포넌트를 등록 순서대로 배선. API surface 검증, 전역/컴포넌트 disable 플래그, 방어적 배선 담당 |
| `src/components/` | 실제 동작하는 25개 라이브 컴포넌트 (`src/extension/component-list.ts` 등록 순서) |

`packages/omo-senpi/AGENTS.md` 첫 문단이 성격을 규정한다:

> "This package is **adapter-only**. It may depend on harness-neutral core packages plus the Senpi-coupled `@oh-my-opencode/senpi-task` engine, but those packages **must not import Senpi, Pi packages, or this adapter** through their harness-neutral entrypoints. The Senpi runtime boundary stays here."

등록 컴포넌트 25개: `config-startup`, `model-profile`, `bundled-skills`, `skill-commands`, `native-badge`, `onboarding`, `init-deep-advisor`, `telemetry`, `ultrawork`, `skill-pointers`, `ulw-execute-continuation`, `ulw-loop`, `todo-fanout-reminder`, `git-master`, `fallback-architect`, `ast-grep`, `builtin-mcps`, `lsp`, `x-search`, `computer-use`, `comment-checker`, `task`, `thread`, `memory`, `config-watch`. (헬퍼 `config-resolution`·`agent-home`은 등록 컴포넌트가 아님.)

Senpi 전용 스킬 10개도 이 어댑터가 직접 Authors 한다 — `packages/omo-senpi/skills/`: `dag-library`, `give-me-tips`, `hyperplan`, `init-deep`, `mass-ulw`, `onboarding`, `ultrawork`, `ulw-loop`, `ulw-plan`, `ulw-research`. AGENTS.md는 이들이 "**authored directly against the Senpi tool surface** (not ported from Codex or the shared pool)"라고 명시한다.

### `omo-codex` — Codex Light 어댑터 (설치러 중심)

`packages/omo-codex/src/index.ts`:

```ts
export * from "./telemetry"
export * from "./install"
```

`package.json` description: "Codex harness adapter for oh-my-openagent. **Vendored Codex plugin namespace (`omo`) + TypeScript installer + telemetry.**" 즉 `src/`는 **런타임이 아니라 설치러**, 실제 플러그인은 `packages/omo-codex/plugin/`에 벤더링돼 있다.

- `plugin/` = 벤더링된 Codex 플러그인 namespace. 패키지명 `@sisyphuslabs/omo-codex-plugin`, marketplace `sisyphuslabs` / 플러그인 `omo`, `.codex-plugin/plugin.json`에 23개 훅 JSON을 배선.
- 컴포넌트 10개 + 특수 3개: `comment-checker`, `git-bash`, `lazycodex-executor-verify`, `lsp`, `rules`, `teammode`, `telemetry`, `ultrawork`, `ulw-loop`, `ulw-execute-continuation` / 특수: `bootstrap`(워크스페이스 밖 단독 빌드), `test-support`, `lcx`(skills-only carrier).
- Codex 훅 이벤트는 7개뿐이다: `SessionStart` / `UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `PostCompact` / `Stop` / `SubagentStop`.
- 배포: `npx lazycodex-ai install` → `~/.codex/plugins/cache/sisyphuslabs/omo/<version>/` + `~/.codex/config.toml`에 `omo@sisyphuslabs` 활성화 + `~/.codex/agents/`에 agent TOML 12개.
- `omo-codex/README.md`가 OpenCode 타깃을 직접 명시: "The Codex plugin bundle includes Context7 as a default MCP in its `.mcp.json` … The same plugin-scoped MCP manifest also bundles `grep_app`, `git_bash`, and `lsp`."

### `omo-native` — Senpi를 띄우는 런처 (src LoC 0)

- `package.json`: `name: "omo-ai"`, `bin: { omo: "bin/omo.js" }`, description "OmO Native - the standalone omo command with the OMO extension built in: bun add -g omo-ai", **의존성 1개** = `@code-yeongyu/senpi@2026.10.3`.
- `src/` 디렉터리가 없다. `build-info.ts`, `compile-entry.ts`, `category-coverage-entry.ts`, `claude-code-doctor.ts` 같은 TS 파일은 전부 패키지 최상위에 있고, `bun run build:omo-native`(`script/build-omo-native.ts`)이 `bin/lib/` 런타임으로 번들하는 **스테이징 진입점 shim**이다. 그래서 src LoC가 0이다.
- `bin/omo.js` 본체:

```js
ensureBunBinShim({ scriptPath })
const reexeced = await maybeReexecUnderBun({ scriptPath })
if (!reexed) {
  if (process.argv[2] === "setup") await runSetup(process.argv.slice(3))
  else await runLauncher()
}
```

- `packages/omo-native/AGENTS.md`: "The launcher in `bin/` runs the **exact-pinned** `@code-yeongyu/senpi` CLI with `--extension <pkgRoot>/plugin`, where `plugin/` is the staged omo-senpi plugin payload produced by `bun run build:omo-native` (gitignored, never committed)."

**결론**: `omo-senpi`(플러그인 payload) → `omo-native`(엔진 구동기) 관계이며, Native 에디션 = Senpi 어댑터 + Senpi 엔진 + 런처 3종 세트다.

---

## 2. 기능 매트릭스

`✅` = 코드에서 검증됨 / `➖` = 이 타깃에 부재(설계상) / `❓` = 확인 필요

### 2.1 ast-grep — 타깃마다 전달 방식이 다르다

| 타깃 | 형태 | 근거 경로 |
|---|---|---|
| omo-opencode | **스킬 + `sg` 바이너리 프로비저닝 훅** | `packages/shared-skills/skills/ast-grep/`, `packages/omo-opencode/src/hooks/ast-grep-sg-provision/` |
| omo-senpi | **MCP 서버** (`ast-grep-mcp`, stdio, `search`/`rewrite`/`scan`) | `packages/omo-senpi/src/components/ast-grep/` (AGENTS.md: "registers the ast-grep MCP (stdio)") |
| omo-codex | **스킬 + 런타임에 `sg` 프로비저닝** | `omo-codex/README.md`: "The ast-grep capability ships as the `ast-grep` skill and provisions `sg` into the Codex runtime" |
| omo-native | Senpi 경유로 위 `omo-senpi` 쪽 상속 | `packages/omo-senpi/src/components/ast-grep` |

검증 근거: `packages/omo-opencode/package.json`에 `ast-grep` 문자열은 **0건**(MCP 의존성 없음). 반대로 `packages/omo-senpi/package.json`은 `@oh-my-opencode/ast-grep-mcp: workspace:*`를 명시 의존한다. `packages/AGENTS.md`도 `ast-grep-mcp`를 "staged into the Senpi plugin runtime by `build:senpi-plugin`"으로 명시 — 즉 **ast-grep MCP는 Senpi 전용**이다.

### 2.2 LSP — 세 타깃 모두 있지만 형태가 다르다

| 타깃 | 형태 | 도구 수 | 근거 경로 |
|---|---|---|---|
| omo-opencode | tier-1 **MCP `lsp`** (로컬 stdio, `lsp-tools-mcp`) | 8 alias | `packages/omo-opencode/src/mcp/lsp.ts`, `createBuiltinMcps()`의 `mcps.lsp`, `packages/lsp-tools-mcp/` |
| omo-senpi | **직접 툴 등록** + 공유 **lsp-daemon**으로 아웃프로세스 포워딩 | 6 | `packages/omo-senpi/src/components/lsp/adapter/descriptors.ts`, `daemon-runtime.ts`, `@code-yeongyu/lsp-daemon` |
| omo-codex | **plugin-scoped MCP `lsp`** + post-edit 훅 | 미확인 | `packages/omo-codex/plugin/components/lsp/`, `plugin/.mcp.json` |
| omo-native | Senpi 경유 | = senpi | 동일 |

- OpenCode 쪽 8 alias는 `packages/AGENTS.md` — `lsp_status`, `lsp_diagnostics`, `lsp_goto_definition`, `lsp_find_references`, `lsp_symbols`, `lsp_prepare_rename`, `lsp_rename`, `lsp_install_decision`.
- Senpi 쪽은 6개(위 descriptors.ts 행)이며 `omo-senpi/AGENTS.md`가 **왜 다른지** 명시한다: "**No MCP-shaped LSP** (the `lsp` component registers direct tools)" / `components/lsp/AGENTS.md`: "**Every tool call is forwarded out of process: the component never spawns language servers itself.**" 즉 데몬 경유 강제가 Senpi 쪽 차이의 설계 이유다.
- `lsp-daemon`은 `packages/omo-senpi/package.json`(`@code-yeongyu/lsp-daemon: file:../lsp-daemon`)만 참조한다. **OpenCode 타깃은 lsp-daemon에 연결되지 않는다**(inline stdio MCP).

### 2.3 Team Mode / computer-use / senpi-desktop-*

| 기능 | omo-opencode | omo-senpi | omo-codex | 근거 |
|---|---|---|---|---|
| `team_*` 툴 패밀리 | ✅ 12개(conditional) | ✅ 6개 lead-only | ➖ 없음 | `packages/omo-opencode/src/tools/`(team 계열, `delegate-task/constants.ts`), `packages/omo-senpi/src/components/task`(team_create/team_delete/task_create/task_get/task_list/task_update) |
| Team Mode(deps `git`/`tmux`/`~/.omo/teams/`) | ✅ `checkTeamModeDependencies()` | ✅ senpi-task 엔진 위 | ⚠️ `teammode` 컴포넌트는 존재하나 "**script-and-skill-driven**" | `packages/omo-opencode/src/index.ts`, `packages/omo-codex/plugin/components/teammode/` |
| computer-use (데스크톱 제어) | ➖ | ✅ | ➖ | `packages/omo-senpi/src/components/computer-use/` |
| `senpi-desktop-*` (Rust computer-use 크레이트) | ➖ | ✅ 3개 | ➖ | `packages/omo-senpi/package.json`: `senpi-desktop-engine`/`-service`/`-tool` (+ repo에 `senpi-desktop-protocol`/`-prelude`) |
| `/computer` (on/off/status/stop/resume) | ➖ | ✅ | ➖ | `packages/omo-senpi/AGENTS.md` computer-use 절 |
| `x_search` (자격증명 게이트) | ❓ | ✅ | ❓ | `packages/omo-senpi/src/components/x-search/`, `plugin/skills-conditional/x-search/` |

`omo-senpi/AGENTS.md`가 computer-use 등록 조건을 못 박는다: 지원 호스트(macOS/Linux/Windows)에서 `computer.enabled`가 true일 때만 등록하고, prelude/doc 텍스트는 **어떤 번들에도 넣지 않으며** `senpi-desktop-prelude`가 `assets.generated.json`을 생성해 memoized getter로 1회 읽는다. 실제 엔진은 `runtime.ts` → `extensions/omo-computer-use.js`로 **지연 import**한다(#9113).

Codex에는 `teammode` 컴포넌트가 있으나 `installation.md`가 "It has **no** OpenCode agent registry or `team_*` tool family, but ships Codex-native agent roles and the script-and-skill-driven `teammode` component"라고 해 명시적으로 축소판임을 밝힌다.

### 2.4 MCP 티어

| 티어 | 소스 | omo-opencode | omo-senpi | omo-codex |
|---|---|---|---|---|
| 1. 내장 | 어댑터 코드가 직접 등록 | ✅ 4개: `websearch`(조건부), `context7`, `grep_app`, `lsp` | ✅ 2개: `context7`, `grep_app` (둘 다 `type: "http"`, `lifecycle: "lazy"`) | ✅ 4개: `context7`, `grep_app`, `git_bash`(Windows만 enabled), `lsp` |
| 2. Claude Code 호환 | `.mcp.json` | ✅ `claude-code-compat-core` 의존 | ✅ `src/components/claude-code` | ✅ `plugin/.mcp.json` |
| 3. 스킬 내장 | SKILL.md frontmatter | ✅ `skill-mcp-manager` | ✅ 번들 스킬 경유 | ✅ `sync-skills` 파이프라인 |

- OpenCode 내장 목록은 `packages/omo-opencode/src/mcp/index.ts` `createBuiltinMcps()` 코드에서 직접 읽힌다: `websearch`(→`createWebsearchConfig`, config 없으면 null), `context7`, `grep_app`, `lsp`(→`createLspMcpConfig`).
- **Senpi에 websearch MCP가 없는 이유**는 `omo-senpi/AGENTS.md`가 명시: "**No websearch server (senpi ships `websearch`/`webfetch` builtins)**". 즉 호스트 내장 기능과 중복이라 뺀다.
- Senpi의 `context7`은 `CONTEXT7_API_KEY`가 실질값일 때만 `auth: "bearer"` + `bearerTokenEnv`으로 전환되고, `headers`에 literal 토큰을 절대 넣지 않는다. 신뢰된 `mcp.json` 항목이 같은 이름이면 extension 선언보다 이긴다 → 서버별 off 스위치.

### 2.5 훅 시스템 티어

| | omo-opencode | omo-senpi | omo-codex |
|---|---|---|---|
| 이벤트 수 | 20 hook 포인트(OpenCode) | Senpi `input`/`session_start`/`agent_end`/`agent_settled`/`tool_result`/`message_end`/`model_select`/`session_compact`/`session_shutdown`/`resources_discover` | 7개 (`SessionStart`/`UserPromptSubmit`/`PreToolUse`/`PostToolUse`/`PostCompact`/`Stop`/`SubagentStop`) |
| 티어 구성 | **5-tier**: Session 24 / Tool Guard 17→18 / Transform 4→6 / Continuation 7 / Skill 2 (+ team-mode 직접 이벤트 핸들러 4) | 컴포넌트별 국소 훅 (컴포넌트 1개 = 훅 1~2개) | 23개 훅 JSON 파일 |
| 등록 수 | **54 base / 61 team-mode / 62 max** | ❓ 정확한 총계는 컴포넌트별 분산이라 집계값 미검증 | 23 |
| 컴포저 | `src/create-hooks.ts` → `createCoreHooks()` + `createContinuationHooks()` + `createSkillHooks()` | `src/extension/component-list.ts` 순 등록 | `plugin/scripts/build-components.mjs` |

근거: `packages/omo-opencode/src/hooks/AGENTS.md`의 TIER COMPOSITION 표 — "54 base registered hooks on default config (61 with team-mode; `monitor-status-injector` adds 1 more with `monitor.enabled` → 62 max)". 같은 문서가 미연결 WIP 2종(`task-reminder/`, `ralph-loop/`)을 명시하므로 "54 디렉터리 = 전부 배선"은 아니다.

**Senpi의 continuation 계층은 OpenCode와 구조적으로 다르다.** `ulw-execute-continuation`/`ulw-loop` 컴포넌트 문서상 결정 지점은 `agent_settled` — "the event the host defines as 'no automatic retry, compaction, or queued continuation will run'" — 이며, `components/ulw-execute-continuation/agent-end-eligibility.ts`가 retry 소유 턴·자동 압축 보류 턴·사용자 abort를 슬롯 소모 없이 차단한다. 연속 continuation은 8회 캡.

### 2.6 서브에이전트 / 위임 카테고리

| | omo-opencode | omo-senpi | omo-codex |
|---|---|---|---|
| 위임 툴 | `call_omo_agent`(explore/librarian), `task`(카테고리 위임) | `task`, `task_send`, `task_cancel`, `task_output` | Codex 네이티브 agent TOML 12개 |
| 카테고리 | 9종(`visual-engineering`, `architect`, `ultrabrain`, `deep-low`, `deep-high`, `artistry`, `quick`, `unspecified-low`, `unspecified-high`, `writing` — 스키마는 `src/config/schema.ts`) | `senpi-task` 리졸버 카테고리 (동일 `omo.jsonc` 사용) | ❓ Codex Light는 OpenCode 카테고리 모델을 쓰지 않음 |
| 내장 서브에이전트 | 11개 (`agents/AGENTS.md`) | 4개 curated read-only: `explore`, `librarian`, `plan-consultant`, `plan-reviewer` | Codex agent TOML 12개 |
| plan 게이트 | `prometheus-md-only` 훅 | `components/task/skill-invocation-tracker.ts` 3채널 판정 | ❓ |

Senpi 쪽 curated 서브에이전트는 `in-process execution`으로 고정되고 팀 멤버로는 거부된다("process-mode member spawns drop the persona prompt and tool policy") — `packages/omo-senpi/AGENTS.md`.

### 2.7 메모리

| | omo-opencode | omo-senpi | omo-codex |
|---|---|---|---|
| 엔진 | `memory-core` | `memory-core` 위의 어댑터 | ❓ |
| 툴 | `memory`, `memory_apply_patch` | `memory`, `memory_apply_patch` | ❓ |
| 슬래시 커맨드 | ❓ | 10개(`/memory` `/memfs` `/remember` `/init` `/doctor` `/recompile` `/memory-repository` `/sleeptime` `/reflect` `/search`) | ❓ |

Senpi 쪽은 `memory` 컴포넌트가 "Letta-Code-style persistent agent memory as a thin adapter over the harness-neutral `@oh-my-opencode/memory-core` engine (**zero Senpi imports**)"라고 단언한다. 선언된 divergence도 명시: Letta Cloud rows·mods-in-memory·arena/channels·`<memory_update>` 중간 갱신 없음, 텍스트 전용 로컬 검색. Reflection("dreaming")은 `agent_settled` 성공 시 detached `senpi -p` 자식을 `quick` 카테고리로 git worktree에서 띄우고 `--no-ff` 머지한다.

---

## 3. 왜 타깃이 다른가 (설계 이유)

### 3.1 어댑터 경계 원칙: core는 중립, 어댑터는 커플링

`packages/AGENTS.md`가 역할 축을 이렇게 잡는다: **Core 20개** + **MCP 4개** + **어댑터 3개(+2 지원)**.

`packages/omo-senpi/AGENTS.md`가 이 원칙의 enforcement 지점이다 — core 패키지는 "harness-neutral entrypoints"로 Senpi/Pi를 import하면 안 되고, **"The Senpi runtime boundary stays here"**(어댑터)에만 존재한다. 그래서 `senpi-task`는 `*-core`가 아니다: "`senpi-task` (Senpi-coupled task engine consumed only by `omo-senpi`; **not harness-neutral, so not a `*-core` package`)".

### 3.2 중복 제거 원칙: 호스트가 이미 하는 건 만들지 않는다

Senpi에서 기능이 빠지는 데는 대체로 이유가 적혀 있다:

| 빠진 것 | 이유(코드/문서 인용) |
|---|---|
| `websearch` MCP | "senpi ships `websearch`/`webfetch` builtins" — 호스트 내장이라 중복 |
| LSP MCP 래퍼 | Senpi는 `ToolDefinition`을 직접 등록하는 걸 권장. `permissionParser`/`kernelPrelude` 필드를 실제로 읽는 Senpi가 필요(#2178) → "**An older host ignores both fields, so the tool works without tiers or the global**" |
| `rules` 컴포넌트 | "Rules are **intentionally not** a Senpi component (**Senpi has builtin rules**, so this adapter must not add a `rules` component just to mirror Codex or OpenCode)" |
| OpenCode 에이전트 레지스트리 | Senpi에는 자체 에이전트 모델이 있고, `config-startup`이 "OpenCode edition's agent/category model choices Native does not use"를 **한 번만 보고하는 notice**로 처리(`opencode-routing-notice.ts`, `2026-09-opencode-routing-notice`) |

### 3.3 훅 모델의 근본 차이

가장 큰 설계 분기점은 **언제 개입할 수 있느냐**다.

- OpenCode = 이벤트 훅 20개 포인트. `preToolUse`/`postToolUse`에lisibility가 있어 가드 로직(`comment-checker`, `write-existing-file-guard`, `bash-file-read-guard`, `prometheus-md-only`)을 자연스럽게 얹는다. 그래서 5-tier 54~62개가 나온다.
- Senpi = `input`(사전 개입) + `tool_result`(사후 주입) 중심. 그래서 컴포넌트가 "**hidden custom message** with `display: false`"를 보내는 패턴이 반복된다(`ultrawork`, `skill-pointers`, `fallback-architect`). AGENTS.md는 그 이유를 나란히 붙인다:
  - idle 경로: 사용자 입력 텍스트를 절대 변형하지 않고 커스텀 메시지로 지시 주입
  - mid-stream(`streamingBehavior` 있는 큐잉 프롬프트): 커스텀 메시지는 턴을 통째로 소모하므로 **같은 메시지 안에 append**
  - `/skill:` 확장 경로: 텍스트를 아예 재작성하지 않음
- Codex = 7개 라이프사이클 이벤트. 그래서 훅 출력에 4096바이트 제한이 걸린다 — `omo-codex/AGENTS.md`는 ultrawork 포인터를 "<4096 bytes; Codex App truncates large hook output"로 설계하고, "full-directive fallback when the skills tree is absent"를 둔다. 포인터와 본문의 바이트 동일성은 테스트로 고정(`plugin/test/ultrawork-skill-pointer.test.mjs`).

### 3.4 실용적 결론: 언제 Native(Senpi)를 써야 하는가

`docs/guide/installation.md`는 "Most users want **Ultimate**"라면서도 "**Want one command without installing a host first?** Choose **OmO Native**"를 별도 선택지로 둔다. 다음 경우 Native가 명백히 우위다:

| 이유 | 근거 |
|---|---|
| **데스크톱 제어(`computer-use`)** 필요 | `senpi-desktop-*` 3+Rust 크레이트를 쓰는 유일한 타깃. `src/components/computer-use/` |
| **호스트 선행 설치 없이 단일 명령** | `omo-ai` npm 패키지 = `@code-yeongyu/senpi` 핀 + 확장 내장. `bin/omo.js` → `--extension <pkgRoot>/plugin` |
| **단일 바이너리 배포(노드/범 관리자 불필요)** | `docs/guide/binary-install.md`: "The binary is self-contained: it embeds the senpi engine, the omo plugin payload, and every runtime resource, and provisions them into `~/.omo/binary-runtime/<version>/` on first run. **No node, npm, or bun install is required.**" |
| **크로스세션 스레드 도구 17개** | `components/thread`: `thread_create/list/read/send/interrupt/handoff/rename/set_model/set_reasoning` + 릴레이 8개 |
| **세션별 격리된 task 호스트** | `packages/omo-senpi/AGENTS.md` "Engine hosts per session" — `p-<key>.sock`, `OMO_RPC_SHARD_ROOT`, `plugin/daemon-launch-spec.json`이 유일한 argv 소스 |
| **daemon 운영 명령** | `omo daemon run/adopt/status/stop/handoff/gc/rollback-prepare`, `omo thread …`, `omo host status --all` (`packages/omo-native/AGENTS.md`) |

반대로 **OpenCode Ultimate을 써야 하는 경우**: OpenCode를 이미 굴리고 있고 에이전트 11개 레지스트리 · 12개 `team_*` 툴 · hashline edit · 54~62 훅 · 완전한 delegation 카테고리 9종이 필요할 때. `installation.md`의 3-edition 표가 이 분기를 그대로 담고 있다.

### 3.5 명시된 비-패리티(주의할 진화 지점)

`packages/omo-senpi/AGENTS.md` 끝에 중요한 주의가 있다:

> "`packages/omo-opencode` is a separate build that **still uses its prior task/team names**; **cross-edition parity is a deliberate follow-up** outside this adapter."

즉 OpenCode와 Senpi의 task/team **툴 이름은 아직 동일하다고 가정하면 안 된다.** 기능 매트릭스를 크로스 에디션으로 옮겨 읽을 때 이 한 줄이 가장 중요한 유의점이다.

---

## 4. 공유 core 재사용과 분기 지점

### 4.1 의존성 그래프 (워크스페이스 `*-core` 기준)

**OpenCode 타깃 — 27 deps, 전부 core/어댑터 지원**
`agents-md-core`, `boulder-state`, `claude-code-compat-core`, `comment-checker-core`, `delegate-core`, `hashline-core`, `mcp-client-core`, `model-core`, `omo-config-core`, `openclaw-core`, `prompts-core`, `rules-engine`, `shared-skills`, `skills-loader-core`, `team-core`, `telemetry-core`, `tmux-core`, `utils` + **`omo-codex`**(역방향 의존 — Codex 어댑터가 Ultimate을 참조)

**Senpi 타깃 — 20 deps + 3 devDeps**
위 목록 중 공통: `boulder-state`, `comment-checker-core`, `delegate-core`, `model-core`, `omo-config-core`, `prompts-core`, `shared-skills`, `team-core`, `telemetry-core`, `utils` + **Senpi 전용**: `ast-grep-mcp`, `lsp-core`, `@code-yeongyu/lsp-daemon`, `senpi-desktop-engine`, `senpi-desktop-service`, `senpi-desktop-tool`, `senpi-task` + **`omo-opencode`(`/config-migration` 서브패스만)**

**Codex 타깃 — 2 deps**
`shared-skills`, `utils`. 그 외 `omo-config-core`는 직접 의존하지 않는다 — `omo-opencode` 쪽에서 `packages/omo-opencode/src/config-migration/`이 **의존성 정리된 공개 서브패스**(`config-migration-export.test.ts:86`의 "dependency-clean module" 테스트가 이를 검증) 형태로 `omo-config-core`를 노출하고, 코드는 그것을 통해 간접 참조한다. 정확한 진입 파일은 `packages/omo-config-core/src/index.ts`이며 `src/config-loader.ts` 같은 파일은 존재하지 않는다.

**Native 타깃 — 1 dep**
`@code-yeongyu/senpi`. core 의존성 0개 — 재사용은 전부 Senpi 엔진이 디스크에 스테이징한 `omo-senpi` 플러그인 payload를 통해 간접적으로 일어난다.

### 4.2 분기 지점 요약

```
                    utils · model-core · prompts-core · boulder-state
                    delegate-core · team-core · telemetry-core
                    comment-checker-core · shared-skills · omo-config-core
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
     ┌────────▼────────┐     ┌─────────▼──────────┐    ┌────────▼────────┐
     │  omo-opencode   │     │     omo-senpi      │    │    omo-codex    │
     │  (OpenCode)     │     │     (Senpi)        │    │    (Codex)      │
     ├─────────────────┤     ├────────────────────┤    ├─────────────────┤
     │ 5-tier 훅 54~62 │     │ 25 컴포넌트        │    │ 7 이벤트/23훅  │
     │ mcp/ 4종        │     │ builtin-mcps 2종   │    │ .mcp.json 4종  │
     │ agents/ 11      │     │ senpi-task 엔진    │    │ agent TOML 12   │
     │ tools/ (14 디렉)│     │ senpi-desktop-*    │    │ (src=설치러)    │
     │ hashline edit   │     │ ast-grep MCP       │    │ ast-grep 스킬   │
     │ lsp MCP(inline) │     │ lsp 직구 + daemon  │    │ lsp MCP         │
     │ team_* 12       │     │ team_* 6 + thread17│    │ teammode(경량)  │
     └─────────────────┘     └────────────────────┘    └─────────────────┘
              │                        │
              └──── omo-config-migration ┘  (Senpi가 OpenCode의 config-migration만 빌려 씀)
                                       │
                              ┌────────▼────────┐
                              │   omo-native    │
                              │  (bin: omo)     │
                              │  senpi 핀 실행  │
                              └─────────────────┘
```

**핵심 분기 3가지**

1. **MCP 등록 위치**: OpenCode = `src/mcp/index.ts` 한 함수(`createBuiltinMcps`)로 4종 중앙 등록. Senpi = 컴포넌트 2개로 분산(`components/ast-grep` + `components/builtin-mcps`). Codex = 정적 `plugin/.mcp.json` + 컴포넌트별 CLI.
2. **Task/Team 엔진**: OpenCode = `delegate-core` + `team-core` + `tmux-core`를 어댑터 안에서 직접 조립(`src/tools/delegate-task/`, `src/tools/background-task/`). Senpi = `senpi-task`(harness-coupled)를 `components/task`가 감싸고 그 위에 lead-only 팀 툴 6개를 등록.
3. **설치/업데이트 경로**: Codex = `npx lazycodex-ai install`로 `~/.codex/`를 직접 변형(백업·uninstall 로직 보유). Native = `curl … install.sh`(바이너리) 또는 `bun add -g omo-ai`(npm) 2경로.

---

## 5. `omo-native` 런처가 런타임에 하는 일

### 5.1 부트 시퀀스

```
omo (bin/omo.js)
 ├─ ensureBunBinShim()        # POSIX bun-global bin을 'sh → bun' 심으로 복구 (fail-open)
 ├─ maybeReexecUnderBun()     # bun机器 → execve로 bun에 인계 (PID/argv/env 보존)
 ├─ argv[2] === "setup" ? runSetup() : runLauncher()
 └─ runLauncher()  (bin/lib/launcher.js)
     ├─ launch-spec-mode.js   # daemon-launch-spec.json에서 group/world write 비트 제거
     ├─ engine-prepare.js     # Claude Code UA floor · css-tree 데이터 · RPC stream guard
     ├─ SENPI_BRAND 주입      # name / ~/.omo/agent / OMO_* 접두 / wire identity / 업데이트 채널
     └─ spawn: senpi --extension <pkgRoot>/plugin
```

- **런타임 정책**(`bin/lib/bun-runtime.js`): bun이 있는 머신은 설정 없이 bun으로 실행. bun-global 설치는 bun을 신뢰하고, 그 외 설치(npm·프로젝트 로컬·bunx)는 발견된 bun을 한 번 프로브해 `BUN_MIN_VERSION`(1.4.0) 이상이면 인계. `OMO_RUNTIME=node`는 항상 node 유지, `OMO_RUNTIME=bun`은 항상 re-exec. POSIX 인계는 `execve`(argv[0] 포함)로 잔류 래퍼를 없앤다. `spawnSync`는 절대 쓰지 않는다.
- **에이전트 상태 단일 위치**: `~/.omo/agent`. `bin/lib/agent-dir.js`의 `canonicalAgentDir()`이 유일한 답이며, 런처·`omo doctor`·`omo setup`·로컬 설치 런처(`packages/omo-senpi/src/install/local-launcher.ts`)가 전부 거기서 해석한다. 명시적 `OMO_CODING_AGENT_DIR`(구형 `SENPI_CODING_AGENT_DIR`/`PI_CODING_AGENT_DIR`)가 이김. `adoptLegacyFlatState()`가 통합 전 flat `~/.omo` 레이아웃 상태를 1회 이관.
- **자체 서브커맨드**: `omo doctor`(진단 + stale orphan 회수 — 패턴 kill 금지, 명시 pid만), `omo setup`(하네스 감지·SQLite read-only import·provider 매핑), `omo daemon …`, `omo thread …`, `omo host status --all`, `omo gateway …`(lazily named import, omo는 게이트웨이 코드를 싣지 않음).

### 5.2 12개 플랫폼 바이너리 패키지

`packages/AGENTS.md`가 정확히 설명한다: 각 패키지는 `bin/` + `package.json` **둘뿐**이며, `script/build-binaries.ts`가 `createPlatformLauncherSource()`로 **동일한** 생성 Node 런처 페이로드를 12곳에 쓴다.

> "these are **not distinct native binaries**, and the build **does not perform native compilation**." / "Current generated launcher payloads are **identical Node scripts**; the suffixes preserve x64 CPU and musl libc compatibility routing."

패키지 12개: `oh-my-opencode-{darwin,linux,windows}-{arm64,x64,x64-baseline}` + `linux-arm64-musl`, `linux-x64-musl`, `linux-x64-musl-baseline`. 런타임 선택은 `bin/` 심과 `postinstall.mjs`가 한다. publish는 `publish-platform.yml` 워크플로.

> ⚠️ 이름은 `oh-my-opencode-*`(= OpenCode Ultimate 측)이고, 이는 Native 런처 경로와 별개다. Native의 실제 배포 단위는 `omo-ai` npm 패키지와 GitHub Release의 단일 파일 `omo-<os>-<arch>`다.

### 5.3 설치/업데이트 3경로 (문서 대비)

| 문서 | 다루는 경로 | 요지 |
|---|---|---|
| `docs/guide/install.md` | **단일 바이너리(권장)** | `curl -fsSL https://get.omo.dev/install.sh \| bash` (Windows는 `install.ps1`), OS·CPU·libc·AVX2에 맞는 에셋 선택, `SHA256SUMS` 검증, `~/.local/bin/omo` 설치, PATH 미설정 시 셸 프로필에 마크된 블록 추가, `~/.omo/install.json` 영수증 기록, root 실행 거부 |
| `docs/guide/binary-install.md` | 위의 **수동·미러 상세** | GitHub Release별 단일 파일 `omo` 실행파일. 자체 완결형(엔진+플러그인+런타임 리소스 내장), 첫 실행 시 `~/.omo/binary-runtime/<version>/` 프로비저닝. `OMO_INSTALL_DIR`, `OMO_NO_MODIFY_PATH=1` 설정. 미러(get.omo.dev = Cloudflare R2 복사본) 미달 시 GitHub Releases에서 동일 파일 받아 동일 검증. Release URL: `releases/download/v<VERSION>/omo-<os>-<arch>` |
| `docs/guide/installation.md` | **3 에디션 판정** | Ultimate(OpenCode) / Light(Codex) / Native 비교표. `bunx oh-my-openagent install`, `npx lazycodex-ai install`, `curl … install.sh` 또는 `bun add -g omo-ai`. "**Do not install plain `omo` from npm: it is an unrelated package by a different author.**" |

빌드 파이프라인 연결고리: `bun run build:senpi-plugin` → lsp-daemon·ast-grep-mcp 빌드 후 `stage-lsp-daemon-runtime.mjs`·`stage-ast-grep-mcp-runtime.mjs`·`stage-x-search-skill.mjs`·`build-extension.mjs`(6개 번들)·`build-daemon-launch-spec.mjs`·`sync-skills.mjs`·`embed-directive.mjs`·`build-install.mjs` → `bun run build:omo-native`(`script/build-omo-native.ts`)가 그 payload를 `packages/omo-native/plugin/`로 스테이징.

---

## 6. 선택 가이드

### 6.1 사용자 상황 → 권장 타깃

| 사용자의 상황 | 권장 타깃 | 이유 | 근거 |
|---|---|---|---|
| OpenCode를 이미 쓰고 있고 에이전트 11종·`team_*` 12개가 필요 | **omo-opencode** (Ultimate) | 유일하게 OpenCode 에이전트 레지스트리와 `team_*` 패밀리를 제공 | `packages/omo-opencode/src/tools/`, `packages/omo-codex/README.md` |
| Codex CLI에 이미 깊이 투자했다 | **omo-codex** (Light) | `npx lazycodex-ai install` 1줄, Codex 네이티브 플러그인 시스템에 정렬 | `docs/guide/installation.md` |
| OpenCode와 Codex 양쪽 다 필요하다 | **양쪽 설치** | `bunx oh-my-openagent install --platform=both` | `docs/guide/installation.md` |
| 호스트를 먼저 설치하고 싶지 않다, 단일 명령이 필요하다 | **omo-native** | `omo` 명령 하나에 Senpi 엔진 + OMO 확장이 내장 | `packages/omo-native/package.json`, `bin/omo.js` |
| 노드/범/bun 설치가 불가능한 환경(에어갭, 통제된 서버) | **omo-native** (바이너리) | 자체 완결형 단일 실행파일, 런타임 의존성 0 | `docs/guide/binary-install.md` |
| 데스크톱 GUI를 조작해야 한다 (computer-use) | **omo-native** (= omo-senpi) | `senpi-desktop-*` 3패키지 + Rust 엔진은 Senpi 타깃에만 존재 | `packages/omo-senpi/src/components/computer-use/` |
| 여러 세션/프로젝트를 오가는 스레드 워크플로 | **omo-native** | `thread_*` 17개 크로스세션 툴 + `omo thread` CLI | `packages/omo-senpi/src/components/thread/`, `packages/omo-native/bin/lib/thread.js` |
| 백그라운드 자식 프로세스를 격리 실행해야 한다 | **omo-native** | 세션별 `p-<key>.sock` engine host로 shard 라우팅 | `packages/omo-senpi/src/components/task/shard-routing.ts` |
| AST 검색/리라이트를 프로그래밍 가능하게 쓰고 싶다 | **omo-native** | ast-grep가 MCP(`search`/`rewrite`/`scan`)로 제공됨 | `packages/omo-senpi/src/components/ast-grep/` |
| 훅 기반 가드(쓰기 전 파일 존재 검사, 코멘트 체커)를 넓게 쓰고 싶다 | **omo-opencode** | 5-tier 훅 54~62개, Tool Guard 티어 17~18개 | `packages/omo-opencode/src/hooks/AGENTS.md` |
| MCP 스크립팅의 최대 유연성이 필요하다 | **omo-opencode** | 3티어 MCP(내장 4 + `.mcp.json` + 스킬 내장) 전부 활성 | `packages/omo-opencode/src/mcp/index.ts` |
| 버전 고정·재현 가능한 배포가 필요하다 | **omo-native** | senpi를 **exact-pin**(`@code-yeongyu/senpi@2026.10.3`)으로 실행 | `packages/omo-native/AGENTS.md` |
| 최소 침습(설치가 기존 홈 디렉터리를 거의 바꾸지 않아야 함)이 중요하다 | **omo-codex** 또는 **omo-opencode** | 둘 다 플러그인. Native는 `~/.omo/agent`를 단일 정본으로 강제한다 | `packages/omo-native/AGENTS.md`(`adoptLegacyFlatState`) |
| Rust/네이티브 데스크톱 엔진을 유지보수해야 한다 | **omo-senpi** | `senpi-desktop-*` 5개 패키지가 Senpi에만 존재 | 루트 `package.json` workspaces |

### 6.2 교차 참조 시의 유의점

| 위험 | 대응 |
|---|---|
| task/team **툴 이름이 에디션마다 다를 수 있음** | `packages/omo-senpi/AGENTS.md`: "still uses its prior task/team names; cross-edition parity is a **deliberate follow-up**". 크로스 에디션 스크립트 이관 전 반드시 각 타깃의 descriptor 확인 |
| Claude Code 훅 JSON을 그대로 옮겨도 안 됨 | Codex는 7개 이벤트, OpenCode는 20 포인트, Senpi는 `input`/`tool_result`/生命周期. `omo-codex/README.md`의 Git Bash 발견 로직처럼 **훅 출력 바이트 상한**(4096)을 먼저 확인 |
| MCP 이름이 겹치면 사용자 `mcp.json`이 이긴다 | Senpi: "A trusted `mcp.json` entry of the same name **wins over** the extension declaration". Codex: installer가 기존 사용자 `[mcp_servers.context7]` 블록을 **건드리지 않음** |
| Claude Code API 키 없는 프로젝트 인용 텍스트가 키워드를 발동시킴 | Senpi는 `skill-pointers/strip-quoted-regions.ts`로 인용 영역을 blanking. 이 전처리가 없는 타깃으로 포팅하면 재현되는 오탐 |
| Senpi 툴 레벨 권한(`permissionParser`/`kernelPrelude`) | 구형 호스트는 두 필드를 무시하므로 툴은 동작하지만 **티어와 전역 변수는 없음**(`computer-use` AGENTS.md) |
| OpenCode 훅 개수를 세려면 미연결 WIP 제외 | `task-reminder/`, `ralph-loop/`는 barrel에 있으나 composer에 미연결(`hooks/AGENTS.md`) |

### 6.3 확인 필요 항목 (미검증)

이 문서를 쓰면서 코드에서 직접 확인하지 못한 행:

- `omo-codex`의 `lsp` MCP가 제공하는 **정확한 툴 수**
- `omo-codex`에 메모리 엔진(`memory-core`)이 있는지 — Codex 컴포넌트 10개 목록에 없음(없다고 추정하나 미검증)
- `omo-opencode`의 `memory` 툴에 대응하는 슬래시 커맨드 수
- Senpi 훅의 **총 등록 수**(컴포넌트별 분산이라 집계 없음 — `components/` 25개가 곧 상한)
- `omo-codex`의 delegation 카테고리 — `installation.md`가 "no OpenCode agent registry"라고 하는 것 외에 Codex 네이티브 위임 모델
- OpenCode 타깃의 `x_search` 존재 여부 — `src/`에 `x-search` 디렉터리를 확인하지 못함
