# 10 — `look_at` vs `multimodal-looker`, 그리고 네이티브 전용 컴퓨터 유즈

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13`
> 상위 문서: [oh-my-openagent.md](../oh-my-openagent.md) · 교차 비교: [topics/vision-computer-use.md](../../topics/vision-computer-use.md)

OmO에는 "이미지를 본다"가 **세 개 층**으로 나뉘어 있고, "데스크톱을 만진다"는 **오직 네이티브 타깃에만** 존재한다.
이 문서는 두 가지 질문에 집중한다. (A) `look_at` 툴과 `multimodal-looker` 에이전트의 정확한 차이, (B) 네이티브 타깃만 지원하는 컴퓨터 유즈 접근법.

## 한눈에 보기

| 축 | 핵심 사실 |
|---|---|
| 비전 층위 | **3개** — `multimodal-looker`(에이전트) / `look_at`(툴) / 폴백 체인(모델 해석) |
| 에이전트 제약 | `temperature: 0.1` · 툴 allowlist `["read"]` 단 하나 · `mode: "subagent"` · 프롬프트가 "툴을 쓰지 마라"고 명시 |
| 툴 제약 | 기본 `READ_ENABLED = false` — 파일은 **경로 로드가 아니라 메시지 첨부**로 전달 |
| 재귀 방지 | 자식 세션에서 `task` / `call_omo_agent` / `look_at` 세 개를 모두 `false`로 강제 |
| 폴링 | 1초 간격 · 120초 타임아웃 · idle 3회 안정 확인 |
| 컴퓨저 유즈 | `computer.*` 12개 설정 키 **전부 `["native"]`** — OpenCode 플러그인 타깃엔 존재하지 않음 |
| 안전 | 2단 권한(`computer:read`/`computer:exec`) · 강제 정지 화음 · 감사 로그 · **샌드박스가 아님** |

---

# PART A — `look_at` vs `multimodal-looker`

## A0. 먼저 정정: "두 개"가 아니라 "세 개"다

대부분의 서술은 이 둘을 한 쌍으로 묶는다. 실제로는 **모델 해석 층이 세 번째 축**으로 존재하고, 이 층이 있어야 앞의 둘이 성립한다.

```
┌─ 3층: 모델 레벨 레지스턴스 ───────────────────────────────┐
│  multimodal-fallback-chain.ts / multimodal-agent-metadata.ts │
│  hooks/runtime-fallback/agent-resolver.ts                  │
│  "비전 capable 모델이 뭐가 있나, 어떤 순서로 폴백하나"       │
└───────────────┬───────────────────────────────────────────┘
                │ 해석된 모델을 주입
┌───────────────▼───────────────────────────────────────────┐
│  2층: look_at — 툴 (오케스트레이션)                          │
│  tools/look-at/  26파일                                      │
│  인수 정규화 → 입력 준비 → 자식 세션 스폰 → 폴링 → 텍스트 회수 │
└───────────────┬───────────────────────────────────────────┘
                │ 프롬프트 + 파일 첨부로 호출
┌───────────────▼───────────────────────────────────────────┐
│  1층: multimodal-looker — 에이전트 (능력)                   │
│  agents/multimodal-looker.ts  temp 0.1, read 전용           │
└────────────────────────────────────────────────────────────┘
```

---

## A1. 1층 — `multimodal-looker` 에이전트

**경로**: `packages/omo-opencode/src/agents/multimodal-looker.ts` (62줄, 전체 읽음)

```typescript
// multimodal-looker.ts:5
const MODE: AgentMode = "subagent"

// multimodal-looker.ts:7-12
export const MULTIMODAL_LOOKER_PROMPT_METADATA: AgentPromptMetadata = {
  category: "utility",
  cost: "CHEAP",
  promptAlias: "Multimodal Looker",
  triggers: [],          // ← 명시적 트리거 없음. 즉 "직접 부르지 마라"
}

// multimodal-looker.ts:14-22
export function createMultimodalLookerAgent(model: string): AgentConfig {
  const restrictions = createAgentToolAllowlist(["read"])
  return {
    mode: MODE,
    model,                 // ← 모델은 주입받는다 (factory 패턴)
    temperature: 0.1,
    ...restrictions,
    ...
```

`createAgentToolAllowlist`의 실제 구현(`packages/omo-opencode/src/shared/permission-compat.ts:31-42`)은 **화이트리스트가 아니라 deny-all 기반**이다.

```typescript
export function createAgentToolAllowlist(allowTools: string[]): PermissionFormat {
  return {
    permission: {
      "*": "deny" as const,
      ...Object.fromEntries(allowTools.map((tool) => [tool, "allow" as const])),
    },
  }
}
```

즉 이 에이전트의 권한은 `{"*": "deny", "read": "allow"}`다. `write`도 `edit`도 `bash`도 `task`도 없다.

### 프롬프트가 이미 자기 사용법을 부정한다

`multimodal-looker.ts:26`의 두 번째 문장이 이 에이전트의 정체다.

> During look_at invocations, the file or image is already attached to the message. Analyze the attachment directly. **Never call tools, never spawn other agents, and never try to load the file by path.**

그리고 `multimodal-looker.ts:47`:

> The main agent never processes the raw file - **you save context tokens**

이건 지시문이 아니라 **자기 자신의 설계 의도를 기술한 문장**이다. 컨텍스트 절약을 위해 만들어진 일회성 분석 전용 에이전트이며, 그 목적은 "사용자에게 파일 내용을 중계하는 것"이 아니라 "**원본 픽셀/바이트를 메인 컨텍스트에 올리지 않는 것**"이다.

### 등록 지점

| 파일 | 라인 | 내용 |
|---|---|---|
| `agents/builtin-agents.ts` | 10 | `import { createMultimodalLookerAgent, MULTIMODAL_LOOKER_PROMPT_METADATA } from "./multimodal-looker"` |
| `agents/builtin-agents.ts` | 38 | `agentSources` 등록: `"multimodal-looker": createMultimodalLookerAgent` |
| `agents/builtin-agents.ts` | 55 | 프롬프트 메타데이터 등록 |
| `config/schema/agent-names.ts` | 10, 45 | `"multimodal-looker"` — builtin 이름 enum + 프롬프트 표시명 맵 |
| `config/schema/agent-overrides.ts` | 84 | `"multimodal-looker": AgentOverrideConfigSchema.optional()` |
| `shared/agent-tool-restrictions.ts` | 53-55 | `"multimodal-looker": { read: true }` (레거시 제한 테이블. 이 맵의 `read: true`가 factory의 deny-all을 덮는지 여부는 **확인 필요** — 실권한의 1차 출처는 `multimodal-looker.ts:15`의 `createAgentToolAllowlist(["read"])`다) |
| `features/team-mode/types.ts` | — | `AGENT_ELIGIBILITY_REGISTRY`에서 `multimodal-looker`는 **hard-reject** (root `AGENTS.md` "TEAM MODE" 절 명시). read-only 에이전트는 TeamSpec 파싱 시점에 거부되고, lead는 `task`로만 위임한다 |

> **주의**: `tools/look-at/AGENTS.md:47`은 에이전트가 `src/agents/builtin-agents/multimodal-looker.ts`에 있다고 적어 있으나 **오답**이다. `builtin-agents/multimodal-looker.ts`는 존재하지 않으며 실제 경로는 `agents/multimodal-looker.ts`다(`builtin-agents.ts:10`의 import가 증명). 이 문서를 추적할 때 이 한 줄에 막히지 말 것.

### 모델 폴백 체인

**경로**: `packages/model-core/src/agent-model-requirements.ts:78-85`

```typescript
"multimodal-looker": {
  fallbackChain: [
    { providers: ["openai", "chatgpt-subscription", "opencode"], model: "gpt-5.6-sol", variant: "low" },
    { providers: ["opencode-go"],                              model: "kimi-k3" },
    { providers: ["zai-coding-plan"],                          model: "glm-4.6v" },
    { providers: ["openai", "chatgpt-subscription", "github-copilot", "opencode"], model: "gpt-5-nano" },
  ],
},
```

즉 **GPT-5.6 Sol(low) → Kimi K3 → GLM-4.6V → GPT-5 Nano**. 전 구간이 비전 capable 이어야 한다. 하드코딩된 폴백 체인은 마지막 사다리일 뿐이고, 실제 런타임 체인은 다음 A3 층이 만든다.

---

## A2. 2층 — `look_at` 툴

**경로**: `packages/omo-opencode/src/tools/look-at/` (26파일, `AGENTS.md` 생성일 2026-05-18)

### 실행 흐름 6단계

`tools/look-at/AGENTS.md`의 명세:

1. **인수 정규화** (`look-at-arguments.ts`) — `file_path`/`image_data` 별칭 정규화, one-of 검증, **원격 URL 거부**
2. **입력 준비** (`look-at-input-preparer.ts`) — 경로 해석, 확장자 또는 Base64 헤더에서 MIME 감지, 미지원 포맷은 JPEG로 변환 (`image-converter.ts`: HEIC/WebP/RAW/PSD → sips 또는 ImageMagick)
3. **자식 세션 스폰** (`look-at-session-runner.ts`) — `multimodal-looker` 에이전트로 세션 생성, 파일을 **메시지 파트**로 첨부, `task`/`call_omo_agent`/`look_at` 비활성화
4. **폴링** (`session-poller.ts`) — 1초 간격, 120초 타임아웃
5. **텍스트 추출** (`assistant-message-extractor.ts`) — 최신 assistant 텍스트
6. **반환** — 요약 텍스트만

### 실제 스폰 코드

`look-at-session-runner.ts:34-84`:

```typescript
const createResult = await ctx.client.session.create({
  body: {
    parentID: toolContext.sessionID,
    title: `look_at: ${goal.substring(0, 50)}`,
  },
  query: { directory: parentDirectory },   // 부모 세션의 디렉터리 상속
})
...
await promptSyncWithModelSuggestionRetry(ctx.client, {
  path: { id: sessionID },
  body: {
    agent: MULTIMODAL_LOOKER_AGENT,
    tools: {
      task: false,            // ← 재귀 차단
      call_omo_agent: false,  // ← 재귀 차단
      look_at: false,         // ← 자기 자신 재귀 차단
      read: READ_ENABLED,     // ← false
    },
    parts: [
      { type: "text", text: prompt },
      ...inputParts,          // ← 파일이 여기서 첨부된다
    ],
    ...(agentModel ? { model: {...} } : {}),
    ...(agentVariant ? { variant: agentVariant } : {}),
  },
}, { queueBehavior: "defer" })
```

**`tools` 맵이 핵심이다.** 에이전트 자체 권한은 `read` 하나 허용이지만, `look_at`가 스폰할 때는 **`read`조차 `false`** 로 내린다. 파일은 경로가 아니라 **메시지 첨부**로 전달되므로 읽을 필요가 없기 때문이다.

### `READ_ENABLED = false`의 의미

`look-at-prompt.ts:3` 전체가 32줄짜리 프롬프트 빌더이고, 첫 줄이 플래그다.

```typescript
export const READ_ENABLED = false

function sanitizeFilename(filename: string): string {
  const basename = filename.split("/").pop() ?? filename
  return basename.replace(/[^a-zA-Z0-9._-]/g, "_").slice(0, 100)
}
```

`buildLookAtPrompt`은 `READ_ENABLED` 값에 따라 **서로 다른 문장을 삽입**한다 (`look-at-prompt.ts:19-21`):

- `true`일 때: `"Use the Read tool on the provided file path to load its contents, then analyze it."`
- `false`(현재): `"The attached file/image is already included in this message. Analyze it directly from the attachment. **Do NOT attempt to load by path** — the file/image cannot be loaded by path."`

`false`인 이유는 프롬프트가 아니라 **설정**에 있다. 디렉터리 목록이나 `file_path`가 프롬프트에 그대로 새어 들어가면 그 문자열이 모델의 명령으로 오인될 수 있어, basename만 취하고 허용 문자 외는 `_`로 바꾸고 100자에서 자른다. 복수 파일일 때만 `File 1: …, File 2: …` 라벨 목록을 덧붙인다.

### 폴링 세부

`session-poller.ts` 상단 상수:

```typescript
const DEFAULT_POLL_INTERVAL_MS = 1000
const DEFAULT_TIMEOUT_MS = 120_000
const IDLE_STABILITY_POLLS_REQUIRED = 3
const TERMINAL_STATUSES = new Set(["idle", "interrupted", "error"])
```

- 세션 생성 직후엔 `session.status()` 맵에 아직 안 올라와 있다. 코드는 이를 "아직 시작 중"으로 취급한다(`supportedButNeverSeen`, 106행).
- idle 상태에서 **메시지 수가 3회 연속 동일**해야 종료로 인정한다(126·145행). 한 번의 idle은 스트라이크로 안 친다.
- `abortSignal`이 걸리면 `session.abort()`로 자식 세션을 정리한 뒤 던진다(92-95행).
- 10회마다(≈10초) 진행 로그를 남긴다(151-158행).
- 120초를 넘기면 `Polling timed out after 120000ms …` 예외.

### 등록 게이트

`packages/omo-opencode/src/plugin/tool-registry-core-tools.ts:38`

```typescript
const isMultimodalLookerEnabled = !(pluginConfig.disabled_agents ?? []).some(...)
...
if (isMultimodalLookerEnabled) {          // 138행
  tools.look_at = factories.createLookAt(ctx)
}
```

에이전트를 끄면 툴도 같이 사라진다. 역방향은 성립하지 않는다 — 툴만单独으로 끌 수 없다. 이 게이트는 `tools/AGENTS.md`가 "Conditional / 최대 +26개"로 세는 26개 중 하나다.

**다른 참조 지점**: `hooks/json-error-recovery/hook.ts:9`에서 `look_at`가 `JSON_ERROR_TOOL_EXCLUDE_LIST`에 들어 있다. 즉 `look_at` 결과가 JSON 파싱 실패를 일으켜도 JSON 복구 훅이 개입하지 않는다.

---

## A3. 3층 — 모델 레벨 레지스턴스

이 층이 없으면 `multimodal-looker`는 "어떤 모델인지"를 모른다. 세 파일이 그 역할을 나눈다.

### `multimodal-fallback-chain.ts` — 폴백 체인 조립

`packages/omo-opencode/src/tools/look-at/multimodal-fallback-chain.ts` (66줄, 전체 읽음)

`buildMultimodalLookerFallbackChain(visionCapableModels)`의 로직은 **두 번 순회**한다.

1. **1차 순회** — 비전 capable 모델 캐시(`readVisionCapableModelsCache`에서 옴)를 넣는다. 중복 키는 `seen`으로 제거. 하드코딩 체인에 같은 `{providerID, modelID}`가 있으면 `variant`만 빌려온다(`hardcodedEntry?.variant`).
2. **2차 순회** — 하드코딩 체인(`gpt-5.6-sol low → kimi-k3 → glm-4.6v → gpt-5-nano`)을 뒤에 붙인다. 단 **모든 provider 변형이 이미 `seen`에 있으면 통째로 건너뛴다**(55행).

결과적으로 **사용자가 실제로 연결된 비전 모델이 체인 앞쪽에 오고, 하드코딩 체인은 뒤쪽 안전망이 된다.** 환경에 따라 순서가 달라지는 이유다.

`isHardcodedMultimodalFallbackModel()`도 같은 소스를 읽어, 이 모델이 하드코딩 체인에 이미 있는지를 판별한다.

### `multimodal-agent-metadata.ts` — 실제 모델 1개 해석

`tools/look-at/multimodal-agent-metadata.ts`. 위에서 만든 체인을 쓰고, 최종적으로 `{ agentModel, agentVariant }`를 돌려준다(`look-at-session-runner.ts:26`에서 소비).

- `isVisionCapableAgentModel()`로 검증 — 해석된 모델이 비전 capable 목록에 없으면 그대로 쓰지 않는다.
- 참조하는 것: `shared/model-availability`(`fetchAvailableModels`), `shared/connected-providers-cache`, `shared/model-resolution-pipeline`(`resolveModelPipeline`), `shared/vision-capable-models-cache`.
- 결과가 `undefined`일 수도 있고, 그 경우 `look-at-session-runner.ts:79-80`의 조건부 스프레드가 **모델/variant 지정 자체를 생략**한다. 즉 자식 세션은 에이전트 설정이 정한 기본 모델로 떨어간다.

### `agent-resolver.ts` — 세션 귀속 감지

`packages/omo-opencode/src/hooks/runtime-fallback/agent-resolver.ts:3-17`

```typescript
export const AGENT_NAMES = [
  "sisyphus", "oracle", "librarian", "explore", "prometheus", "atlas",
  "metis", "momus", "hephaestus", "sisyphus-junior", "build", "plan",
  "multimodal-looker",       // ← 13개 중 마지막
]
```

`detectAgentFromSession(sessionID)`는 세션 ID 문자열에 이 이름이 들어 있는지 정규식으로 찾아 에이전트를 결정한다. `build`/`plan`이 다른 11개 builtin과 이름이 겹치지 않는다는 점이 눈에 띈다(런타임 에이전트명).

이 배열은 **에이전트 인벤토리가 아니라 runtime-fallback 훅이 아는 이름 목록**이다. `multimodal-looker`가 여기 있다는 것은 `look_at`가 스폰한 자식 세션이 **폴백 훅의 관리 대상**이라는 뜻이다 — 비전 분석이 실패하면 런타임 폴백이 개입할 수 있다.

---

## A4. 정면 비교표

| 항목 | `multimodal-looker` (에이전트) | `look_at` (툴) | 모델 레지스턴스 (폴백) |
|---|---|---|---|
| **책임 분리** | 첨부된 미디어를 해석해 **요약 텍스트만** 반환 | 인수 정규화 → 포맷 변환 → 자식 세션 스폰 → 폴링 → 텍스트 회수 | 어떤 비전 모델을 어떤 순서로 쓸지 결정 |
| **실행 주체** | LLM이 단독 실행 (세션 단위) | 메인 세션의 모델이 호출하는 툴 | 코드. LLM 없음 |
| **모델** | 주입됨 (`createMultimodalLookerAgent(model)`). 폴백 `gpt-5.6-sol low → kimi-k3 → glm-4.6v → gpt-5-nano` | 모델을 **직접 선택**. `resolveMultimodalLookerAgentMetadata()` 결과를 `model`/`variant`로 주입 | `vision-capable-models-cache` + 하드코딩 체인을 병합해 조립 |
| **툴 접근** | `createAgentToolAllowlist(["read"])` = `{"*": "deny", "read": "allow"}` | 자식 세션에서 `task`/`call_omo_agent`/`look_at` = `false`, `read` = `READ_ENABLED`(`false`) | 해당 없음 (훅 레벨) |
| **컨텍스트 격리** | **격리 자체가 목적**. "The main agent never processes the raw file" | 부모 세션을 `parentID`로 연결하되 **별도 세션**. 원본은 메인 컨텍스트에 안 들어감 | 해당 없음 |
| **실패 처리** | 프롬프트 차원에서 "못 찾으면 뭐가 없는지 명시" | 120초 타임아웃 예외, `unauthorized` 세션 생성 실패 전용 메시지(`look-at-session-runner.ts:46-53`), "No response from multimodal-looker agent" | 체인이 비면 다음 rung으로. 없으면 해석 결과가 `undefined`가 되어 모델 지정 자체가 생략됨 |
| **재귀/에스컬레이션 차단** | 프롬프트로 "절대 툴 호출/에이전트 생성 금지" | 코드 레벨 강제(`tools: { task:false, call_omo_agent:false, look_at:false }`) | 해당 없음 |
| **사용자가 직접 호출 가능한지** | **구조적으로는 가능하나 실질적으로 퇴화** (아래 참조) | **네.** 메인 모델이 자연 호출. 사용자는 프롬프트로 유발 | 불가 (사용자 비노출) |

### "사용자가 `multimodal-looker`를 직접 호출할 수 있는가" — 구조적으로 예, 실질적으로 열려 있음

확인된 사실:

1. `task` 툴의 `subagent_type`는 **자유 문자열**이다. `tools/delegate-task/tools.ts:73` — `tool.schema.string().optional()`. enum 제약이 없다.
2. 유일한 차단 로직은 coordinator 가드이고, 대상은 **단 하나**다. `tools/delegate-task/constants.ts:405` — `export const COORDINATOR_AGENT_NAMES = ["prometheus"]`. `isCoordinatorAgent()`도 그것만 검사한다.
3. `call_omo_agent`은 명시적으로 explore/librarian만 허용한다(`tools/AGENTS.md` "Always On" 표의 "named agent only: explore, librarian").
4. Team Mode는 `AGENT_ELIGIBILITY_REGISTRY`에서 hard-reject다.

따라서 `task(subagent_type="multimodal-looker")`는 **가드에서 걸리지 않는다.** 그런데도 이 경로는 권장되지 않는다. 프롬프트의 전제("the file or image is already attached to the message")가 깨지기 때문이다. `look_at`가 해주는 **첨부·포맷 변환·MIME 감지**를 bypass하면 에이전트는 temp 0.1에 read 외 툴 전부 차단된 채 텍스트 프롬프트만 받게 된다.

> 실제 `resolveSubagentExecution`이 `multimodal-looker`를 어떻게 처리하는지(`tools/delegate-task/subagent-resolver.ts` 전문)는 이번에 읽지 않았다 — **확인 필요**. 위 판단은 스키마가 enum이 아니고(`tools.ts:73`) coordinator 가드가 `prometheus`만 막는다는(`constants.ts:405`) 구조적 사실에 기반한다.

### 왜 3층으로 나눴나

| 층 | 분리 이유 |
|---|---|
| 에이전트 ↔ 툴 | **오염된 메인 컨텍스트 방지**. 이미지가 메인 컨텍스트에 들어가면 토큰을 먹고 컴팩션 압축 대상이 된다. `look_at`는 자식 세션에서 분석하고 텍스트만 되돌린다. |
| 에이전트 ↔ 모델 | **능력(capability) 정체성**. `category: "utility"` + `cost: "CHEAP"` + temp 0.1 + read 전용은 "廉价한 관찰자"라는 정체성의 일부다. 모델이 바뀌어도 에이전트 정체성은 유지된다. |
| 모델 ↔ 나머지 | **폴백의 비전 보장**. 일반 에이전트 폴백 체인은 비전 capable을 보장하지 않는다. `multimodal-fallback-chain.ts`는 캐시된 비전 모델을 앞에 넣고 하드코딩 체인을 뒤에 붙여, **항상 비전 capable만 체인에 들어가는** 것을 구조적으로 보장한다. |

### 비용

| 항목 | 값 |
|---|---|
| 세션 스폰 | 호출 1회당 자식 세션 1개 (`session.create` + `parentID` 연결) |
| 추가 지연 | 폴링 간격 1초 + idle 안정 확인 3회 → **최소 약 3초의 유휴 대기** |
| 상한 | 120초 타임아웃. 그 안에 안 끝나면 예외로 죽는다 |
| 결과 크기 | 첨부 대신 텍스트 요약만 메인으로 복귀 — 이것이 절약한 토큰 |

**120초가 하드 천장이라는 점이 실무상 중요하다.** 40페이지 PDF나 고해상도 다중 이미지 분석은 이 천장에 걸릴 수 있고, 그때는 부분 결과가 아니라 예외만 남는다.

---

## A5. 정직성 노트: 왜 정밀 작업이 `Read` 툴로 넘어가는가

`tools/look-at/AGENTS.md:7`:

> This is **a summary extractor, not a precise reader**.

같은 문서가 43행에 이유를 붙인다:

> PDFs, screenshots, diagrams — quick summary extraction. **NOT for visual precision, aesthetic evaluation, or exact accuracy. Use the Read tool for those cases instead.**

그리고 툴 설명 자체(`tools/look-at/constants.ts`)가 모델에게 직접 말한다:

> Extract basic information from media files (PDFs, images, diagrams) **when a quick summary suffices over precise reading**. … **NEVER use for visual precision, aesthetic evaluation, or exact accuracy - use Read tool instead for those cases.**

이 스코프 결정의 이유는 세 층 구조에서 나온다. 요약 추출은 **텍스트 변환**이라 LLM이 정확히 하는 일이다. 반면 픽셀 단위 판단(색상 값, 정렬, 1px 테두리, 렌더링 아티팩트)은 텍스트 요약의 손실이 곧 정답의 손실이다. 자식 세션이 텍스트를 반환하는 구조 자체가 정밀도를 상한으로 못 박는다 — 아무리 좋은 비전 모델을 붙여도 **텍스트로 되돌리는 한계를 넘어설 수 없다.**

그래서 정확한 비전 작업은 `Read` 툴이 한다. OpenCode 코어의 `read` 툴은 `image/jpeg|png|gif|webp`를 그대로 모델에 붙이고(`packages/core/src/tool/read.ts`), 리사이즈는 photon WASM + Lanczos3로 `packages/core/src/image.ts`가 처리한다. **원본 픽셀이 메인 컨텍스트에 직접 들어간다.** 비싼 대신 정확한 경로다.

비유로 정리하면: `look_at`는 **요약 읽기(skim)**, `Read`는 **정밀 열람(close read)**. `look_at`를 안 쓰는 게 아니라 목적이 다른 것이다.

---

# PART B — 컴퓨터 유즈 (네이티브 타깃 전용)

## B1. 하드 네이티브 게이트

**경로**: `packages/omo-config-core/src/schema/computer.ts:47-60` (60줄, 전체 읽음)

```typescript
export const COMPUTER_HARNESS_SUPPORT: Record<ComputerSettingPath, readonly OmoHarnessId[]> = {
  "computer.enabled":                   ["native"],
  "computer.display":                   ["native"],
  "computer.max_width":                 ["native"],
  "computer.max_height":                ["native"],
  "computer.screenshot_max_bytes":      ["native"],
  "computer.stop_hotkey":               ["native"],
  "computer.allow_host_relay_only_stop":["native"],
  "computer.macos_canary":              ["native"],
  "computer.audit_log":                 ["native"],
  "computer.screenshot_gc":             ["native"],
  "computer.engine_path":               ["native"],
  "computer.cua_adapter":               ["native"],
} as const
```

**12개 키 전부, 예외 없이 `["native"]` 단일 값.** 다른 하니스를 하나도 열거하지 않는다.

### 이게 의미하는 것

- **OpenCode 플러그인 타깃(`packages/omo-opencode`)에는 컴퓨터 유즈가 존재하지 않는다.** 설정 블록을 써도 무시된다. 비활성화되는 게 아니라 애초에 경로가 없다.
- **Codex Light 타깃(`packages/omo-codex`)도 같다.**
- `docs/guide/computer-use.md:7`이 명시한다: *"This feature belongs to OmO Native; the same `computer` block does not enable it in the OpenCode or Codex editions."*
- 반대로 `senpi-desktop-*` 패키지 5개와 `crates/senpi-desktop-*` Rust 크레이트들은 이 게이트의 유일한 고객이다. TS 래퍼와 Rust 엔진이 나뉘어 있는 이유다 — 게이트가 어댑터보다 **엔진 위에** 있기 때문에, 어느 어댑터가 이 블록을 해석할 수 있을지는 독립적인 결정 사항이 된다.

### 왜 이 구조인가

게이트를 설정 스키마 레벨에 박아넣으면 두 가지가 동시에 성립한다. (1) 사용자는 OpenCode에서 `computer.enabled: true`를 써도 조용히 무시되며, 왜 안 되는지 모른다. (2) 네이티브 전용 기능임에도 공용 설정 파일(`~/.omo/omo.jsonc`)에 키가 노출되어 혼선을 만든다.

`COMPUTER_HARNESS_SUPPORT`는 12개 키를 **정적으로 열거**하므로, 설정 검증 단계에서 "이 하니스에서 이 키는 안 된다"를 표현할 수 있다. `OmoComputerSettingsSchema`의 `.describe()`(36-38행)도 그렇게 쓴다: *"Experimental computer use in OmO Native."*

### 관련 패키지 배치

| 패키지 | 역할 |
|---|---|
| `packages/omo-config-core/src/schema/computer.ts` | 타입 있는 네이티브 전용 설정 + 게이트 맵 |
| `packages/omo-senpi/src/components/computer-use/index.ts` | 컴포넌트 등록, 세션 라이프사이클, 리소스 탐색 |
| `packages/senpi-desktop-tool/src/{tool,params,command,permission,settings}.ts` | 툴 액션·검증·권한 티어·기본값 |
| `packages/senpi-desktop-prelude/src/{prelude.js,prelude.py}` | Eval 커널의 `computer` 글로벌 |
| `packages/senpi-desktop-{protocol,service,engine}/src` | 프로토콜, 클라이언트, 엔진 로케이터 |
| `crates/senpi-desktop-backend-{macos,x11,wayland,win32}` | OS별 네이티브 백엔드 |

---

## B2. 액션 모델

**경로**: `docs/reference/computer.md` (64줄, 전체 읽음) + `docs/guide/computer-use.md` (167줄, 전체 읽음)

| `action` | 파라미터 | 반환 |
|---|---|---|
| `call` | `chain`: 데스크톱 헬퍼 `{ method, args? }`, 뒤에 창/엘리먼트 메서드 1개까지 | 헬퍼 결과. 스크린샷은 이미지 콘텐츠도 |
| `run` | `code`: `desktop`/`wait`/`assert`/`tool`을 쓰는 JS async 함수 본문. 선택 `read_only`, `timeout`(초) | 반환값 + displays |
| `capabilities` | 없음 | 백엔드, 캡처/입력/AX 권한, 정지 경로, 포커스 가드 |
| `close` | 없음 | 데스크톱 세션 종료 |

```jsonc
{ "action": "call", "chain": [{ "method": "screenshot" }] }
```

`run`은 지속 세션이라 `await desktop.screenshot()`으로 좌표를 얻고 `await desktop.click(x, y)`로 누르는 식의 다단계 작업이 한 블록에 들어간다.

### `read_only`는 권한 라벨이 아니라 런타임 강제다

`docs/reference/computer.md:28`:

> A `run` marked `read_only: true` is **constrained by the runtime, so attempted input fails rather than escaping the read tier.**

여기서 "권한 레이블"과 "런타임 강제"의 차이가 결정적이다. 권한 라벨이면 프롬프트에 `read_only`를 못 붙여서 exec를 우회할 수 있다. **런타임 강제는 시도 자체를 실패시키므로** 우회가 불가능하다. 권한 분류가 `read`로 판정된 `run` 안에서 입력 호출을 시도하면 그 입력은 실패한다.

### 좌표 안전성

`docs/reference/computer.md:26`:

> Screenshots are limited to a **coordinate-safe size** when passed to a model; a stale frame is rejected with **`InvalidCoordinateFrame`**.

- 스크린샷이 모델에 넘겨질 때 좌표가 안전한 크기로 제한된다. 즉 **스케일링된 이미지의 픽셀을 그대로 클릭 좌표로 쓰면 안 된다.**
- 오래된 프레임은 `InvalidCoordinateFrame`로 거부된다.
- 좌표 두 종류가 섞이지 않는다 (`docs/reference/computer.md:36`): **포인터 좌표 = 같은 타깃의 최신 스크린샷 기준**, **접근성 좌표 = 전역 데스크톱 좌표**. 접근성 스냅샷을 새로 받으면 reference generation이 바뀌고 옛 ref는 `StaleRef`로 실패한다.
- 스크롤 단위는 전 OS에서 **픽셀**. macOS는 픽셀 스크롤 이벤트, Windows/X11은 40px당 1노치(반올림, 최소 1), Wayland는 120분의 1노치. `dy: 120`은 어디서나 약 3노치다.

---

## B3. 안전 모델 — 진짜 차별점

**경로**: `docs/guide/computer-use.md:114-130` ("Safety model" 절) + `docs/reference/computer.md:30-40`

### 1) 2단 권한 — 엔진 이전에 검사

`docs/reference/computer.md:32`:

> `capabilities`, inspection-only `call` chains, and `run` with `read_only: true` use **`computer:read`**. `close`, other calls and runs, unknown methods and malformed inputs use **`computer:exec`**. **The host checks the tier before the engine receives an action, so an exec-denied click is not audited as an engine click.** Rules may target the entire `computer` tool or the `computer:read` and `computer:exec` tiers. **An approval for read never approves exec.**

여기서 결정적인 절이 "before the engine receives an action"이다. **거부된 클릭은 엔진 클릭으로 감사 로그에 남지 않는다.** 반대로 말하면, 엔진 레벨에서 거부를 구현했다면 감사 로그에 "시도했으나 거부됨"이 찍혔어야 하고, 그건 실제 엔진에 전달된 적이 없다는 사실과 모순된다. **거부 지점을 호스트에 두는 것 자체가 감사 로그의 의미를 보존하는 선택**이다.

`docs/guide/computer-use.md:116`이 못 보게만 보는 지점을 더한다: **비대화형 실행은 `ask` 결정에 답할 수 없다.** 즉 자동화 컨텍스트에서 `ask` 권한 티어는 실용상 `deny`와 같다. 실행 전에 명시적으로 허용해 줘야 한다.

몰아보기만 허용하려면:

```sh
omo --permission computer:read=allow --permission computer:exec=deny
```

### 2) 정지 화음 (stop chord)

`docs/guide/computer-use.md:122`:

> While input is possible, a global stop chord is armed (**Control+Option+Command+Escape on macOS, Ctrl+Alt+Shift+Escape on Linux and Windows, or `stop_hotkey`**). Pressing it **suspends all computer input immediately**: a `type` already in progress **stops at the next character**, and an action cut short by the suspension **releases any keys and buttons it was holding**; `/computer stop` does the same from the prompt. Input stays suspended until you run `/computer resume`. **If the chord cannot be armed, input is refused with `StopPathUnavailable` rather than running without a way to stop it**; `allow_host_relay_only_stop: true` accepts `/computer stop` as the only stop path, and is meant for hosts that relay a stop reliably.

설계 철학이 세 겹으로 드러난다.

| 요소 | 의미 |
|---|---|
| **즉시 정지** | 진행 중 `type`이 **다음 글자에서 멈춘다**. 중간에 잘리지 않는다 — 텍스트가 깨지는 일이 없다 |
| **키·버튼 해제** | 중단된 액션이 쥔고 있던 키와 마우스 버튼을 놓는다. modifiers 고착 방지 |
| **`/computer resume`은 사용자만** | `docs/guide/computer-use.md:21` — *"only you can do this, the model has no action that reaches it"*. 모델이 스스로 재개할 수 없다 |
| **정지 경로 없으면 실행 거부** | `StopPathUnavailable`. **"멈출 방법 없는 자동화는 하지 않는다"** 가 실패 모드가 아니라 설계다 |
| **`allow_host_relay_only_stop`** | 정지 화음을 못 거는 호스트(Wayland + GlobalShortcuts 미지원)의 우회책. 대신 `/computer stop` 하나만 정지 경로가 된다. **"정지 가능한 상태로만 실행"이라는 불변식은 유지되고, 경로만 좁아진다** |

### 3) 입력 전 사전 거부

`docs/guide/computer-use.md:124`:

> **Before input reaches an application**, the engine refuses it when input is suspended, the screen is locked, the stop path is gone, or the OS permission is missing.

애플리케이션에 닿기 **전에** 네 가지 조건에서 거부한다. 즉 캡처가 되고 인증을 넘긴 뒤에만 막지 않는다.

### 4) 감사 로그

`docs/guide/computer-use.md:130`:

> Every mutating action is appended to **`.computer-audit.jsonl`** in the session directory. Typed text is recorded by **length and digest, never content**, and an interrupted `type` also records **how many characters were delivered**. Set `audit_log.enabled: false` to turn it off.

| 항목 | 처리 |
|---|---|
| 기록 위치 | 세션 디렉터리의 `.computer-audit.jsonl` (append) |
| 기록 대상 | 모든 변경 액션 |
| 타이핑 텍스트 | **길이 + 다이제스트만. 내용은 절대 안 기록** |
| 중단된 `type` | 전달된 문자 수까지 기록 |

내용을 안 쓰는 건 프라이버시 설계지만 **감사 가능성도 보존한다** — "무엇을 했는가"가 아니라 "얼마나 했는가"가 남으므로, 사후에 "이 세션에서 얼마나 많은 문자가 입력됐는가"를 재구성할 수 있다.

### 5) 프롬프트 레벨 방어

`docs/guide/computer-use.md:126`:

> The computer-use skill instructs the model to treat **screen text, notifications and documents as untrusted data that can never authorize an action**, and to **confirm immediately before consequential actions** (sending or publishing, purchases and transfers, deletion, account, security and permission changes, disclosing private data, accepting terms) unless your own message authorized that exact action. A `Suspended` or `StopPathUnavailable` error means **stop and report**; the model is told **never to work around it**.

프롬프트 인젝션 방어다. 화면에 "이 작업을 승인합니다"라고 적혀 있어도 **승인이 아니다.** 사용자의 실제 메시지가 그 액션을 명시적으로 승인했어야 한다. `Suspended`/`StopPathUnavailable`는 우회가 아니라 중지 후 보고다.

### 6) 정직성 있는 한계 고백 — "샌드박스가 아니다"

**원문** (`docs/guide/computer-use.md:126`, 그대로 인용):

> **These are model instructions, not a sandbox: `computer.run` code runs with full host access**, so keep permission rules strict for untrusted work and **use a separate account or VM for risky automation**.

이 고백이 왜 중요한가. 앞의 6개 항목이 전부 튜토리얼처럼 읽히기 쉽다. 2단 권한, 강제 정지, 감사 로그, 프롬프트 인젝션 방어 — 이게 전부 모이면 **샌드박스가 있는 것**으로 착각하기 쉽다. 실제로는 그렇지 않다.

- 2단 권한은 **호스트의 정책 결정**이지 코드 격리가 아니다. 승인하면 그 액션은 실제 호스트 권한으로 실행된다.
- 정지 화음은 **비상 정지 버튼**이지 봉쇄가 아니다. 실행은 이미 시작됐다.
- 감사 로그는 **사후 기록**이지 사전 차단이 아니다.
- 프롬프트 레벨 방어는 **모델이 잘 따르면** 작동한다. 보장하지 않는다.
- **`computer.run`은 풀 호스트 접근권으로 실행된다.** Eval 커널 안의 JS가 호스트에 마음대로 접근한다.

그래서 문서가 권하는 처방은 HYBRID이다. 위험한 자동화는 **계정 또는 VM을 분리하라.** 권한 규칙을 조여도 계정 하나의 모든 파일에 대한 접근은 남으므로, 진짜 경계는 프로세스 바깥이어야 한다.

이것은 보안 문서가 아니라 **기능 문서**(`guide/`)에 있다. 제품이 자기 기능의 한계를 본문에 박아넣은 흔적이다.

### 7) 포커스 관리

`docs/guide/computer-use.md:128`:

> Input defaults to **background delivery**. On macOS it leaves your **frontmost app, visible front window, keyboard focus, cursor and next-keystroke destination unchanged**, although a clicked target may rise directly beneath your front window. Background typing into an app with multiple windows can report `BackgroundUnavailable`; use accessibility actions or foreground delivery instead. Foreground delivery briefly activates the target and then **restores the previous window and cursor; a failed restore is reported, not hidden**.

| 전이 | 동작 |
|---|---|
| background (기본) | macOS에서 프론트앱·가시 창·키보드 포커스·커서·다음 키 입력 대상을 **모두 그대로 유지**. 단, 클릭된 대상 창은 사용자의 프론트 창 바로 아래로 올라올 수 있음 |
| background 실패 | `BackgroundUnavailable` (다중 창 앱 타이핑, Windows 툴킷 제약, Wayland). 에러에 **금지된 액션명과 툴킷 이름**이 실려 있어서 모델이 스스로 우회로를 고를 수 있다 |
| foreground | 대상을 잠깐 활성화했다가 **이전 창과 커서를 복원** |
| 복원 실패 | **`FocusRestoreFailed` / `CursorRestoreFailed`로 보고한다. 숨기지 않는다** |

"reported, not hidden"이 두 번 나온다. `docs/reference/computer.md:57`의 에러 표도 *"Tell the user; nothing is hidden"*으로 같은 말을 반복한다. 사용자의 데스크톱 상태를 바꾼 뒤 **되돌리지 못한 채 침묵하는 것**을 명시적으로 금지한다.

> 주: `docs/guide/computer-use.md:150`(트러블슈팅)은 macOS에서 "클릭된 창이 앞으로 이동하는" 것을 알려진 한계로 나열하고, 전체 이전 창 순서 복원은 issue [#8930](https://github.com/code-yeongyu/oh-my-openagent/issues/8930)으로 추적 중이라고 명시한다.

### 8) 스크린샷 GC와 관련 설정

`packages/omo-config-core/src/schema/computer.ts:27-30` + `docs/guide/computer-use.md:95-110`

| 키 | 기본값 | 의미 |
|---|---|---|
| `screenshot_gc.enabled` | `true` | 오래된 스크린샷 파일 삭제 |
| `screenshot_gc.stale_ms` | `43200000` (12시간) | 이 나이가 지나면 삭제 |
| `screenshot_gc.scan_interval_ms` | `1800000` (30분) | 스윕 주기 |
| `max_width` | `3840` | 최대 스크린샷 너비(px), 초과 시 축소 |
| `max_height` | `2400` | 최대 스크린샷 높이(px) |
| `screenshot_max_bytes` | `5000000` | 최대 인라인 스크린샷. 초과 시 **파일 경로로만** 반환 |
| `macos_canary` | `"session"` | macOS 배경 전달 체크. `"off"`면 체크와 그 다이얼로그를 건너뜀 |
| `audit_log.enabled` | `true` | 변경 액션 감사 로그 |

**macOS canary**는 눈에 띄는 장치다 (`docs/guide/computer-use.md:60`). 세션의 첫 배경 입력 직전에 "senpi desktop canary"라는 작은 다이얼로그를 띄워 표시된 키 입력 한 번을 보내고 자동 폐기(최대 5초 뒤 자동 종료)한다. 배경 키보드 전달이 실제로 먹히는지 **OmO가 그것에 의존하기 전에 증명하는 것**이다. 실패하면 `stopReason=skylight-canary-failed`로 보고하고 `/computer resume`이 재무장한다.

---

## B4. 엔진 상호운용성

**경로**: `docs/guide/computer-use.md:163-167` + `crates/senpi-desktop-engine/src/cli.rs:15`

```rust
#[command(group(ArgGroup::new("mode").args(
    ["stdio", "serve", "oneshot", "mcp", "resume", "selftest", "schema"])))]
```

| 플래그 | 의미 |
|---|---|
| `--stdio` | NDJSON 위의 JSON-RPC |
| `--serve <socket>` | 데몬 |
| `--oneshot` | 데몬으로 요청 1개를 포워딩 (bunshin 사이드카 계약, `oneshot.rs:1`) |
| `--resume` | **사용자 전용** 정지 복구. 데몬의 0600 토큰 파일을 읽는다 (`client.rs:162`) |
| `--mcp` | `--serve` 데몬 위의 MCP stdio 서버 (tools capability) |
| `--selftest` | 내장 fake 세션을 엔진 자신으로 구동 (`selftest.rs:1`) |
| `--schema` | 스키마 출력 |

`--mcp` 모드에서는 `tools/list`가 엔진 메서드를 `desktop_<method>` 툴로 노출한다(`docs/guide/computer-use.md:165`). 캡처 결과는 MCP 이미지를 포함하고, **입력은 여전히 동일한 안전 게이트를 통과한다.**

> **사용자 전용 `--resume`은 절대 모델 툴로 노출하지 않는다** — 문서가 두 번 명시한다(`docs/guide/computer-use.md:165`, `docs/reference/computer.md:64`). `crates/senpi-desktop-engine/src/mcp/catalog.rs:42`의 `desktop.stop` 설명도 *"only the desktop user resumes"*로 같은 불변식을 새긴다. `--resume`가 모델 도달 가능해지면 "사용자만 재개할 수 있다"는 안전 모델 전체가 무너진다.

엔진 탐색 순서 (`docs/guide/computer-use.md:33`, `packages/senpi-desktop-engine/src/locator.ts`): `computer.engine_path` 설정 → 컴파일된 실행 파일의 사이드카(`native/prebuilds/<host>/senpi-desktop-engine`) → 패키지 네이티브 prebuild → 개발 빌드(`target/release/`).

macOS 격리 속성(`com.apple.quarantine`)을 가진 후보는 **건너뛰고 보고만 하며 해제하지 않는다.** 예외는 하나: 런처 설치 디렉터리 안의 사이드카는 정규경로가 그 런처의 `native/prebuilds/` 아래에 있고 SHA-256이 `native/prebuilds/senpi-desktop-engine-checksums.txt`의 호스트 항목과 일치할 때만 허용된다. 체크섬 파일 누락·불일치·설치 디렉터리 밖으로 빠지는 심볼릭 링크는 거부를 유지한다. **이 예외는 명시적 `computer.engine_path`나 다른 경로에는 적용되지 않는다.**

---

## B5. 크로스 하니스 대비

> 출처: [topics/vision-computer-use.md](../../topics/vision-computer-use.md)

### 컴퓨터 유즈

| 하니스 | 구현 | 게이트/비고 |
|---|---|---|
| **oh-my-openagent** | `computer` + `computer_actions`(OpenAI CUA) + Rust 엔진 `senpi-desktop-engine` | **`COMPUTER_HARNESS_SUPPORT` 전 키 `["native"]`**. OpenCode/Codex 타깃엔 부재 |
| oh-my-pi | `computer` **Eval 프렐류드** | 네이티브 Rust(`crates/pi-natives/src/desktop/`). `computer.enabled` 기본 `false`, `/computer` 토글. **샌드박스 아님** 명시 |
| openclaw | `computer` + `cua-computer` 확장 + Peekaboo | 25개 v2 액션. Rust 크레이트는 데스크톱이 아니라 노드 런타임/게이트웨이 클라이언트 |
| hermes-agent | `computer_use` + `cua-driver` + `bot_desktop` | 단일 action 디스크리미네이터 14액션. 백엔드는 MCP-over-stdio → `cua-driver`. `bot_desktop`은 프로필별 headless Xfce + RFB |
| codex | **설정/게이트만** | `Feature::ComputerUse`는 "Requirements-only gate". 실제 실행은 번들 플러그인 `computer-use@openai-bundled`. OSS 레포에 캡처/입력 크레이트 없음 |
| opencode / pi-mono / claude-code | **없음** | opencode는 `computer_use_call` 타입 passthrough만. claude-code는 CHANGELOG 산문에만 존재 |

**OmO가 유일하게 "설정 스키마에 하드 게이트를 정적 열거"한다.** 다른 하니스는 기능 유무가 어댑터 배치에 흩어져 있다(`packages/senpi-desktop-*` vs `senpi-task`). `COMPUTER_HARNESS_SUPPORT`는 12개 키를 타입 수준에서 강제하므로, 실수로 다른 타깃에 설정을 심는 일이 구조적으로 불가능하다.

### 비전 "이미지를 본다"

| 하니스 | 진입점 | 구조 |
|---|---|---|
| **oh-my-openagent** | `look_at` **위임 툴** | **전용 비전 에이전트 + 비전 모델 폴백 체인** |
| codex | `view_image` | 원본을 메인 모델에 직접. `InputImage` data URL 반환 |
| openclaw | `view_image` + `media-understanding` 엔진 | `describeImageWithModel`, `modelSupportsVision` 게팅 |
| oh-my-pi | `read <image>?q=<질문>` | text-only 모델이면 `local://` 저장 → 비전 모델 폴백 |
| hermes-agent | `vision_analyze` / `video_analyze` | `region` 크롭 + `image_routing.py`(`auto\|native\|text`) 비용 제어 |
| opencode / pi-mono / claude-code | `read`에 이미지 첨부 | 전용 툴 없음 |

**OmO가 유일하게 "메인 모델 대신 전용 비전 모델 라우팅"한다.** 나머지는 메인 모델이 직접 이미지를 본다(또는 cheap fallback 경로). `multimodal-fallback-chain.ts`가 비전 capable 모델만 체인에 넣는 구조는 8개 하니스 중 하나뿐이다. 대가는 그 A4에서 본 비용(세션 스폰 + 폴링 지연 + 120초 천장)이다.

---

## B6. 실용적 한계

### 실험적 단계

`packages/omo-config-core/src/schema/computer.ts:14-18` — `enabled`의 `.describe()` 리터럴:

> **"Experimental**: register the computer tool in OmO Native sessions (default: on where the host is supported; false leaves it unregistered)"

**스키마 설명 필드 자체가 "Experimental"로 시작한다.** 설정 스키마는 사용자에게 노출되는 계약이므로, 여기에 "Experimental"을 넣은 것은 의도된 고백이다. `docs/reference/computer.md:5`도 반복한다: *"Experimental. Computer use is experimental support. The tool contract below may change between releases."*

### 게이트

- **네이티브 전용.** OpenCode/Codex 타깃엔 부재 (§B1).
- **그래픽 데스크톱 세션 필요.** 로그인하지 않은 데스크톱은 못 쓴다. SSH 세션, 디스플레이 없는 CI 러너, 잠긴 화면은 캡처와 입력을 모두 거부한다 (`docs/guide/computer-use.md:39`).
- `computer.enabled: false`면 툴이 아예 등록되지 않고 `/computer`가 "이 세션에서 사용할 수 없다"고 답한다. `/computer on`/`off`는 **현재 세션에만** 유효하다.

### OS 권한

macOS는 두 개의 개인정보 권한이 필요하고, **`omo`가 아니라 OmO를 띄운 애플리케이션(Terminal, iTerm2, Ghostty, 에디터, OmO 데스크톱 앱)** 에 부여된다 (`docs/guide/computer-use.md:43-48`).

| 권한 | 필요한 것 | 위치 |
|---|---|---|
| Screen Recording (신규 macOS는 Screen & System Audio Recording) | 스크린샷, 디스플레이·창 목록 | Privacy & Security > Screen & System Audio Recording |
| Accessibility | 마우스·키보드 입력, AX 트리, 엘리먼트 액션, **전역 정지 화음** | Privacy & Security > Accessibility |

핵심 함정 두 가지가 문서에 명시돼 있다.

1. **승인은 실행 중인 프로세스에 도달하지 않는다.** Screen Recording 변경 후 macOS가 "Quit & Reopen"을 제안할 때까지, 이미 떠 있던 프로세스에는 승인이 적용되지 않는다. 재시도가 아니라 **앱을 완전히 종료하고 재실행**해야 한다.
2. **OmO를 띄우는 앱을 바꾸면 그 앱에 별도 승인이 필요하다.** 터미널 앱마다 독립적이다.

문서는 상상의 최종 사용자에게 직접 말한다: *"The agent must stop and wait until you confirm the grant and relaunch rather than retrying."*

### 크로스 플랫폼 차이

| OS | 알려진 구멍 (`docs/guide/computer-use.md:155-161` "Known limitations") |
|---|---|
| macOS | 배경 클릭이 클릭된 창을 프론트 바로 아래로 올린다 (프론트앱·포커스·커서·다음 키 대상은 유지). 전체 이전 창 순서 복원은 [#8930](https://github.com/code-yeongyu/oh-my-openagent/issues/8930) 추적 중 |
| Linux | **Wayland: 단일 창 캡처 없음, 창별 입력 없음, `raise()` 없음**, 정지 화음에 GlobalShortcuts 포털 필요. **Linux arm64용 릴리스 엔진 없음** |
| Windows | 비승격 OmO에서 승격 앱으로 입력 불가(**UIPI**). 접근성 클릭 폴백도 우회 못 함. 배경 입력은 액션별 제한(WPF 포인터/텍스트, Chromium posted input). **Windows arm64용 릴리스 엔진 없음** |

X11 백엔드는 RandR 캡처 + XTEST 입력(배경 전달은 XSendEvent) + D-Bus 위 AT-SPI를 쓴다. 일부 툴킷이 합성 배경 입력을 무시하면 OmO는 `BackgroundUnavailable`을 보고한다 — *"instead of pretending the event landed"* (이벤트가 도착했다고 가장하지 않는다).

Wayland는 **포커스된 표면으로만** 입력을 허용하므로 창별 입력과 `raise()`가 원천적으로 불가능하다.

### 상시 주의점

- **스크린샷이 모델 제공자로 나간다.** `docs/guide/computer-use.md:136` — 툴 호출이 돌려준 스크린샷·AX 텍스트·창 제목은 대화의 일부가 되어 다른 툴 결과와 마찬가지로 모델 제공자에게 전송된다. 민감한 창은 닫거나 가리거나 `computer:read`를 거부하라.
- **네트워크는 엔진 다운로드 한 번뿐.** 엔진은 네트워크 요청을 하지 않는다. npm 설치 시 최초 엔진 다운로드만 예외 (`docs/guide/computer-use.md:135`).
- **텔레메트리는 열거형 3종뿐.** `computer_use_activation` / `computer_use_permission_denied` / `computer_use_engine_error`. 스크린샷·화면 텍스트·창 제목·앱 이름·좌표·타이핑 텍스트·툴 인자·경로·에러 메시지는 **절대 전송하지 않는다** (`docs/guide/computer-use.md:137`).
- **엔진 부팅 실패 상태 3종**: `engine: native-unavailable`(이 호스트용 엔진 없음), `quarantined`(macOS 격리 속성), `abi-mismatch`(`computer.engine_path`가 다른 릴리스 엔진). `/computer status`가 보고한다.

---

# 검증 메모

## 읽고 확인한 1차 출처

| 경로 | 확인 내용 |
|---|---|
| `agents/multimodal-looker.ts` | 62줄 전체. `mode`/`temperature`/`createAgentToolAllowlist(["read"])`/`triggers: []`/프롬프트 원문 |
| `tools/look-at/AGENTS.md` | 52줄 전체. 26파일·6단계 플로우·`READ_ENABLED=false`·"summary extractor, not a precise reader" |
| `tools/look-at/look-at-prompt.ts` | 32줄 전체. `READ_ENABLED`·`sanitizeFilename`·`sourceClause` 분기 |
| `tools/look-at/look-at-session-runner.ts` | 125줄 전체. 세션 생성·`tools` 강제 비활성화·모델/variant 주입·`unauthorized` 처리 |
| `tools/look-at/session-poller.ts` | 164줄 전체. 폴링 상수·`TERMINAL_STATUSES`·`IDLE_STABILITY_POLLS_REQUIRED`·타임아웃 예외 |
| `tools/look-at/multimodal-fallback-chain.ts` | 66줄 전체. 두 번 순회 조립 로직·중복 제거 규칙 |
| `tools/look-at/constants.ts` | 전체. `LOOK_AT_DESCRIPTION` 원문 |
| `tools/look-at/multimodal-agent-metadata.ts` | 40줄. 해석 파이프라인 참조·`isVisionCapableAgentModel` |
| `omo-config-core/src/schema/computer.ts` | 60줄 전체. 스키마 12필드 + `COMPUTER_HARNESS_SUPPORT` 12행 |
| `docs/reference/computer.md` | 64줄 전체. 액션 표·권한 분류·에러 코드 15종·엔진 상호운용성 |
| `docs/guide/computer-use.md` | 167줄 전체. 명령 5종·엔진 탐색·OS별 설정·안전 모델 7항목·개인정보·트러블슈팅·Known limitations·엔진 모드 |
| `hooks/runtime-fallback/agent-resolver.ts` | 33줄. `AGENT_NAMES` 13개 + `detectAgentFromSession` |
| `model-core/src/agent-model-requirements.ts:78-85` | `multimodal-looker` 폴백 체인 4rung |
| `plugins/tool-registry-core-tools.ts:38,138-140` | `isMultimodalLookerEnabled` 게이트 |
| `shared/permission-compat.ts:31-42` | `createAgentToolAllowlist` = deny-all + 화이트리스트 |
| `hooks/json-error-recovery/hook.ts:1-25` | `JSON_ERROR_TOOL_EXCLUDE_LIST`에 `look_at` 포함 |
| `tools/delegate-task/constants.ts:395-415` | `COORDINATOR_AGENT_NAMES = ["prometheus"]` |
| `tools/delegate-task/tools.ts:72-73,130-131` | `category`/`subagent_type` 둘 다 자유 문자열 + one-of 검증 |
| `shared/agent-tool-restrictions.ts:53-55` | `"multimodal-looker": { read: true }` |
| `agents/builtin-agents.ts:10,38,55` | import + `agentSources` + 메타데이터 등록 |
| `config/schema/agent-names.ts:10,45` · `agent-overrides.ts:84` | 이름 enum + 오버라이드 스키마 |
| `crates/senpi-desktop-engine/src/cli.rs:15` | `ArgGroup` 모드 플래그 7종 |
| `crates/senpi-desktop-engine/src/mcp/catalog.rs:42` | `desktop.stop` = "only the desktop user resumes" |
| `crates/senpi-desktop-engine/src/{client,serve,daemon,oneshot,selftest,mcp}` | 모듈 doc comment으로 `--serve`/`--resume`/`--oneshot`/`--selftest`/`--mcp` 역할 확인 |

## 확인 필요

| 항목 | 사유 |
|---|---|
| `task(subagent_type="multimodal-looker")`의 실제 런타임 동작 | `tools/delegate-task/subagent-resolver.ts` 전문 미독. 스키마가 enum이 아니고 coordinator 가드가 `prometheus`만 막는다는 구조적 사실까지만 확인 |
| `shared/agent-tool-restrictions.ts`의 `read: true`가 factory의 deny-all을 덮는지 | 이 맵은 `write: false` 같은 값을 담는 레거시 형식이라, `true`가 "허용" 예외인지 "비허용" 표기인지 의미가 모호하다. 실권한의 1차 출처는 `multimodal-looker.ts:15` |
| `tools/look-at/AGENTS.md:47`의 에이전트 경로 | `src/agents/builtin-agents/multimodal-looker.ts`는 **존재하지 않는다**(실제: `agents/multimodal-looker.ts`). 문서 오류로 판단되나 원 작성 의도는 미확인 |
| `look-at-arguments.ts`의 원격 URL 거부 규칙 | `AGENTS.md` 플로우 명세로만 확인. 실제 정규식·허용 프로토콜 목록 미독 |
| `image-converter.ts`의 HEIC/WebP/RAW/PSD 목록 | `AGENTS.md` 파일 카탈로그로만 확인 |
| `computer_actions`의 9개 액션 목록 | `topics/vision-computer-use.md`가 `packages/senpi-desktop-tool/src/cua-actions.ts` L54를 근거로 인용. 이번엔 미검증 |
| `COMPUTER_HARNESS_SUPPORT`의 소비 지점 | 이 맵을 실제로 읽는 코드 위치 미추적. "네이티브 외 타깃에서 무시된다"는 `docs/guide/computer-use.md:7`의 명시 + 맵 값으로만 판단 |
| `--selftest` / `--schema`의 출력 상세 | `cli.rs` 플래그 존재와 `selftest.rs` doc comment만 확인 |

## 교차 링크

- 상위: [oh-my-openagent.md](../oh-my-openagent.md) — "무엇이 있는가" 목록
- 형제: [06-skills-mcp-tools.md](06-skills-mcp-tools.md) — 툴 등록 게이트(12+26)와 MCP 3-tier
- 형제: [02-targets.md](02-targets.md) — `omo-opencode` / `omo-native` / `omo-senpi` / `omo-codex`의 실제 차이
- 형제: [07-config-distribution.md](07-config-distribution.md) — 하니스 블록 해석이 `COMPUTER_HARNESS_SUPPORT`와 어떻게 만나는지
- 형제: [08-upstream-docs-map.md](08-upstream-docs-map.md) — 공식 45개 문서 지도
- 저장소 전체 교차 비교: [topics/vision-computer-use.md](../../topics/vision-computer-use.md)