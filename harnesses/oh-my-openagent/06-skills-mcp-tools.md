# 06. 스킬 발견 · MCP 계층 · 툴 표면

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13`
> 상위 문서: [oh-my-openagent.md](../oh-my-openagent.md) — 교차 비교: [topics/skills.md](../../topics/skills.md) · [topics/mcp.md](../../topics/mcp.md) · [topics/lsp-lint.md](../../topics/lsp-lint.md)

OmO는 8개 하니스 중 **유일하게 MCP를 3-tier로 계층화**하고, 스킬 발견도 **7-source + 명시적 우선순위 숫자**로 정렬한 하니스다. 동시에 "에이전트 툴 12개는 항상, 나머지 26개는 게이트로" 나눠 등록하는 유일한 하네이기도 하다.

## 한눈에 보기

| 축 | 핵심 사실 |
|---|---|
| 스킬 발견 | 7 discover 소스 → 6개 scope 값 → `SCOPE_PRIORITY` 숫자 병합 |
| 번들 스킬 | TS 내장 13 스킬 파일(=`*Skill` export 13종) + shared `SKILL.md` **18개** |
| 프론트매터 | `name`/`description`/`allowed-tools`/`metadata`/`model`/`agent`/`subtask`/`license`/`compatibility`/`mcp` |
| MCP | Tier 1 내장 4개(3 remote HTTP + 1 local stdio) / Tier 2 `.mcp.json` / Tier 3 SKILL.md frontmatter |
| 보안 | Tier-3 클라이언트 키 `${sessionID}:${skillName}:${serverName}`, `mcp_env_allowlist`는 **user 레이어만** 강제 |
| 툴 | 항상 12 + 조건부 26 (=38), LSP는 툴이 아니라 **Tier-1 MCP 서버** |
| 편집 | `hashline_edit` opt-in. `LINE#HASH\|content` 포맷, xxHash32 % 256 |

---

## 1. 스킬 발견: 7소스 · SCOPE_PRIORITY

로더 본체는 `packages/skills-loader-core/src/features/opencode-skill-loader/`다. 내부 `AGENTS.md`는 7개 `discover*` 함수를 **5개 scope 패밀리**로 묶어 서술한다.

```typescript
// packages/skills-loader-core/src/features/opencode-skill-loader/merger/scope-priority.ts
export const SCOPE_PRIORITY: Record<SkillScope, number> = {
  builtin: 1,
  shared: 1,          // ← shared도 builtin과 동점(1)
  config: 2,
  user: 3,
  opencode: 4,
  project: 5,
  "opencode-project": 6,
}
```

| # | discover 소스 | scope | 우선순위 | 경로 | trust 게이팅 |
|---|---|---|---|---|---|
| 1 | OpenCode project | `opencode-project` | **6** | `<cwd>/.opencode/skills/` | 없음 (프로젝트 파일을 신뢰) |
| 2 | Claude project | `project` | 5 | `<cwd>/.claude/skills/` | 없음 |
| 3 | Agents project | `project` | 5 | `<cwd>/.agents/skills/` | 없음 |
| 4 | OpenCode config/global | `opencode` | 4 | `$XDG_CONFIG_HOME/opencode/skills/` | 없음 |
| 5 | Claude user | `user` | 3 | `~/.claude/skills/` | 없음 |
| 6 | Agents global | `user` | 3 | `~/.agents/skills/` | 없음 |
| 7 | Shared skills root | `shared` | 1 | `packages/shared-skills/skills/` (번들) | 패키징으로 고정 |
| — | config 선언 | `config` | 2 | `SkillsConfig.sources[]` / `omo.jsonc`의 `skills:` | 사용자 설정 |
| — | TS 내장 | `builtin` | 1 | `builtin-skills/skills/*.ts` | 코드에 컴파일 |

**병합 규칙** (`merger.ts:86`): 같은 `name`이면 `SCOPE_PRIORITY[skill.scope] > SCOPE_PRIORITY[existing.scope]` 일 때만 교체. 즉 **높은 숫자가 항상 이긴다**. `disabled_skills`에 있으면 로드되지 않는다(`merger/scope-priority.ts` 주석: "disabled names never load").

> **부모 문서 정정**: 부모 문서의 `SCOPE_PRIORITY: opencode-project(6)>project(5)>opencode(4)>user(3)>config(2)>builtin(1)`은 **맞지만 `shared`가 빠져 있다**. 실제 상수는 `shared: 1`을 가지며 `builtin`과 동점이다. 즉 번들 `shared-skills`와 TS 내장 스킬은 같은 이름이면 **서로 구별되지 않고 나중에 온 쪽이 이긴다**.

> **LoC 정정**: 부모 문서의 `packages/skills-loader-core/ (7,560 LoC)`는 실제 측정과 다르다. `.ts` 기준 **121 파일 / 10,774줄**, 그중 테스트를 뺀 production 코드는 **5,368줄**이다. `AGENTS.md`가 단독으로 지목하는 `opencode-skill-loader/`만 보면 "55 files (~6.2k LOC)".

### 비동기 로더와 지연 로딩
`async-loader.ts`가 동기 로더와 병존한다. 두 가지 특징이 있다:
- **bounded concurrency**로 디렉터리 트리 순회
- `LazyContentLoader` — `content`를 미로딩 상태로 두고 `load()`로 지연 로딩. 프롬프트에는 name/description만 먼저 노출된다

---

## 2. 번들 스킬: TS 내장 + shared 18개

### 2.1 TS 내장 스킬 (`packages/skills-loader-core/src/features/builtin-skills/skills/`)
`index.ts`의 export 목록으로 실제 개수를 세면 다음과 같다.

| export | 비고 |
|---|---|
| `createPlaywrightSkill`, `playwrightSkill`, `PlaywrightSkillOptions` | Playwright 스킬 1개 + 옵션 타입 |
| `playwrightCliSkill` | 별도 스킬 |
| `frontendSkill` | |
| `gitMasterSkill` | + `git-master-sections/` 8분할 모듈 |
| `devBrowserSkill` | |
| `reviewWorkSkill` | |
| `removeAiSlopsSkill` | |
| `initDeepSkill` | |
| `debuggingSkill` | |
| `securityResearchSkill` | |
| `securityReviewSkill` | |
| `visualQaSkill` | |
| `* from "./team-mode"` (`teamModeSkill`) | |

> **부모/토픽 문서 정정**: `skills/index.ts` 기준 스킬 스킬 파일은 **13개**(`team-mode.ts`, `security-research.ts` 등)다. 부모 문서·`topics/skills.md`의 "14개"는 `index.ts`의 **export 라인 수**(한 스킬이 3줄 export)를 세거나, `git-master-skill-metadata.ts` 같은 비스킬 파일까지 포함한 결과다. 실제 `skills/` 디렉터리에서 스킬 정의를 가진 파일은 `debugging / dev-browser / frontend / git-master / init-deep / playwright / playwright-cli / playwright-mcp-skill / remove-ai-slops / review-work / security-research / security-review / team-mode / visual-qa` = **14개 파일**이며, 그중 `playwright-mcp-skill.ts`는 `index.ts`에서 re-export되지 않으므로 **사용자에게 노출되는 스킬은 13개**다.

### 2.2 shared-skills (`packages/shared-skills/skills/`) — 정확히 18개

`packages/shared-skills/`의 `.md`는 총 37,440줄(상위 문서의 39,139줄과 소폭 차이 — 스냅샷 시점 차이).

| 스킬 | 한 줄 목적 |
|---|---|
| `ast-grep` | 25개 언어의 AST 모양 기반 검색·리라이트(`sg`/`ast-grep` CLI + Python 래퍼 `scripts/ast_grep_helper.py`) |
| `browser` | js eval 커널의 omowright 라이브러리로 실제 브라우저 구동(로그인 세션 포함) |
| `coding-agent-sessions` | Codex/Claude/OpenCode/OMO 등 코딩 에이전트 세션 탐색·복원 |
| `data-scientist` | DuckDB/Polls 상주 커널 + 일회용 툴로 CSV/Parquet/JSON 처리 |
| `debugging` | 가설 주도 디버깅 루프(언어·바이너리 무관) |
| `frontend` | 웹 UI/UX 구축·스타일링·폴리시 |
| `git-master` | 원자적 커밋, rebase, squash, blame, bisect, 이력 검색 |
| `init-deep` | 계층형 AGENTS.md 지식 베이스 초기화 |
| `lsp-setup` | 언어 서버 설정(diagnostics/goto-definition/references/rename) |
| `programming` | Python/Rust/TypeScript/Go 엄격 실무(타입드 에러, exhaustive match) |
| `refactor` | 리팩터링·정리·구조 분해 가이드 |
| `remove-ai-slops` | 회귀 테스트로 먼저 고정하고 AI 생성 코드 냄새 제거 |
| `review-work` | 구현 후 게이트 리뷰(실제 표면 QA → 게이트 리뷰어 1명) |
| `ultimate-browsing` | JS 렌더링 소스·클릭·양식·WAF 우회 등 어려운 웹 접근 |
| `ulw-execute` | ulw-plan 실행기(Boulder 상태, 증거 원장, worktree 규율) |
| `ulw-plan` | 코딩 전 결정 완결(work plan) 1개를 먼저 쓰는 탐색 우선 기획 컨설턴트 |
| `ulw-research` | 최대 포화 리서치(협업 팀, 인용된 산출물) |
| `visual-qa` | 웹·터미널·페이지 표면 시각 QA + 스크린샷 증거 + 판정 |

`shared`와 `builtin`은 SCOPE_PRIORITY 동점(1)이라 `debugging`, `frontend`, `git-master`, `init-deep`, `remove-ai-slops`, `review-work`, `visual-qa`는 **이름이 겹친다** — 두 벌이 실제로 공존한다.

### 2.3 번들 스킬이 상속하는 MCP/브라우저 설정
`BuiltinSkill` 인터페이스(`features/builtin-skills/types.ts`)는 `name`, `description`, `content`에 더해 **optional MCP config**를 갖는다. 그래서 `playwright` 스킬은 스킬 본문과 함께 MCP 서버 선언을 품고, 이 선언이 Tier-3 파이프라인으로 흘러간다. `BrowserAutomationProvider = "playwright" | "dev-browser" | "playwright-cli"`(`skills-loader-core/src/types.ts`)가 이 축을 타입 수준에서 고정한다.

---

## 3. 스킬 프론트매터 계약

실제 파서가 읽는 키는 `features/opencode-skill-loader/types.ts`의 `SkillMetadata`와 `skills-loader-core/src/types.ts`의 `SkillDefinition` 두 타입이 결정한다.

### 3.1 정식 키 (Honored)

| 키 | 타입 | 위치 | 역할 |
|---|---|---|---|
| `name` | string | `SkillMetadata` | 스킬 ID. 디렉터리명·병합 키로 사용 |
| `description` | string | `SkillMetadata` | 프롬프트에 주입되는 요약. 스킬 선택의 유일한 신호 |
| `allowed-tools` | `string \| string[]` | 둘 다 | 허용 툴. 좁히기 전용 |
| `metadata` | `Record<string,string>` | 둘 다 (`unknown`/`string`) | 자유 공간 |
| `model` | string | `SkillDefinition`만 | 스킬 실행 모델 고정 |
| `agent` | string | 둘 다 | 스킬 전담 에이전트 |
| `subtask` | boolean | 둘 다 | 서브태스크로 실행 여부 |
| `license` | string | 둘 다 | 메타데이터로 승격 |
| `compatibility` | string | 둘 다 | 호환 힌트 |
| `argument-hint` | string | `SkillMetadata` | `/-command` 인자 힌트 |
| `disable` | boolean | `SkillDefinition`만 | 개별 비활성화 |
| `from` | string | `SkillDefinition`만 | 출처 표기 |
| `template` | string | `SkillDefinition`만 | 본문 템플릿 |
| `mcp` | `SkillMcpConfig` | `SkillMetadata`만 | **Tier-3 MCP 선언** |

`mcp`의 타입은 `Record<string, ClaudeCodeMcpServer>` — Tier-2의 `.mcp.json`과 **같은 서버 스키마**를 재사용한다. 그래서 Tier-3 하나의 선언으로 stdio·HTTP 양쪽이 표현된다.

`disable`는 `SkillDefinition`(설정 스키마)에만 있고 `SkillMetadata`(SKILL.md 파싱 결과)에 없다. 즉 **프론트매터가 아니라 `omo.jsonc`의 스킬 항목에서만** 쓸 수 있다.

### 3.2 OmO 고유 확장

| 확장 | 위치 | 설명 |
|---|---|---|
| `mcp:` 블록 | SKILL.md frontmatter | Tier-3. `skill-mcp-config.ts`가 YAML에서 추출 |
| `disabled_skills` / `disabled_mcps` / `disabled_tools` / `disabled_agents` | `omo.jsonc` | 전역 강제 비활성화 목록. 병합 시 `mergeUniqueStrings` |
| `enable` / `disable` | `SkillsConfig` | 스킬 단위 온/오프 |
| git-master 템플릿 주입 | `git-master-template-injection.ts` | 번들 git-master 스킬의 프롬프트 변수를 런타임에 치환 |

`skill-template-resolver.ts`가 `{{directory}}`, `{{agent}}` 같은 변수를 로드 시점에 치환하고, "disabled watermark" 옵션도 처리한다.

### 3.3 다른 하니스의 키와 명시적 불일치
`topics/skills.md`가 나열하는 oh-my-pi의 프론트매터 키 `globs` / `alwaysApply` / `hide` / `disableModelInvocation` / `enabled`는 **Ohmy-Pi의 키**다. OmO의 `SkillMetadata`에는 **`globs`, `alwaysApply`, `hide`, `disableModelInvocation`가 없다.** OmO에서 대응하는 것은 `disabled_skills`(전역 목록)와 `disable`(설정 항목)이며, **`enabled`도 없다.** 이 함정은 Claude Code의 Agent Skills 스펙(`disable-model-invocation`)과도 혼동하기 쉬우므로 주의할 것.

---

## 4. MCP 3-tier

`packages/omo-opencode/src/mcp/AGENTS.md`가 명시하는 구조다.

| Tier | 소스 | 관리자 | 스코프 |
|---|---|---|---|
| 1. Built-in | `packages/omo-opencode/src/mcp/` | `createBuiltinMcps(disabledMcps, config, options)` | 전역. 3 remote HTTP + 1 local stdio |
| 2. Claude Code 호환 | `.mcp.json` | `claude-code-mcp-loader` → `plugin-handlers/mcp-config-handler.ts` (설정 로드 5단계) | project + user |
| 3. 스킬 내장 | SKILL.md YAML frontmatter | `SkillMcpManager` (`features/skill-mcp-manager/`) | **세션별**, stdio + HTTP |

### 4.1 Tier 1 — 내장 4개 (`createBuiltinMcps()`)

| 이름 | transport | endpoint / command | env | 툴 |
|---|---|---|---|---|
| `websearch` | remote HTTP | `mcp.exa.ai`(기본) 또는 `mcp.tavily.com` | `EXA_API_KEY`(선택), `TAVILY_API_KEY`(tavily일 때) | 웹 검색 |
| `context7` | remote HTTP | `mcp.context7.com/mcp` | `CONTEXT7_API_KEY`(선택) | 라이브러리 문서 |
| `grep_app` | remote HTTP | `mcp.grep.app` | 없음 | GitHub 코드 검색 |
| `lsp` | **local stdio** | `node packages/lsp-tools-mcp/dist/cli.js mcp` 또는 `bun packages/lsp-tools-mcp/src/cli.ts mcp` | `LSP_TOOLS_MCP_PROJECT_CONFIG`, `LSP_TOOLS_MCP_USER_CONFIG`, `LSP_TOOLS_MCP_INSTALL_DECISIONS` | status/diagnostics/goto_definition/references/symbols/prepare_rename/rename (+ `installDecisionTool: true` 컨텍스트) |

- 타입은 `McpNameSchema = "websearch" | "context7" | "grep_app" | "lsp"`로 **폐쇄 집합**(`mcp/types.ts`).
- **`disabled_mcps`에 없으면 CLI 산출물이 아직 빌드되지 않았어도 등록된다.** 소스 체크아웃은 Bun 소스 CLI로 폴백, 패키징 빌드는 Node `dist` CLI를 우선(`mcp/lsp.ts`가 런타임 레이아웃을 동적으로 해석).
- OpenCode가 넘기는 프로젝트 설정 경로는 **순서 고정 3개**: `<cwd>/.opencode/lsp.json`, `<cwd>/.omo/lsp.json`, `<cwd>/.omo/lsp-client.json`. 사용자 레벨은 `$XDG_CONFIG_HOME/opencode/lsp.json` + `lsp-install-decisions.json`.
- **격리**: 전역. 세션별 격리 없음. 인증은 remote 3개에 대한 API 키 헤더뿐이고 OAuth 없다.

### 4.2 Tier 2 — `.mcp.json` (Claude Code 호환)
파이프라인:
```
loadMcpConfigs(ctx)
  → scope-filter.ts   : project + user 스코프에서 .mcp.json 탐색
  → loader.ts         : JSON 파싱
  → env-expander.ts   : ${VAR} → process.env[VAR] 재귀 치환
  → transformer.ts    : Claude Code 형식 → OpenCode McpLocal / McpRemote
```
- **지원 transport**: `stdio`(command/args/env), `http`(url/headers). 레거시 `"sse"`는 http로 매핑.
- **인증**: `${VAR}` 문자열 치환뿐이다. **`$()` 셸 실행이 아니다** — `AGENTS.md`가 명시적으로 경고한다. 값은 `mcp_env_allowlist`로 게이트된다.
- **격리**: 스코프 기반(project > user). 세션 격리 없음.

### 4.3 Tier 3 — 스킬 내장 (`SkillMcpManager`)
하네스 중ral MCP 클라이언트 수명주기 전담 모듈이다. 중립 핵심(`mcp-client-core/`, `mcp-stdio-core/`)을 추출해 두고 OpenCode 배선만 이 디렉터리에 유지한다.

**클라이언트 키 형식**
```
${sessionID}:${skillName}:${serverName}
```
AGENTS.md가 명시하는 존재 이유 3가지:
1. **세션 격리** — 한 세션이 만든 클라이언트를 다른 세션이 절대 못 씀
2. **동일 스킬의 동시 다중 세션 사용** — 같은 스킬이 N개 세션에서 병렬로 떠도 각자 자기 클라이언트를 가짐
3. **스킬당 다중 서버** — 한 스킬이 여러 서버를 선언해도 키가 충돌하지 않음

세 요소가 모두 있어야 한다. `sessionID`만으로는 "같은 스킬의 다른 서버"를 못 갈라내고, `skillName`만으로는 세션 격리가 깨진다.

| 항목 | 구현 |
|---|---|
| transport 판별 | `connection-type.ts`: 명시적 `type` → URL 존재 → command 존재. 레거시 `sse` → `http` |
| stdio 백엔드 | `StdioClientTransport` (`stdio-client.ts`) |
| http 백엔드 | `StreamableHTTPClientTransport` (`http-client.ts`) |
| OAuth | `oauth-handler.ts` — 401 시 토큰 갱신, **403 시 step-up(scope 승격)**. auth provider는 **서버 URL 키**이므로 토큰이 서버 간 이동하지 않음 |
| 재시도 | `getOrCreateClientWithRetry()` 3회 force-reconnect, `withOperationRetry()`는 OAuth 인식 래퍼 |
| race 방지 | `pendingConnections`(동일 키 연결 중복), `inFlightConnections`(세션 카운터로 조기 cleanup 방지), `shutdownGeneration`(stale 연결 탐지) |
| idle cleanup | 60초 인터벌 / 5분 TTL |
| 종료 | `session.deleted` 훅(`src/plugin/event.ts`) → `disconnectSession(id)`; SIGINT/SIGTERM → `disconnectAll()` |

**보안 2중 방어**:
- `env-cleaner.ts` — stdio spawn 전에 npm/pnpm/yarn 설정 변수와 **25개 이상의 시크릿 패턴**(`_KEY`, `_SECRET`, `_TOKEN`) 제거
- `error-redaction.ts` — 로그 전 토큰/시크릿 마스킹

### 4.4 Tier 간 참조
`mcp-client-core/`는 OAuth primitive와 클라이언트 수명주기를, `mcp-stdio-core/`는 stdio 전송을, `claude-code-compat-core/`는 Tier-2 파싱·치환·변환을 담는다. 즉 **Tier 2와 Tier 3이 같은 서버 스키마 타입(`ClaudeCodeMcpServer`)을 공유**한다.

---

## 5. 보안: `mcp_env_allowlist`는 user 레이어만

부모 문서가 주장한 "user 레이어만"은 **검증 확인**된다. 스키마는 `packages/omo-opencode/src/config/schema/oh-my-opencode-config.ts:57`에 `z.array(z.string()).optional()`로 선언되고(그리고 `omo-config-core`가 아니라 **omo-opencode의 config 스키마**에 있다), 병합 규칙은 `plugin-config/config-merger.ts:22`다:

```typescript
mcp_env_allowlist: override.mcp_env_allowlist ?? base.mcp_env_allowlist,
```

겉보기에는 "override 우선"이지만 실제로는 **병합 후 보호 단계**가 병합 결과를 덮어쓴다:

```typescript
// packages/omo-opencode/src/config/validate.ts:76
function protectUserFields(config, userConfig): OhMyOpenCodeConfig {
  const userMcpEnvAllowlist = userConfig?.mcp_env_allowlist ?? []
  return {
    ...config,
    mcp_env_allowlist: userMcpEnvAllowlist,   // ← 병합 결과 폐기, user 값으로 강제
    ...
  }
}
```

**의미**:
1. 체크인된 **프로젝트 `.omo/omo.jsonc`는 `mcp_env_allowlist`를 설정할 수 없다.** 값이 있어도 `protectUserFields`가 user 값(없으면 빈 배열)으로 교체한다.
2. 따라서 **allowlist는 공격 표면이 아니다.** 체크인된 저장소가 신규 시크릿 환경변수를 `.mcp.json`의 `${VAR}`로 끌어오르는 건 막히고, 승인된 변수는 **사용자가 `~/.omo/omo.jsonc`에 직접 써넣어야만** 확장된다.
3. 빈 배열(기본) = **기본 거부**. 명시적 allowlist 없이는 어떤 `${VAR}`도 확장되지 않는다.
4. 같은 보호가 `browser_automation_engine.playwright_mcp_args`에도 적용된다 — 프로젝트가 브라우저 엔진 인자를 바꿀 수 없다.

Tier-3는 `env-cleaner.ts`가 다른 축의 방어를 한다: allowlist가 아니라 **비밀 자체를 spawn 전에 걷어낸다.** 두 방어는 직교한다 — Tier-2는 "어떤 변수를 넣을 것인가"를 사용자만 결정하고, Tier-3는 "어떤 변수를 자식 프로세스에 넘길 것인가"를 스킬이 절대 결정 못 한다.

---

## 6. 툴 표면: 항상 12 vs 조건부 +26

부모 문서의 12+26은 **정확하다**. 등록 코드는 `packages/omo-opencode/src/plugin/tool-registry.ts`(+ `-core-tools.ts`, `-gated-tools.ts`, `-team-tools.ts`, `-trimming.ts`)에 있고, 스펙 문서는 `src/tools/AGENTS.md`(제목이 "12–38 Native Tools Across 14 Tool Directories")다.

### 6.1 항상 켜짐 (12개)

| 그룹 | 툴 | 소스 디렉터리 |
|---|---|---|
| 검색 (2) | `grep` (60초 타임아웃, 10MB 제한), `glob` (60초, 100파일 제한) | `tools/grep/`, `tools/glob/` |
| 세션 (4) | `session_list`, `session_read`, `session_search`, `session_info` | `tools/session-manager/` |
| 백그라운드 (2) | `background_output`, `background_cancel` | `tools/background-task/` (엔진은 `features/background-agent`) |
| 위임 (2) | `task`(카테고리/스킬 전체 지원), `call_omo_agent`(explore·librarian만) | `tools/delegate-task/`, `tools/call-omo-agent/` |
| 스킬·MCP (2) | `skill`, `skill_mcp` | `tools/skill/`, `tools/skill-mcp/` |

**LSP는 이 목록에 없다.** `lsp_*` 툴은 Tier-1 내장 stdio MCP(`lsp`)가 제공하는 것이므로 MCP 네이밍을 달고 노출된다. AST 검색도 툴이 아니라 `ast-grep` 스킬(`sg`)이다.

### 6.2 조건부 (+26개)

| 툴 | 활성화 조건 | 소스 |
|---|---|---|
| `look_at` (1) | `multimodal-looker`가 `disabled_agents`에 없음 | `tools/look-at/` |
| `interactive_bash` (1) | `isInteractiveBashEnabled(config)` (tmux 설정) | `tools/interactive-bash/` |
| `task_create`, `task_get`, `task_list`, `task_update` (4) | `experimental.task_system` | `tools/task/` |
| `monitor_start`, `monitor_stop`, `monitor_list`, `monitor_output` (4) | `monitor.enabled` | `tools/monitor/` (엔진 `features/monitor/`) |
| `edit` (hashline) (1) | `hashline_edit: true` | `tools/hashline-edit/` |
| `create_goal`, `update_goal`, `get_goal` (3) | `goal.enabled` | `hooks/goal/tools.ts` |
| `team_*` (12) | `team_mode.enabled: true` | `features/team-mode/tools/` |

12 `team_*` = `team_create`, `team_delete`, `team_shutdown_request`, `team_approve_shutdown`, `team_reject_shutdown`, `team_send_message`, `team_task_create`, `team_task_list`, `team_task_update`, `team_task_get`, `team_status`, `team_list`.

따라서 **12 + 26 = 38개가 이론적 상한**이며 `src/tools/AGENTS.md`의 제목("12–38")이 이 합을 가리킨다. 실제 개수는 게이트 설정에 좌우된다.

**게이트 패턴**: 항상 켜짐 툴은 `allTools`에 직접 spread, 조건부는 `Record<string, ToolDefinition>`을 만들어 게이트를 통과시켜 spread한다. 끄려면 `filterDisabledTools` allow-list에 이름이 있어야 하고 `disabled_tools`와 대조된다.

### 6.3 위임 카테고리 9개
`task`가 카테고리로 모델을 고른다. 기본 모델은 `src/tools/delegate-task/*-categories.ts`에 있고, 권위 있는 fallback 체인은 `packages/model-core/src/category-model-requirements.ts`의 `CATEGORY_MODEL_REQUIREMENTS`. 사용자 정의 `categories: { ... }`는 이 집합을 덮어쓰고 확장한다.

---

## 7. hashline edit

`packages/hashline-core/` (18 source 파일, 테스트 포함 **2,495줄**)가 엔진이고, `packages/omo-opencode/src/tools/hashline-edit/`가 얇은 re-export shim 약 15개다. **Codex edition(Light)은 hashline을 쓰지 않는다.**

### 7.1 라인 포맷과 해시
```typescript
// packages/hashline-core/src/hash-computation.ts:11, 23-26
const index = hash % 256
...
export function formatHashLine(lineNumber: number, content: string): string {
  const hash = computeLineHash(lineNumber, content)
  return `${lineNumber}#${hash}|${content}`
}
```

- **포맷**: `LINE#HASH|content` — 줄 번호, `#`, 1바이트 해시, `|`, 원본 내용.
- **해시**: `xxhash32.ts`가 호출 시점에 네이티브 `Bun.hash.xxHash32()`를 우선 쓰고 순수 JS 구현으로 폴백한다(npm 의존성 없음, "Option 2" 방식). 결과는 **256으로 나눈 나머지**라 한 줄당 2자 16진수.
- **시드 규칙**: 콘텐츠 줄은 시드 `0`, **공백 전용 줄은 시드 `lineNumber`** — 같은 빈 줄도 위치마다 해시가 다르다.
- **정규화**: `computeLineHash`는 `trimEnd` + `\r` 제거. `computeLegacyLineHash`는 공백 전부 제거. **검증은 두 해시를 모두 받아들여** 과거 저장 해시가 계속 유효하다.
- **legacy 검증 오류 메시지**: `normalizeHashlineEdits`가 알 수 없는 연산을 거부하며 `"Legacy format was removed; use op/pos/end/lines."` — 판별 유니온은 **`replace` | `append` | `prepend` 3종뿐**이다.

### 7.2 3-연산 모델
| 연산 | 의미 | 구현 |
|---|---|---|
| `replace` | 특정 라인 구간 교체 | `applyReplaceLines` / `applySetLine` |
| `append` | 파일 끝에 추가 | `applyAppend` |
| `prepend` | 파일 앞에 추가 | `applyPrepend` |

파이프라인: `validateLineRef`로 현재 내용 대조(스테일 라인 거부) → `dedupeEdits` → `detectOverlappingRanges` → `applyHashlineEditsWithReport` → `generateUnifiedDiff`. `autocorrectReplacementLines`가 접두어·들여쓰기·echo를 제거하고, `canonicalizeFileText`/`restoreFileText`가 BOM과 LF/CRLF을 보존한다. 청크 포매터는 **200줄 / 64KB** 단위로 잘라 읽기 토큰을 제어한다.

### 7.3 read-enhancer 훅
`packages/omo-opencode/src/hooks/hashline-read-enhancer/`(3 파일)가 모델이 파일을 읽을 때 **해시 태그를 주입**한다. 이것이 없으면 모델이 `LINE#HASH|`를 절대 볼 수 없다.

동작:
1. `shouldProcess()` → `config.hashline_edit?.enabled ?? false`. **기본 off.**
2. `read` 툴 출력만 처리 (`isReadTool`)
3. `isTextFile()` — 첫 줄이 `/^\s*(\d+): ?(.*)$/`(콜론형) 또는 `/^\s*(\d+)\| ?(.*)$/`(파이프형)이면 텍스트 파일로 간주
4. `<content>`…`</content>` 블록 안에서만 줄 단위 변환. `<file>` 태그도 추적
5. `transformLine()` → `computeLineHash(lineNumber, content)` 후 `${n}#${hash}|${content}`
6. **가드**: `... (line truncated to 2000 chars)` 접미사가 붙은 줄은 **건너뛴다** — 잘린 내용으로 해시를 계산하면 검증이 영구히 실패한다
7. `WRITE_SUCCESS_MARKER = "File written successfully."` — write 툴 경로는 별도로 식별

### 7.4 게이트
`hashline_edit: z.boolean().optional()` — **default false**(`config/schema/oh-my-opencode-config.ts:59`). 켜면 다음이 함께 활성화된다: `edit` 툴 등록(`tool-registry-gated-tools.ts`), read-enhancer 훅(`create-tool-guard-hooks.ts`), `hooks/atlas/write-edit-tool-policy.ts`의 편집 정책. `config-migration/transform-opencode.ts`가 이전 설정 형태를 변환한다.

---

## 8. LSP: MCP 서버 vs 내장 툴

OmO에서 LSP는 **두 겹으로** 있다.

| 층 | 구현 | 성격 |
|---|---|---|
| MCP 서버 | `packages/lsp-tools-mcp/` (Tier-1 `lsp`, stdio) | 모델에 `lsp_*`로 노출. `disabled_mcps`로 끌 수 있음 |
| 공유 데몬 | `packages/lsp-daemon/` (**5,849줄**) | 유저당 **하나**의 LSP 프로세스를 여러 세션이 공유 |

`lsp-tools-mcp/`는 추출된 `packages/lsp-core/` + `packages/mcp-stdio-core/`를 소비하고, 이 디렉터리는 **OpenCode MCP 설정 빌드만** 담당한다. 상류 프로젝트는 `code-yeongyu/lsp-tools-mcp`. `lsp.ts`가 CLI 경로를 동적으로 해석해 `src/`와 `dist/` 레이아웃 양쪽을 지원한다.

**공유 데몬 설정**: `OMO_LSP_DAEMON_DIR` 또는 짝지어진 `OMO_LSP_DAEMON_CLI` + `OMO_LSP_DAEMON_VERSION` 오버라이드로만 설정된다. 소스 모드는 Bun으로 `packages/lsp-daemon/src/cli.ts`를 돌리고 paired override를 설정하고, dist 모드는 public `@code-yeongyu/lsp-daemon/cli` export를 해석한다(생성된 daemon 파일을 deep-run 하지 않는다).

### 8.1 테스트가 곧 기능 목록 (`packages/lsp-tools-mcp/test/`)
이 레포에서 `test/directory-diagnostics.test.ts`의 존재는 기능 그 자체의 증거다.

| 테스트 파일 | 검증하는 능력 |
|---|---|
| `directory-diagnostics.test.ts` | **디렉터리 단위 진단** — 파일이 아니라 디렉터리를 넘기는 진단. `topics/lsp-lint.md`가 pi 생태 대비 우위로 지목하는 지점 |
| `workspace-edit.test.ts` | rename 등 워크스페이스 편집 |
| `transport-security.test.ts` | **전송 보안** |
| `server-installation-security.test.ts` | **서버 설치 보안** |
| `config-loader-security.test.ts` | 설정 로더 보안 |
| `initialize-timeout.test.ts` | 초기화 타임아웃 복원 |
| `startup-failure.test.ts` | 시작 실패 복원 |
| `mcp-protocol-pin.test.ts` (ast-grep-mcp) | 프로토콜 핀 |

**보안 테스트 3개가 같은 디렉터리에 있는 것**이 중요하다. LSP 서버는 (a) 설정 파일을 읽고 (b) 외부 프로세스를 spawn하고 (c) 네트워크/stdio로 토큰을 왕복하므로, 전송·설치·설정 로더가 각각 별도 공격면이 된다. 기능 하나를 내면서 그 기능의 공격면을 테스트로 못 박는 하네스는 8개 중 이례적이다.

### 8.2 디렉터리 진단이 왜 다른가
파일 단위 `lsp_diagnostics`는 "이 파일의 타입/린트 오류"만 준다. 디렉터리 단위는 **언어 서버 인덱싱 상태를 신뢰할 수 있는지**까지 말해준다 — 인덱스가 깨졌는지, 서버가 살아 있는지, 그 디렉터리에 아예 서버가 붙어 있지 않은지. 규모가 있는 저장소에서 "탐색 단계에서 무엇을 믿어도 되는가"를 판단하는 유일한 신호다.

---

## 9. ast-grep 분기: 같은 기능, 다른 표면

| 어댑터 | 표면 | 구현 |
|---|---|---|
| **OpenCode** (`omo-opencode`) | **스킬** (`sg` CLI) | `packages/shared-skills/skills/ast-grep/SKILL.md` + `scripts/ast_grep_helper.py` + `install.sh`/`install.ps1` |
| **Senpi** (`omo-senpi`) | **MCP** | `packages/ast-grep-mcp/src/mcp.ts` → `components/ast-grep/index.ts`가 등록 |

`omo-senpi/src/components/ast-grep/index.ts`에서 서버 이름은 **`_ast_grep`**(앞의 `_`는 lazy/숨김 표시)이고, 엔트리 경로를 패키징 레이아웃(`../runtime/ast-grep-mcp/cli.js`)과 소스 트리(`../../../plugin/runtime/ast-grep-mcp/cli.js`) 두 곳에서 시도한다. Senpi의 초깃값 지침(`init-deep-pre-amendment.md`)은 "**ast-grep 스킬(`sg` with `$VAR`/`$$$`) 또는 `ast_grep` MCP(`search`/`scan`)** — LSP와 동급의 first-class 동료"로 명시하며, 둘 중 있는 것을 쓰라고 지시한다.

`ast-grep-mcp/`는 스킬 경유가 안 되는 환경(멀티모달·단일 툴 노출)에 필요한 대안이며, 테스트도 MCP 계약에 집중한다: `mcp-tools.test.ts`, `mcp-schema.test.ts`, `mcp-protocol-pin.test.ts`, `mcp-idle-timeout.test.ts`, `mcp-startup.test.ts`, `mcp-timeout.test.ts`, `sg-runner.test.ts`, `pattern-hints.test.ts`.

**분기의 이유**: OpenCode는 이미 `bash`로 CLI를 자유롭게 돌릴 수 있으므로 스킬+CLI가 더 싸고 사용자 오버라이드가 쉽다. Senpi는 `bash`를 `eval` 커널로 대체하면서 명령 접근이 제한되므로, 같은 기능을 MCP로 승격시켜 툴로 노출한다. 기능은 하나, 노출 표면만 다르다.

---

## 10. 다른 7개 하니스 대비

출처: [topics/skills.md](../../topics/skills.md) · [topics/mcp.md](../../topics/mcp.md) · [topics/lsp-lint.md](../../topics/lsp-lint.md)

| 하니스 | 내장 스킬 | MCP 기본 서버 | MCP exposure 모델 | MCP tier | LSP 형태 |
|---|---|---|---|---|---|
| **oh-my-openagent** | **13 TS + 18 shared** | **4** (websearch/context7/grep_app/lsp) | 3-tier 전부 모델 노출 | **3** (내장/`.mcp.json`/스킬) | `lsp-tools-mcp` MCP + 공유 `lsp-daemon` |
| opencode | 1 (`customize-opencode`) | 없음 (hosted Exa/Parallel만) | 전부 노출, `server_tool` 네이밍 | 없음 | 코어 내장 `lsp_*` 9종 |
| pi-mono | **0** (의도적) | 없음 | **기본 `codemode`** (스크립트에서만 접근) | 없음 (확장으로 교체 가능) | 코어 없음 (`pi-lsp-client` 확장) |
| oh-my-pi | 0 (rules 26개) | 없음 | essential/discoverable/declared **3-tier** | **3** (9 외부 포맷 인식) | 내장 기본 off (`lsp`/`ast_grep`/`ast_edit`) |
| codex | 5 | 없음 (Codex Apps) | Direct/DirectModelOnly/Deferred + `tool_search` | 없음 | **없음** (`lsp-types`는 starlark의 전이 의존성) |
| claude-code | 바이너리 내장 | 없음 (claude.ai 커넥터) | `alwaysLoad:false`로 deferred | 없음 | 내장 툴 + 플러그인 `.lsp.json` |
| openclaw | **48** | 없음 | profile + tool-policy 파이프라인 | 없음 | 번들 경유 (`bundle-lsp.ts`) |
| hermes-agent | **58** + optional 152 | 없음 (65 recipe) | Tool Search로 deferred | 없음 | 내장 `agent/lsp/` 8모듈 |

### 8개 하니스 중 OmO만의 지점

| 지점 | 왜 독특한가 |
|---|---|
| **유일한 3-tier MCP** | 나머지 7개는 기본 MCP 서버가 0개다. oh-my-pi의 3-tier는 MCP가 아니라 **노출 등급**(essential/discoverable/declared)이고, OmO의 3-tier는 **소스 계층**이다. 같은 숫자, 다른 축. |
| **유일한 MCP 기본 서버 보유** | `grep_app`은 8개 하니스 중 이 레포에만 있는 GitHub 코드 검색 MCP다. 웹 검색·문서 조회·코드 검색을 기본 제공하므로 "MCP 붙이러" 하니스의 실측 기준선이 된다. |
| **유일한 스킬 수 이원화** | TS 컴파일 내장 + `SKILL.md` 번들이 공존. 단일 번들(Claude Code·codex)이나 단일 런타임 로딩(pi·omp) 구조가 아니다. |
| **유일한 SCOPE_PRIORITY 숫자 병합** | 다른 하니스는 "같은 이름이면 X가 이긴다"는 자연어 규칙만 문서화. OmO만 `merger/scope-priority.ts`에 **상수 테이블**이 있다. |
| **skill_mcp를 툴로 승격** | Tier-3가 프롬프트에 스킬 목록으로만 드러나는 게 아니라 `skill_mcp`라는 **네이티브 툴**로 등록되어 있다. |
| **LSP가 툴이 아니라 MCP 서버** | opencode·omp는 LSP를 툴로 노출. OmO는 `lsp-tools-mcp`를 Tier-1 MCP로 올린다 — 그래서 `disabled_mcps`로 끄면 LSP가 함께 죽고, 반대로 프롬프트 네이밍이 `lsp_*`가 아니라 MCP 규약을 따른다. |
| **디렉터리 단위 진단** | `topics/lsp-lint.md`가 pi 생태 대비 우위로 지목한 유일한 기능. |
| **환경변수 allowlist의 체크인 무력화** | project `.omo/omo.jsonc`의 `mcp_env_allowlist`를 `protectUserFields`가 폐기한다. 다른 하니스에서 같은 기능은 project 설정으로도 열쇠가 돌아간다. |

### 정리
OmO는 **"플러그인 레이어에서 기능을 최대한 많이 노출"** 하는 하니스다. 다른 7개가 노출을 줄이는 방향(`codemode` 기본, `deferred`, `direct`만 선언, Tool Search로 지연)이라면 OmO는 12개 항상-툴 + 최대 26개 조건부-툴 + 4개 MCP 서버 + 31개 스킬을 겹쳐 쌓는다. 대신 그 폭을 지키는 장치가 프론트매터가 아니라 **상수와 병합 로직**이다 — `SCOPE_PRIORITY`, `protectUserFields`, `${sessionID}:${skillName}:${serverName}`, `env-cleaner.ts`. 문서화보다 코드로 규칙이 굳어 있는 하니스이며, 그래서 이 레포의 `AGENTS.md`가 1차 출처가 된다.