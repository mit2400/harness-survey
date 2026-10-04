# OmO 설정 해석 · 배포 (config resolution & distribution)

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` · 버전 `5.1.13` (2026-10-04)
> 관련: [harnesses/oh-my-openagent.md](./oh-my-openagent.md) · [topics/config.md](../topics/config.md)
> 코드 기준 경로: `packages/omo-config-core/src/{loader,schema,migration,writer}/`, `packages/omo-opencode/src/cli/`, `script/build-binaries.ts`, `bin/platform.js`

OmO의 설정 시스템은 두 축으로 갈린다.

1. **해석(resolution)** — JSONC 레이어를 찾아 읽고, 하니스 블록과 프로필로 접어, Zod로 검증하며, 최소 단위로 잘라낸다.
2. **배포(distribution)** — npm 래퍼 패키지 하나 + 플랫폼 바이너리 패키지 12개 + GitHub Release 단일 파일 바이너리 + Desktop 앱. 네 갈래 경로가 서로 다른 업데이트 전략을 가진다.

---

## 1. 설정 파일과 우선순위

### 1.1 실제 파일명 — `packages/omo-config-core/src/loader/paths.ts`

| 스코프 | 경로 | 근거 |
|---|---|---|
| user | `$HOME/.omo/omo.jsonc` | `resolveUserOmoConfigPath()` → `join(resolveUserOmoConfigDirectory(), "omo.jsonc")` |
| user (구버전 호환) | `$HOME/.omo/omo.json` | `detectUserOmoJsonPath()`: `omo.jsonc` 없으면 `omo.json`으로 폴백 |
| project | `<dir>/.omo/omo.jsonc` → 없으면 `.omo/omo.json` | `detectOmoJsonPath()` |

**부모 문서(`harnesses/oh-my-openagent.md` L82)의 서술은 정확하지만 불완전하다.** 빠져 있는 것들:

| 동작 | 세부 | 코드 |
|---|---|---|
| 깊이 상한 | walk-up 최대 **256단계** | `MAX_PROJECT_CONFIG_DIRECTORY_DEPTH = 256` |
| 홈 경계 | `cwd`에서 올라가다가 `$HOME`(=계정 홈) 만나면 **멈춤**. 홈의 `.omo`는 user 레이어로만 취급되므로 project로 중복 청구하지 않음 | `findProjectConfigPathsFarthestFirst()`의 `isHomeDir` 분기 |
| realpath 비교 | `process.cwd()`, `HOME`, 계정 홈이 심볼릭 링크로 어긋날 수 있어(`/var` vs `/private/var`) 경계 비교는 realpath로, 반환 후보는 호출자 형태 유지 | `realBoundaryDirs` |
| 심볼릭 링크 스킵 | `.omo` 디렉토리 자체가 심볼릭 링크면 project 후보 아님. `omo.jsonc`/`omo.json` 파일이 심볼릭 링크면 로드 불가 | `isSymlinkedProjectPath()`, `isLoadableProjectConfigFile()` |
| 홈 결정 | `env.HOME ?? env.USERPROFILE ?? process.cwd()` — Windows 대응 | `resolveHomeDir()` |

### 1.2 레이어 순서 — farthest-first

`resolveOmoConfigPaths()`가 반환하는 배열은 **user 1개 → project 여러 개(farthest-first)**:

```
[ { user, $HOME/.omo/omo.jsonc }, { project, <최상위 조상>/.omo/omo.jsonc }, ... { project, <cwd>/.omo/omo.jsonc } ]
```

`loader.ts`는 이 배열을 정방향으로 순회하며 `mergeOmoConfigRecords()`로 누적한다. **뒤에 오는 project가 이기므로 nearest wins**가 Emergent하게 성립한다 — "nearest 우선" 규칙이 코드에 명시적으로 적혀 있지 않은 점이 중요하다. 상위 문서의 "cwd→$HOME walk-up, nearest wins"는 결과를 기술한 것이지 구현된 규칙이 아니다.

---

## 2. 하니스별 해석 — `src/loader/resolution.ts`

### 2.1 실제 코드 경로

`resolveOmoConfigView({ config, harness, profile })`는 **4개 레이어를 순서대로 deep-merge**한다:

```ts
const layers = [
  withoutControlKeys(options.config),      // ① shared base (profiles·[harness] 키 제거)
  harnessLayer(options.config, harness),   // ② 하니스 블록
  profile === undefined ? {} : withoutControlKeys(profile),        // ③ 프로필 shared
  profile === undefined ? {} : harnessLayer(profile, harness),    // ④ 프로필 안의 하니스 블록
]
let config = {}
for (const layer of layers) config = mergeOmoConfigRecords(config, layer)
return { config: withoutControlKeys(config), diagnostics, profile: resolvedProfile }
```

`withoutControlKeys()`는 `profiles` 와 `HARNESS_KEYS`(`HARNESS_IDS` + `OMO_CONFIG_HARNESS_IDS` + `OMO_CONFIG_LEGACY_HARNESS_IDS`의 블록명 `[opencode]`/`[native]`/`[codex]`/`[senpi]`)를 제거한다. 마지막에 한 번 더 호출되어 제어 키가 최종 결과에 남지 않는다.

**레거 시 퀄리티**가 ①shared → ②harness → ③profile-shared → ④profile-harness 순이다. 즉 프로필이 하니스 블록을 이기고, 프로필 안에서는 하니스 블록이 shared를 이긴다.

### 2.2 하니스 ID와 레거시 별칭

| 상수 | 값 | 출처 |
|---|---|---|
| `OMO_CONFIG_HARNESS_IDS` | `["opencode", "native", "codex"]` | `schema/harness.ts` |
| `HARNESS_IDS` (엔진 전역) | `["codex", "opencode", "omo"]` | `schema/harness.ts` |
| `OMO_CONFIG_LEGACY_HARNESS_ALIASES` | `{ senpi: "native" }` | `schema/harness.ts` |

`harnessLayer()`는 `canonicalHarnessName(harness)`를 구한 뒤, **레거시 별칭 블록을 먼저, canonical 블록을 나중에** 머지한다:

```ts
for (const key of [...legacyKeys, harnessBlockKey(canonical)]) {
  layer = mergeOmoConfigRecords(layer, toRecord(config[key]) ?? {})
}
```

따라서 `[senpi]`가 먼저 깔리고 `[native]`가 그 위를 덮는다. `[senpi]`만 쓰는 기존 설정은 온전히 적용되면서, 양쪽 다 쓰는 설정에서는 `[native]`가 승리한다. 파일 자체는 시작 시 마이그레이션이 `[native]`로 다시 써준다.

### 2.3 단일 타깃 전용 블록 worked example

`~/.omo/omo.jsonc`:

```jsonc
{
  // shared — 모든 하니스가 본다
  "model_profile": "balanced",
  "telemetry": { "enabled": false },

  // opencode 전용: OpenCode 플러그인 설정 원문 통째로 넘긴다
  "[opencode]": {
    "plugin": ["oh-my-opencode@5.1.13"],
    "permission": { "edit": "ask" }
  },

  // codex 전용: strict 스키마. 오타 하나면 해당 값이 잘려나간다
  "[codex]": {
    "agents": { "Sisyphus-Junior": { "model": "gpt-5-codex-low" } }
  }
}
```

`omo run`(=opencode 어댑터) vs `omo-codex` 실행 결과:

| 키 | opencode | codex | native(senpi) |
|---|---|---|---|
| `model_profile` | `balanced` | `balanced` | `balanced` |
| `telemetry.enabled` | `false` | `false` | `false` |
| `plugin`, `permission` | **적용** | 없음 | 없음 |
| `agents.Sisyphus-Junior.model` | 없음 | **적용** | 없음 |

`[opencode]`가 typed가 아니라 `z.record(z.string(), z.unknown())`(`OmoOpenCodeHarnessConfigSchema`)라는 점이 핵심이다. opencode 네이티브 설정을 손대지 않고 통과시키기 위한 설계이며, 그 결과 **opencode 블록에는 OmO의 검증·진단·prune가 전혀 걸리지 않는다**. 실수는 `omo doctor`가 아니라 opencode가 잡는다.

### 2.4 프로필 해석 순서 — 전부 검증됨

`src/loader/resolution.ts` `resolveOmoProfileName()`:

```ts
return profileName(options.profile)          // ① 명시적 인자 (highest)
  ?? profileName(env["OMO_PROFILE"])          // ② OMO_PROFILE
  ?? profileName(env["OCX_PROFILE"])          // ③ OCX_PROFILE
  ?? profileNameFromOpenCodeConfigDir(env["OPENCODE_CONFIG_DIR"])  // ④ OPENCODE_CONFIG_DIR tail
```

세 환경변수 **모두 코드에 존재한다**(상위 문서의 주장 그대로). 세 가지 디테일:

- `profileName()`은 `value === ""`를 `undefined`로 바꾼다. **빈 문자열은 "미설정"으로 취급** — `OMO_PROFILE=` 로 exporting하면 프로필이 꺼진다.
- ④는 `path?.match(/(?:^|[\\/])profiles[\\/]([^\\/]+)[\\/]*$/)` 로 끝부분을 파싱한다. 즉 `~/.config/opencode/profiles/work` 꼴이어야 하며, 경로 중간에 `profiles/`가 있어도 마지막 세그먼트가 이름이 된다.
- 프로필이 없으면 diagnostic을 낸다: `{ kind: "profile", message: 'Activated omo profile "X" does not exist; using the base configuration', path: "profiles.X" }`. **throw가 아니라 base 설정으로 폴백**한다.

---

## 3. 스키마 — `packages/omo-config-core/src/schema/`

### 3.1 Zod v4 + strict object

`schema/config.ts`의 3개 루트 스키마:

| 스키마 | 용도 | 차이 |
|---|---|---|
| `OmoConfigSchema` | 최종 문서 검증 | 모든 섹션 full 스키마, `profiles`에 `.default({})` |
| `OmoConfigLayerSchema` | **레이어 단위 사전 검증** (머지 전) | 모든 섹션 `*LayerSchema`(부분 스키마), 전부 optional |
| `OmoTypedHarnessConfigSchema` | `[native]`/`[senpi]`/`[codex]` 블록 | 14개 섹션, `.strict()` |
| `OmoConfigProfileSchema` | `profiles.<P>` | 12개 섹션 + 4개 하니스 블록, `.strict()` |

전부 `.strict()`다. **모르는 top-level 키는 조용히 무시되지 않고 `unrecognized_keys` 이슈가 되어 해당 키가 삭제되고 진단에 오른다.**

### 3.2 top-level 키 22개 (스키마에서 직접 추출)

| # | 키 | 도메인 |
|---|---|---|
| 1 | `formatOnMutation` | 포맷 정책 (camelCase 유 outlier) |
| 2 | `gateway` | 게이트웨이/라우팅 |
| 3 | `$schema` | JSON Schema URL |
| 4 | `categories` | 딜리게이션 9카테고리 |
| 5 | `agents` | 11개 내장 에이전트 오버라이드 |
| 6 | `git_master` | git-master 스킬 설정 |
| 7 | `task` | task/ulw 실행 |
| 8 | `teams` | Team Mode |
| 9 | `models` | 모델 카탈로그/앰비언트 |
| 10 | `model_profiles` | 모델 프로필 정의 |
| 11 | `model_profile` | 활성 모델 프로필 이름 (문자열) |
| 12 | `memory` | memory-core / Kibitzer |
| 13 | `telemetry` | 텔레메트리 |
| 14 | `computer` | computer-use (senpi desktop) |
| 15 | `disabled_skills` | 스킬 denylist |
| 16 | `[opencode]` | **자유 형식 패스스루** (`z.record(z.string(), z.unknown())`) |
| 17 | `[native]` | typed 하니스 블록 |
| 18 | `[senpi]` | 레거시 별칭(`→ [native]`) |
| 19 | `[codex]` | typed 하니스 블록 |
| 20 | `profiles` | 프로필 맵 (`record(string, OmoConfigProfileSchema)`) |
| 21 | `_migrations` | 적용 완료 마이그레이션 ID 배열 |
| 22 | `legacy_migrations` | 레거시 상태 기록 |

**도메인 섹션 13개**: `categories, agents, task, teams, models, model_profiles, git_master, memory, telemetry, computer, gateway, formatOnMutation, disabled_skills`

### 3.3 문서 ↔ 스키마 불일치 3건

| 항목 | 문서/상위 정리 | 스키마 실측 |
|---|---|---|
| 키 개수 | `topics/config.md` L31에 19개 열거 | **22개**. `formatOnMutation`, `gateway`, `computer`가 빠져 있음 |
| 네이밍 | "snake_case" | **거의 snake_case지만 `formatOnMutation`만 camelCase**. 전 구간 snake_case 규칙이 아니다 |
| 레거시 블록 | `topics/config.md`에 `[senpi]` 없음 | 스키마·해석 코드 모두 `[senpi]`를 1급 키로 유지 |

또한 `docs/reference/configuration.md`(1,344줄, 레포 최대 공식 문서)는 `##` 섹션이 `Table of Contents / Getting Started / Core Concepts / Task System / Features / Advanced / Reference` 7개뿐이라, **22개 키를 평탄한 레퍼런스 목록으로 제공하지 않는다.** 정합성 기준은 스키마가 유일한 소스다.

`profiles.<P>`는 `$schema`, `gateway`, `_migrations`, `legacy_migrations`를 **설정할 수 없다** — 프로필이 `_migrations`를 지정하면 strict 검증에서 그 키가 잘린다.

### 3.4 스키마 파일 지도

```
schema/
├── config.ts        # 3개 루트 스키마 (위 표)
├── harness.ts       # HARNESS_IDS, OMO_CONFIG_HARNESS_IDS, LEGACY_ALIASES
├── agent.ts  category.ts  task.ts  team.ts
├── memory.ts  git-master.ts  telemetry.ts  computer.ts  gateway.ts
├── model-catalog.ts  model-profile.ts  model-ref.ts  fallback-models.ts
├── format-on-mutation.ts  reasoning-vocabulary.ts
├── legacy-category-names.ts  legacy-harness-names.ts   # 런타임 별칭 정규화
└── index.ts
```

`legacy-*.ts`는 스키마가 아니라 **읽기 시점 이름 정규화**다. `canonicalizeLegacyHarnessBlocks()` / `canonicalizeLegacyCategoryNames()`가 `loader.ts`에서 호출되며, 정규화 사실 자체가 `kind: "deprecated-keys"` 진단으로 보고된다(같은 키가 canonical과 레거시 양쪽에 있으면 레거시를 **버린다** — `dropped`).

---

## 4. 머지 의미론 — `src/loader/merge.ts`

`mergeOmoConfigRecords(base, override)` 전체가 이거다:

```ts
const DANGEROUS_KEYS = new Set(["__proto__", "constructor", "prototype"])

export function mergeOmoConfigRecords(base, override): Record<string, unknown> {
  const result = { ...base }
  for (const [key, value] of Object.entries(override)) {
    if (isUnsafeObjectKey(key)) continue                              // ① 위험 키 스킵
    const safeValue = sanitizeOmoConfigValue(value)                   // ② 재귀 sanitize
    const baseValue = result[key]
    result[key] = isPlainObject(baseValue) && isPlainObject(safeValue)
      ? mergeOmoConfigRecords(baseValue, safeValue)                   // ③ 둘 다 plain object → 재귀 병합
      : safeValue                                                      // ④ 그 외 → 통째 교체
  }
  return result
}
```

정밀 서술:

| 규칙 | 구현 |
|---|---|
| **deep-merge 대상** | `isPlainObject` = `typeof === "object"` && `!== null` && `!Array.isArray` && `Object.prototype.toString.call(v) === "[object Object]"`. 클래스 인스턴스는 plain이 아니므로 **교체**된다 |
| **scalar 교체** | 위 ④번 분기. 숫자·문자열·불리언은 병합되지 않는다 |
| **array 교체** | `sanitizeOmoConfigValue()`가 배열을 `map`으로 새 배열 만들 뿐 **원소별 병합은 없다**. `teams.members` 같은 배열은 상위 레이어가 통째로 덮어쓴다 |
| **prototype pollution 방어** | 2중. (a) `merge.ts`가 `__proto__`/`constructor`/`prototype`을 override 키와 **모든 중첩 깊이**에서 제거. (b) `layer-validation.ts` `sanitizeUnsafeKeys()`가 **Zod에 넘기기 전** 같은 키를 제거하고 `unrecognized_keys` 이슈로 승격 → `stripUnrecognizedKeys()`가 삭제하고 진단에 기록. 프로토타입이 `Object.prototype`도 `null`도 아닌 객체 자체를 하나의 `__proto__` 위반으로 보고한다 |
| **순수성** | 입력 객체를 mutate하지 않는다. `{...base}` 복사 + 새 객체 생성 |

**예외 하나**: `disabled_skills`는 레이어 간 **union(합집합)**이며 교체되지 않는다(`loader/disabled-skills.ts`). 스키마 주석이 명시한다: "layers (user, project, `[harness]`, profile) are unioned, never replaced". 즉 위 표의 "array 교체"는 `disabled_skills`를 제외한 일반 규칙이다.

---

## 5. 견고성 — 잘못된 값이 파일 전체를 죽이지 않는다

### 5.1 파이프라인 (`loader.ts` → `layer-validation.ts` → `prune-invalid-leaves.ts`)

```
readFileSync 실패 ──────────────────→ diagnostic { kind: "read"  }, 레이어 미로드
JSONC 파싱 오류 ────────────────────→ diagnostic { kind: "parse" }, 레이어 미로드
sanitizeUnsafeKeys → 위험 키 제거    → stripped 복제본 + unrecognized_keys 이슈
OmoConfigLayerSchema(strict) 검증   → issues[]
pruneInvalidConfigPaths() 반복      → 최소 subtree만 삭제, 재검증 (최대 32패스)
```

JSONC 파서 옵션: `jsonc-parser`의 `parse(content, errors, { allowTrailingComma: true, disallowComments: false })`. BOM(`0xfeff`)은 먼저 벗겨낸다. **JSONC 문법 오류는 그 레이어를 통째로 버린다** — 이 한 경로만 "부분 적용"되지 않는다.

### 5.2 최소 subtree prune — `prune-invalid-leaves.ts`

파일 헤더 주석이 규칙 전체를 서술한다:

> "One bad value must not take down the file that carries it: every validation issue names a path, and the loader drops only the smallest subtree that fails (the value at that path, or the object that lacks it), keeps every healthy sibling at every depth, and re-validates after each pass until the document parses."

`targetPaths()`가 이슈 유형별로 제거 대상을 정한다:

| 이슈 유형 | 제거 대상 |
|---|---|
| 값이 틀림(wrong value) | 그 경로의 값 자체. 예: `task.host_engine_policy` |
| required 키 누락 | 그 키가 없는 **객체 전체**. 예: `teams.alpha` |
| `unrecognized_keys` | 이슈가 지목한 키 각각을 경로 뒤에 붙여 제거 |
| 경로가 빈 배열 | 문서 루트 자체 → **fail-closed** |

부가 규칙:

- `MAX_PRUNE_PASSES = 32` — 이 한계를 넘기면 레이어 전체 거부.
- `removePathAndEmptiedAncestors()` — 자식이 다 빠져 빈 컨테이너가 되면 컨테이너도 함께 죽는다. 유효한 것이 하나도 없는 문서는 빈 문서로 온다.
- `compareDescending()` — 배열 인덱스를 **내림차순**으로 지워, splice로 아직 제거 대기 중인 경로가 밀리지 않게 한다. `teams.alpha.members.0.color` 같은 중첩 배열 경로를 안전하게 지운다.
- fail-closed 원칙(헤더 주석): "old totality preserved as the last resort, never the first". 패스 한계 도달이나 루트 이슈면 이전 완전한 값을 유지한다.

### 5.3 진단 종류

`OmoConfigDiagnosticKind`에서 관측되는 값: `read`, `parse`, `validation`, `invalid-value`, `deprecated-keys`, `profile`.

메시지 포맷:

| kind | 형식 | 출처 |
|---|---|---|
| `parse` | `JSONC parse error in <path>: <codes>` | `loader.ts` `readConfigSource()` |
| `read` | `Failed to read <path>: <message>` | `loader.ts` |
| `validation` | `Invalid omo config at <path>: <issuePaths.join(", ")>` | `validationDiagnostic()` |
| `invalid-value` | `Ignored invalid value in <path>: <dotted key>: <message>` | `invalidValueDiagnostics()` |
| `deprecated-keys` | `Deprecated harness block in <path>: [senpi] renamed to [native] / ignored because [native] is also configured` | `loader.ts` |
| `profile` | `Activated omo profile "X" does not exist; using the base configuration` | `resolution.ts` |

머지 이후 레이어에 대한 진단은 합성 경로 `MERGED_OMO_CONFIG_PATH = "(merged omo config)"`(`loader/types.ts`)를 쓴다.

### 5.4 `omo doctor`

`cli-program.ts` L243-247에 정의: `doctor` + `--status`(compact 대시보드) / `--verbose` / `--json`.

`packages/omo-opencode/src/cli/doctor/checks/` — 설정 관련 체커:

| 체커 | 역할 |
|---|---|
| `config.ts` | 설정 로드/스키마 상태 |
| `legacy-config-leftovers.ts` | 옛 설정 파일 잔여물 |
| `deprecated-reasoning-keys.ts` | 폐기된 reasoning 키 |
| `tui-plugin-config.ts` / `-unresolvable.ts` | TUI 플러그인 설정 해석 |
| `model-resolution*.ts` (7개) | 모델 해석/캐시/effective model |
| `tuple-plugin-entries.ts` | 플러그인 엔트리 포맷 |
| `system.ts` / `system-binary.ts` / `system-plugin.ts` / `system-loaded-version.ts` | 런타임 바이너리·플러그인·버전 |
| `latest-version.ts` | 배포 채널 최신 버전 |
| `codex*.ts`, `dependencies.ts`, `telemetry.ts`, `team-mode.ts`, `tools*.ts`, `browser-provider.ts` | 그 외 설치 상태 |

진단 경로는 **CLI doctor가 소비**하는 경로이고, 런타임은 `loadOmoConfig()`가 돌려주는 `diagnostics` 배열을 그대로 전달받는다. `omo doctor`는 이것을 사람이 읽을 형태로 렌더링하는 또 하나의 소비자다.

---

## 6. 마이그레이션 — lock + journal

### 6.1 명령 표면 (검증됨)

`cli-program.ts` L270-275:

```
omo config migrate [--dry-run] [--json]
  migrate  "Migrate legacy OMO configuration into ~/.omo/omo.jsonc"
  --dry-run "Print the transform, backup move plan, and conflicts without new migration writes"
  --json   "Print machine-readable migration output"
```

구현: `cli/config-migrate.ts` `runConfigMigrate()` → `runOpenCodeStartupMigration()`(`packages/omo-opencode/src/startup-migration`). 즉 **startup 마이그레이션과 CLI 마이그레이션이 같은 엔진**을 공유한다.

텍스트 출력(`printText`)은 migration별로 `status:` / `target:` / `transform:`(JSON, 들여쓰기 2) / `move-plan:` / `conflict:`(diagnostic) 줄을 낸다. 전부 `skipped`면 `Nothing to migrate.`

`--json` 스키마: `{ dryRun, error, journalResumed, migratedFrom, migrations[], skippedConflictCount }`. 복구되면 `Recovered pending migration journal.` 출력. **종료 코드**: 성공 0, `result.error` 있으면 1.

### 6.2 lock — `src/migration/lock.ts`

| 항목 | 값 |
|---|---|
| 경로 | `~/.omo/.migration.lock` (`migrationLockPath()`) |
| 레코드 | `{ leaseExpiresAt: number, pid: number }` JSON |
| 기본 lease | `DEFAULT_LEASE_DURATION_MS = 30_000` |
| guard lease | `GUARD_LEASE_DURATION_MS = 1_000` |
| 재생성 임계 | `LIVE_OWNER_STALE_LEASE_MULTIPLIER = 2` — 살아있는 소유자가 만료 후 **2회 갱신 윈도우를 더** 놓쳐야 탈취 가능. 죽은 소유자는 만료 즉시 회수 가능 |
| mutation guard 재시도 | `MUTATION_GUARD_RETRY_DELAYS_MS = [2, 4, 8, 16, 32]` ms (지수 백오프) |
| 대기 | `SharedArrayBuffer` + `Atomics.wait` — 이벤트 루프를 블로킹하지 않는 스핀 (`MUTATION_GUARD_SLEEP_VIEW`) |
| 보조 파일 | `~/.omo/.migration.lock.guard` |
| 생성 원자성 | `EEXIST` 처리 (`isFileExistsError`) — `wx` 플래그 계열 |

**lease + pid 이중 판정**이라, 죽은 프로세스의 락은 즉시 회수되고 살아있는 프로세스의 락은 조용히 뺏기지 않는다.

### 6.3 journal — `src/migration/journal.ts`

| 항목 | 값 |
|---|---|
| 경로 | `~/.omo/.migration-journal.json` |
| 버전 | `version: 1` (다른 값은 throw) |
| 레코드 필드 | `{ version, migrationId, targetPath, targetWrite: { additions, mode?: "replace-target" }, targetWritten, completedMoves[], backupMoves[], diagnostics[] }` |
| 원자적 생성 | `EEXIST` 감지 (`isFileExistsError`) |
| 임시 파일 | `<path>.<pid>.<now>.tmp`, 재시도 시 `.tmp.<attempt>.tmp` |
| 검증 | `parseJournal()`이 plain object·version·targetPath/migrationId 문자열·`targetWrite.additions` plain object·`mode` 화이트리스트·`targetWritten` boolean·`completedMoves` 배열 전부 강제 |

모듈 분해: `engine.ts`(오케스트레이션), `batch.ts`(일괄 실행 + dry-run 분기), `commit.ts`(타깃 쓰기), `merge.ts`(기존 대상 병합), `backup-move.ts`(백업 이동), `recovery.ts`(저널 복구), `predicate.ts`/`should-run`, `types.ts`.

**dry-run 분기**(`batch.ts`): `if (input.dryRun) return { diagnostics, journalResumed, preview, status: "planned" }` — 쓰기 없음. 추가로 `if (options.dryRun !== true) options.afterMigrations?.(results)` 로 후크까지 건너뛴다. 즉 dry-run은 **저널·락·백업 이동·커밋 어디에도 흔적을 남기지 않는다.**

### 6.4 백업

`migration/backup-move.ts`가 `backupMoves[{from, to}]`를 실행하며, `--dry-run` 출력의 `move-plan:`에 그대로 나타난다. 백업 위치 규칙(`~/.omo/migration-backup-<ts>/`)은 `topics/config.md` L32에 기재되어 있으나 **이 핀의 `migration/` 코드에서 `<ts>` 접미 규칙은 확인 필요**.

---

## 7. 상태 디렉토리

### 7.1 검증된 것

| 경로 | 내용 | 근거 |
|---|---|---|
| `~/.omo/` | user 설정 루트 | `paths.ts` |
| `~/.omo/.migration.lock`, `.migration.lock.guard` | 마이그레이션 락 | `lock.ts` |
| `~/.omo/.migration-journal.json` | 마이그레이션 저널 | `journal.ts` |
| `~/.omo/binary-runtime/<version>/` | 컴파일 바이너리의 자기 프로비저닝 런타임(엔진·플러그인·PTY prebuild·테마·에셋) | `docs/guide/binary-install.md` |
| `~/.omo/install.json` | 설치 영수증 | `docs/guide/install.md`/`binary-install.md` |
| `<project>/.omo/` | project 설정 + 상태 | `paths.ts` |
| `<project>/.omo/plans/` | ulw-execute 플랜 | `hooks/ulw-execute/` |
| `<project>/.omo/goal/` + `.omo/ulw-loop/<sessionID>/goals.json` | Goal 컨트롤러 | `hooks/goal/controller.ts` |
| `<project>/.omo/notepads/<planName>/` | ulw 메모 캡처 | `hooks/ulw-execute/notepad-scaffold.ts` |
| `<project>/.omo/teams/<teamName>/` | Team Mode 레지스트리 | `packages/team-core/src/team-registry/paths.ts` |
| `<project>/.omo/rules/*.md` | rules-engine 규칙 파일 | `packages/omo-codex/plugin/components/rules/` fixture |
| `<state>/teams/runtime` | senpi-task 팀 런타임 | `senpi-task` QA 스크립트 |

### 7.2 확인 필요

`~/.omo/agent`(canonical), `~/.omo/teams/`, `~/.omo/rules/`, `~/.omo/plans/`, `~/.omo/goal/`, `~/.omo/lsp-daemon/` 의 **user 레벨 변형**은 이번 스캔에서 코드 리터럴을 확보하지 못했다. `team-core/src/team-registry/paths.ts`는 `<projectRoot>/.omo/teams/<name>`(project 기본)과 `baseDir` 변형 두 갈래만 보이고, `lsp-daemon`은 `packages/senpi-desktop`/`omo-senpi` **플러그인 번들 내부 경로**(`runtime/lsp-daemon/dist/*`)로만 확인된다. `topics/config.md` L33의 나열을 그대로 신뢰하려면 별도 확인이 필요하다.

코드에서 확인되는 **레거시 상태 디렉토리**: `.sisyphus/plans/` (테스트 픽스처에 잔존). 마이그레이션 대상일 가능성이 높다.

---

## 8. 배포 · 릴리스 엔지니어링

이 절이 이 하니스의 차별점이다. **동일한 제품이 네 개의 독립적인 배포 경로를 갖는다.**

### 8.1 네 갈래 경로

| 경로 | 단위 | 업데이트 방식 | 근거 |
|---|---|---|---|
| **npm 래퍼 + 플랫폼 패키지** | `oh-my-opencode@5.1.13` + `oh-my-opencode-<platform>@5.1.13` ×12 | `npm i -g`, postinstall이 플랫폼 패키지 해석 | `package.json` |
| **GitHub Release 단일 파일** | `omo-<platform>` ×12 + `SHA256SUMS` | `get.omo.dev/install.sh` 재실행 또는 `omo update` | `docs/guide/binary-install.md` |
| **npm `omo-ai`** | OmO Native 정품 | version 파생 채널(stable→`latest`, prerelease→`beta`), Trusted Publisher merge gate | `docs/reference/omo-ai-publishing.md` |
| **OmO Desktop** | 데스크톱 앱 | 앱 내 업데이트 UI(Stable / Nightly) | `docs/guide/desktop-updates.md` |

### 8.2 npm 래퍼 구조

루트 패키지 `oh-my-opencode@5.1.13`은 **바이너리가 아니라 래퍼**다.

- **bin alias 5개가 전부 같은 파일로**: `oh-my-opencode`, `oh-my-openagent`, `omo-agent-toolkit`, `lazycodex`, `lazycodex-ai` → 모두 `bin/oh-my-opencode.js`
- **optionalDependencies 12개, 전부 exact pin**: `"oh-my-opencode-darwin-arm64": "5.1.13"` — `^`나 `~`가 없다. 래퍼와 플랫폼 패키지의 버전 드리프트를 원천 차단
- `postinstall: node postinstall.mjs`, `prepare: bun run build`, `prepack: bun run build:materialize-frontend`

### 8.3 플랫폼 패키지 12개

`package.json` optionalDependencies 순서 그대로:

| # | 패키지 | 대상 |
|---|---|---|
| 1 | `oh-my-opencode-darwin-arm64` | macOS Apple Silicon |
| 2 | `oh-my-opencode-darwin-x64` | macOS Intel |
| 3 | `oh-my-opencode-darwin-x64-baseline` | macOS Intel, AVX2 없음 |
| 4 | `oh-my-opencode-linux-arm64` | Linux ARM64 (glibc) |
| 5 | `oh-my-opencode-linux-arm64-musl` | Linux ARM64 (musl/Alpine) |
| 6 | `oh-my-opencode-linux-x64` | Linux x64 (glibc) |
| 7 | `oh-my-opencode-linux-x64-baseline` | Linux x64 (glibc), AVX2 없음 |
| 8 | `oh-my-opencode-linux-x64-musl` | Linux x64 (musl) |
| 9 | `oh-my-opencode-linux-x64-musl-baseline` | Linux x64 (musl), AVX2 없음 |
| 10 | `oh-my-opencode-windows-arm64` | Windows ARM64 — **x64 에뮬레이션** |
| 11 | `oh-my-opencode-windows-x64` | Windows x64 |
| 12 | `oh-my-opencode-windows-x64-baseline` | Windows x64, AVX2 없음 |

**축 구조**: OS 3(darwin/linux/windows) × 아키텍처(arm64/x64) × 변형(기본/baseline/musl). **baseline은 x64에만 존재** — `bin/platform.js` `getBaselinePlatformPackage()`가 `arch !== "x64"`면 `null`을 반환한다. ARM64에 baseline을 낼 이유가 없다는 판단.

### 8.4 해석 로직 — `bin/platform.js`

```js
// Linux: libc 미감지 시 throw ("Could not detect libc on Linux. Please ensure detect-libc is installed")
// musl이면 "-musl" 접미. win32 → "windows"로 치환
return `${packageBaseName}-${os}-${arch}${suffix}`
```

`getPlatformPackageCandidates()`의 폴백 체인:

| 조건 | 후보 순서 |
|---|---|
| `win32` + `arm64` | `[oh-my-opencode-windows-arm64, oh-my-opencode-windows-x64-baseline]` — 네이티브가 없으므로 x64 baseline으로 |
| `preferBaseline = true` | `[baseline, primary]` |
| 그 외 | `[primary, baseline]` — 네이티브 우선, baseline은 안전망 |

**래퍼 이름이 패키지 계열을 바꾼다** (이건 쉽게 놓친다):

```js
const PLATFORM_PACKAGE_BASE_BY_WRAPPER_NAME = {
  lazycodex: "oh-my-openagent",
  "lazycodex-ai": "oh-my-openagent",
}
```

`lazycodex` / `lazycodex-ai`로 설치하면 **플랫폼 패키지 npm 계열이 `oh-my-opencode-*`가 아니라 `oh-my-openagent-*`로 바뀐다.** 같은 `bin` 파일이 두 개 다른 npm 패밀리를 가리킨다. 릴리스 문서의 "probes **both** platform package families"가 정확히 이 때문이다.

### 8.5 빌드 — `script/build-binaries.ts`

`PLATFORMS: PlatformTarget[]` 12 엔트리. 각 엔트리의 `target`은 **Bun의 크로스 컴파일 타깃**이다: `bun-darwin-arm64`, `bun-darwin-x64-baseline`, `bun-linux-x64-musl-baseline`, `bun-windows-x64-baseline` 등. `binary: "oh-my-opencode.js"`, CLI 엔트리 `dist/cli/index.js`.

각 패키지에는 `createPlatformLauncherSource()`가 생성하는 **런처 스텁**이 들어간다. 이 스텁의 특징:

- `#!/usr/bin/env node` + `spawnSync` — 순수 Node
- `process.env.OMO_WRAPPER_PACKAGE_ROOT`가 **없으면 즉시 `exit(2)`**: `oh-my-opencode: OMO_WRAPPER_PACKAGE_ROOT is required to launch the packaged CLI.`
- 시그널 매핑: `SIGINT: 2, SIGILL: 4, SIGKILL: 9, SIGTERM: 15` → `128 + signal`
- `OMO_INVOCATION_NAME`이 `lazycodex`/`lazycodex-ai`면 설치 계열 명령(`install|setup|update|uninstall|cleanup`)을 `packages/omo-codex/scripts/install-local.mjs`로 라우팅

즉 **플랫폼 패키지 = Bun 단일 파일 실행본 + Node 런처 스텁**, 그리고 래퍼 패키지가 디스크에 같이 있어야 실제로 뜬다. 순수 바이너리가 아니다.

`windows-arm64` 엔트리의 `target`이 `bun-windows-x64`, description이 `Windows ARM64 (x64 emulation / node fallback)`인 것이 npm 측의 한계를 코드에 그대로 적어 둔 곳이다. 반면 **GitHub Release 쪽 `omo-windows-arm64.exe`는 네이티브 ARM64**다 — `binary-install.md`가 이 차이를 명시적으로 경고한다.

### 8.6 루트 scripts (배포 파이프라인)

| script | 실제 명령 |
|---|---|
| `build` | `bun run script/build.ts` |
| `build:binaries` | `bun run script/build-binaries.ts` |
| `build:all` | `build` + `build:binaries` |
| `prepublishOnly` | `clean` → `build:lsp-tools-mcp` → `build:lsp-daemon` → `build` |
| `prepare` | `build` |
| `postinstall` | `node postinstall.mjs` |
| `prepack` | `build:materialize-frontend` |
| `build:omo-native` | `script/build-omo-native.ts` |
| `build:codex-install` / `install:codex-dev` | `script/build-codex-install.ts` + `script/install-codex-dev.ts` |
| `build:senpi-plugin:native` | lsp-daemon·ast-grep-mcp runtime 스테이징 → extension 빌드 → 데몬 launch spec → 스킬 sync → **embed-directive `--check`** → install 번들 |
| `build:schema` / `build:omo-schema` / `build:model-capabilities` | 코드 생성 |
| `build:lsp-daemon` | `npm --prefix packages/lsp-daemon ci && npm --prefix packages/lsp-daemon run build` — **Rust crate이 별도 npm island** |
| `typecheck` | `tsgo --noEmit` + `typecheck:script` + `typecheck:packages`(**30개 tsconfig**을 한 줄에 나열) |
| `test:desktop` | `script/check-cargo-pinned-deps.mjs` 선행 + senpi-desktop 5개 패키지 테스트 |

`check-third-party-notices.mjs --ship`이 `test:codex` 안에 박혀 있는 것으로 보아, 3rd-party 고지 검증이 테스트 스위프의 일부다.

### 8.7 릴리스 워크플로 — `docs/reference/release-process.md`

**2단 디스패치 설계**가 이 문서의 핵심이다.

```
1차 dispatch (prepared_release_sha = "")
  ├─ 게이트 통과
  ├─ prepare-release-state: release-state 브랜치에 패키지 메타데이터 스탬프 → PR 오픈/머지
  ├─ refs/tags/v<version> 생성 또는 검증
  └─ 그 태그에서 2차 dispatch
2차 dispatch (prepared_release_sha = <준비된 SHA>)
  ├─ publish-platform  → publish-platform.yml 위임
  ├─ publish-main      → oh-my-openagent, lazycodex-ai, omo-ai
  └─ release           → GitHub release, LazyCodex marketplace sync
```

**`publish-platform`, `publish-main`, `release`는 2차만 실행 가능하다.** 1차를 재실행하면 디스패처만 다시 돌고 실패한 퍼블리싱 잡은 그대로다.

멱등성 가드(문서에 열거됨):

| 잡 | 가드 |
|---|---|
| `prepare-release-state` | 기존 `refs/tags/v${VERSION}`의 SHA를 재사용하거나 기존 `release: v<version>` 커밋을 재사용 |
| `dispatch-provenance-safe-publish` | 기존 태그가 **준비된 릴리스 SHA로 해석될 때만** 재사용. 다른 SHA면 **fail closed** |
| `publish-platform` | 두 플랫폼 패키지 계열 모두 조회, 이미 있는 버전 skip |
| `publish-main` | npm probe 결과로 각 publish step 게이트 |
| `release-metadata` | `already_published` 노출, omo-ai 빌드·게이트 |
| `release` | marketplace payload를 `git diff --cached --quiet`로 검사, 무변경이면 commit/push 안 함 |

**릴리스 바이너리 자산 레인**: 빌드는 npm already-published skip과 무관하게 **무조건** 돈다(오직 자산 존재 probe로만 게이트). `gh release upload --clobber` → 전 자산 재다운로드 → SHA256 재검증. **정확히 13개 자산(12 바이너리 + `SHA256SUMS`)이 검증되지 않으면 release 잡 실패.** 검증이 무조건이므로 `skip_platform=true`는 **이전 시도가 이미 자산 업로드를 마친 재실행에서만** 유효하다. 자산 없는 상태의 `skip_platform=true`는 fail closed다.

재시도 규칙: 일시적 실패 → `gh run rerun --failed <run-id>`(원래 워크플로 리비전·SHA·inputs 유지). 워크플 수정/버전 오류/`skip_platform`·`publish_lazycodex` 입력 변경이 필요하면 **새 dispatch**. `prepared_release_sha`를 직접 넘기지 말 것(내부 입력). 손으로 npm publish하거나 태그를 옮기거나 `--admin`으로 체크를 우회하지 말 것 — 문서 마지막 문장이 명시적으로 금지한다.

**Post-fix repro 요구사항**: race-condition·동시성 수정은 이슈를 닫기 전 **원래 제보자가 재현 코드를 재실행**하고 이슈 스레드에 `Repro retested: PASS/FAIL on commit <SHA>`를 남겨야 한다. 재현이 불가하면 이슈 클로즈 코멘트와 릴리스 노트 양쪽에 `Fix unverified end-to-end`으로 기록. 문서는 issues #4006, #3996, #3962을 회귀 사례로 인용하고 #4012(prompt-async-gate)의 유발 버그를 계기범으로 든다. **이 문서 규칙은 8개 하니스 중 유일하게 "릴리스 후 수동 검증"을 강제한다.**

### 8.8 컴파일 바이너리 설치 경로

```sh
curl -fsSL https://get.omo.dev/install.sh | bash            # latest
curl -fsSL https://get.omo.dev/install.sh | bash -s -- beta # beta 채널 또는 명시 버전
irm https://get.omo.dev/install.ps1 | iex                    # Windows
```

| 항목 | 동작 |
|---|---|
| 자산 선택 | OS · CPU · libc · **AVX2 지원** 기준. Rosetta 셸이면 Apple Silicon 빌드 |
| 무결성 | release `SHA256SUMS` 대조 |
| 설치 위치 | `~/.local/bin/omo` (`%USERPROFILE%\.local\bin\omo.exe`) |
| PATH 수정 | 셸 프로파일에 마커 블록 `# >>> omo installer >>>` … `# <<< omo installer <<<`으로 삽입 → **재실행해도 중복 추가 안 됨** |
| 영수증 | `~/.omo/install.json` |
| 거절 조건 | root/Administrator로 실행 거부 |
| 미러 | `get.omo.dev` = Cloudflare R2 복제본. 미러에 버전 없거나 도달 불가면 **GitHub Releases에서 같은 파일을 받아 동일하게 검증** |
| 환경변수 | `OMO_INSTALL_DIR`(설치 경로), `OMO_NO_MODIFY_PATH=1`(프로필 무수정) |
| WSL | 배포판 안에서 Linux 명령 실행 → Linux 빌드 설치 |
| macOS 주의 | 바이너리는 **서명 미서명**. 브라우저 다운로드엔 `com.apple.quarantine`가 붙어 Gatekeeper가 suspend. `curl`은 붙이지 않음. `xattr -d`로 해제하는 우회 금지 |
| 첫 실행 | `~/.omo/binary-runtime/<version>/`에 런타임 프로비저닝 후 **그 자리에서 재실행**. 버전당 1회 |
| 업데이트 | `install.sh` 재실행(새 버전일 때만 교체) 또는 `update` 서브커맨드가 플랫폼별 정확한 `curl` 명령 출력 |
| 제약 | `--inspect`/인스펙터 모드 미지원(디버깅은 npm 설치 사용) |
| 채널 함정 | **stable 릴리스의 컴파일 바이너리도 beta 채널 엔진에서 빌드된다** — 문서 스스로 "beta 품질 native 빌드"라고 명시 |

### 8.9 Desktop 업데이트 — `docs/guide/desktop-updates.md`, `packages/senpi-desktop-service`

- 시작 시 + 이후 수 분 간격으로 새 버전 확인. **다운로드·설치는 사용자 승인 전까지 전혀 실행되지 않는다.**
- 사이드바 업데이트 버튼 → 변경 내용 확인 → **Download** → **Install** → 재시작
- **Update track: Stable / Nightly** (Settings, 버전 옆). Nightly는 대체로 매일 빌드
- 건너뛴 버전이 여러 개면 각 버전의 변경 사항을 최신순으로 나열
- Apple Silicon에서 Intel 빌드로 돌면 Rosetta 경유 경고 → 다음 업데이트에서 네이티브로 전환
- 업데이트 실패 시 **아무것도 바뀌지 않음**(지금 버전 유지). 온라인 확인 후 재시도
- 코드 측: `packages/senpi-desktop-{protocol,engine,prelude,service,tool}` 5개 패키지. `test:desktop`이 `script/check-cargo-pinned-deps.mjs`(Cargo 의존성 pin 검증)로 시작. Rust crate + TS 혼합.
- **확인 필요**: 데스크톱 자동 업데이트가 `senpi-desktop-service` 안에서 어떤 트리거/프로토콜로 구현되는지는 이번 스캔에서 미확인.

### 8.10 배포 전략이 함의하는 것

| 관찰 | 함의 |
|---|---|
| optionalDependencies에 `^`가 없고 exact pin | npm이 플랫폼 패키지를 호환 버전으로 자유롭게 올릴 수 없다. 래퍼-BinVersion 스큐를 npm이 강제함 |
| baseline × musl 분기 | CPU(AVX2)와 libc를 매핑해야 하는 폭넓은 하드웨어/컨테이너 기반을 위한 선택. 설치 실패를 런타임이 아니라 설치 시점에 흡수 |
| 플랫폼 패키지 12개 동시 publish | 단순한 단일 아티팩트 배포보다 훨씬 복잡한 릴리스 상태 관리. 그래서 `publish-platform.yml`에 별도 probe와 2단 dispatch가 필요한 구조 |
| npm 래퍼와 Release 바이너리가 **다른 빌드 산출물** | npm 쪽은 Node 런처 스텁 + Bun 실행본, Release 쪽은 진짜 단일 파일. 기능 편차가 생길 수 있음(Windows ARM64이 실제 사례) |
| Release 자산 검증이 무조건 | 자산 없는 릴리스가 구조적으로 불가능 |
| stable 바이너리가 beta 엔진 | 채널 표기와 실제 코드 불일치. 품질 관점에서 사용자가 인지하기 어려운 부분 |
| Desktop에 별도 Stable/Nightly 트랙 | npm(일일)·GitHub Release(릴리스)·Desktop(Nightly) 3개의cadence가 공존. "업데이트"라는 단어의 의미가 경로마다 다르다 |

---

## 9. 설정 디버깅 체크리스트

### 9.1 어떤 파일이 이기고 있는가

```bash
# 1) 후보 경로 전부 보기 — user 1개 + project 조상들
#    (코드의 후보 순서: user → farthest project → ... → nearest project)
#    직접 보려면: $HOME/.omo/omo.jsonc 존재 여부, cwd부터 $HOME까지 각 디렉토리의 .omo/omo.jsonc
ls -la ~/.omo/omo.json*                     # user 레이어
find . -maxdepth 6 -name 'omo.json*' -path '*/.omo/*' 2>/dev/null   # project 레이어

# 2) 최종 판단은 프로그램에 맡긴다
omo doctor --json | head -50               # config 체커 결과
omo doctor --verbose                        # 진단 원문
omo doctor --status                         # compact 대시보드
```

판단 규칙:

| 상황 | 해석 |
|---|---|
| `$HOME/.omo/omo.jsonc`가 `.omo` 디렉토리 자체를 가리키는 심볼릭 링크 | user 레이어 미인식. `isSymlinkedProjectPath()`가 project만 걸러주므로 user 쪽은 별도 확인 필요 |
| 홈 디렉토리의 `.omo/omo.jsonc` | **project가 아니라 user로만 로드된다.** `findProjectConfigPathsFarthestFirst()`가 홈에서 walk를 멈추고 홈 `.omo`를 user로만 청구 |
| 중첩 256단계 아래 | 그 아래는 walk되지 않음 |
| 조상이 `unrecognized_keys` 진단을 낸다 | 그 키는 삭제된 상태로 merge되고 있다. strict 검증이 조용한 무시를 허용하지 않는다 |

### 9.2 하니스 블록이 적용됐는지

```jsonc
{
  "[senpi]": { "telemetry": { "enabled": false } },   // legacy — 먼저 깔림
  "[native]": { "telemetry": { "enabled": true } },   // canonical — 이김
  "[opencode]": { "plugin": ["oh-my-opencode@5.1.13"] }  // free-form, 검증 없음
}
```

- `[senpi]`와 `[native]`에 **같은 키**가 있으면 `[native]`가 이긴다. 런타임에 `deprecated-keys` 진단이 뜬다.
- 진단에 `[senpi] ignored because [native] is also configured`가 아니라 `renamed to [native]`가 붙으면 충돌이 없다는 뜻 — **치환이 아니라 병합**이다.
- `[opencode]` 안의 오타는 OmO가 절대 잡지 않는다. `omo doctor`의 `tui-plugin-config` 체커나 opencode 자체가 잡는다.

### 9.3 프로필이 붙었는지

```bash
echo "OMO_PROFILE=${OMO_PROFILE:-<unset>}"
echo "OCX_PROFILE=${OCX_PROFILE:-<unset>}"
echo "OPENCODE_CONFIG_DIR=${OPENCODE_CONFIG_DIR:-<unset>}"
```

- 우선순위: 명시 인자 > `OMO_PROFILE` > `OCX_PROFILE` > `OPENCODE_CONFIG_DIR`의 `profiles/<name>` tail
- **빈 문자열은 unset과 동일** 처리된다. `export OMO_PROFILE=` 가 프로필을 끈다.
- 프로필 이름이 틀리면 throw 없이 base로 폴백하고 `kind: "profile"` 진단만 남는다. 진단을 못 읽으면 "프로필이 조용히 무시된" 것으로 보인다.
- `OPENCODE_CONFIG_DIR` 방식은 경로가 `.../profiles/<이름>`으로 **끝나야** 인식된다. 중간의 `profiles/`는 무시된다.

### 9.4 머지가 예상과 다른 경우

```jsonc
{ "task": { "members": ["a", "b"] } }     // project
{ "task": { "members": ["c"] } }           // user  → 결과는 ["c"]
```

| 증상 | 이유 |
|---|---|
| 배열이 이어 붙지 않고 통째로 바뀜 | deep-merge는 plain object에만 적용. 배열은 교체 |
| project의 배열 일부가 사라짐 | 정상. nearest가 이긴 결과 |
| user의 `disabled_skills`가 사라지지 않음 | **의도된 예외.** `disabled_skills`는 레이어 간 union |
| user의 스킬 denylist가 project에서 못 지움 | union이므로 project에서만 제거할 수 없다. 제거하려면 user에서 지워야 한다 |
| `__proto__`/`constructor`/`prototype` 키가 조용히 사라짐 | `sanitizeUnsafeKeys()`가 Zod 전에 제거하고 `unrecognized_keys`로 보고. 두 겹 모두 동일하게 제거 |

### 9.5 잘못된 값 진단

```
Ignored invalid value in <path>: <dotted key>: <message>
```

- `<dotted key>`는 제거된 최소 subtree의 경로. `task.host_engine_policy`면 값 하나만 죽고, `teams.alpha`면 그 객체째 죽는다.
- 죽은 컨테이너는 자식이 다 빠져서 비면 함께 사라진다. 그래서 `teams` 아래 모든 멤버가 잘못되면 `teams` 전체가 사라질 수 있다.
- sibling은 **모든 깊이에서** 살아남는다. `teams.alpha`가 죽어도 `teams.beta`는 남는다.
- **JSONC 문법 오류만 예외** — 그 레이어는 통째로 로드되지 않는다(`kind: "parse"`).
- `Invalid omo config at <path>: ...`(`kind: "validation"`)가 남았다면 32패스 한계에 걸렸거나 **문서 루트 자체의 문제**다. 이때는 fail-closed로 이전 완전한 값이 유지된다.

### 9.6 마이그레이션 dry-run

```bash
omo config migrate --dry-run            # 텍스트
omo config migrate --dry-run --json     # 기계 판독
```

```
status: planned
target: ~/.omo/omo.jsonc
transform: { ... }                      # 병합 후의 최종 대상
move-plan: [ { "from": "...", "to": "..." } ]   # 백업 이동 계획
conflict: <diagnostic>                  # 손으로 해결 필요한 충돌
```

| 확인 포인트 | 방법 |
|---|---|
| 락이 걸려 있나 | `cat ~/.omo/.migration.lock` → `{ "leaseExpiresAt": ..., "pid": ... }`. 살아 있으면 30초 lease |
| 저널이 남아 있나 | `cat ~/.omo/.migration-journal.json`. 있으면 다음 실행이 `Recovered pending migration journal.` 출력 후 복구 |
| dry-run이 정말 안 썼나 | dry-run은 `status: "planned"`로 즉시 반환하고 `afterMigrations` 후크도 건너뛴다. **저널·락·백업·타깃 어디에도 흔적 없음** |
| 백업 위치 | `--json`의 `migrations[].movePlan`이 `{from,to}`를 준다. 백업 경로 규칙(`migration-backup-<ts>/`)은 확인 필요 |
| 충돌 수 | `--json`의 `skippedConflictCount`. 0이 아니면 자동 마이그레이션으로 넘어가지 않은 항목이 있다는 신호 |
| 실제 적용 전 확인 | `--json`의 `transform`을 눈으로 확인한 뒤 `--dry-run` 없이 재실행 |

### 9.7 상태 디렉토리 확인

```bash
ls ~/.omo/                                  # 설정, .migration.lock, .migration-journal.json,
                                            # binary-runtime/, install.json
ls .omo/                                    # plans/ goal/ ulw-loop/ notepads/ teams/ rules/
ls .omo/binary-runtime/                     # 컴파일 바이너리 런타임 프로비저닝 버전들
ls ~/.omo/binary-runtime/                   # 버전이 쌓인다 — 정리 여부 확인 필요
```

---

## 요약

- **설정은 `omo.jsonc` 2종**(user `~/.omo/omo.jsonc`, project `<조상>/.omo/omo.jsonc`)이지만 실제 규칙은 더 섬세하다: `omo.json` 폴백, 256단계 상한, 홈 경계 제외, 심볼릭 링크 스킵, realpath 비교, **farthest-first 배열 + 순차 merge가 emergent하게 만드는 nearest-wins**.
- **해석은 4레이어 순차 deep-merge**: shared → `[harness]` → profile-shared → profile-harness. 레거시 `[senpi]`는 `[native]`보다 먼저 깔린다.
- **스키마는 Zod v4 strict, top-level 22개**. `docs/reference/configuration.md`와 상위 정리의 키 목록이 실제 스키마와 불일치하고(`formatOnMutation`/`gateway`/`computer` 누락), `snake_case` 규칙에도 `formatOnMutation` 예외가 있다. `[opencode]`만 유일하게 무검증 free-form이다.
- **프로필 우선순위 `OMO_PROFILE > OCX_PROFILE > OPENCODE_CONFIG_DIR` tail은 코드에 그대로 존재**하며, 빈 문자열은 unset으로 취급된다.
- **머지는 plain object만 deep-merge, 배열·스칼라·클래스 인스턴스는 교체**. 프로토폴 오염 방어는 Zod 전후 2겹. `disabled_skills`만 union 예외.
- **견고성의 핵심은 "최소 subtree drop"**: 경로가 지정하는 가장 작은 단위만 지우고 모든 깊이의 형제와 멀쩡한 값을 보존하며, 32패스 한계 또는 루트 이슈에서만 fail-closed. 단, JSONC 문법 오류는 레이어 전체를 버린다.
- **마이그레이션은 lease+pid 락(30초, 2× stale)과 version 1 저널로 crash-safe**. `config migrate --dry-run`은 `planned`로 즉시 반환해 흔적을 남기지 않는다.
- **배포가 이 하니스의 진짜 차별점**: npm 래퍼 1 + exact-pin 플랫폼 패키지 12 + Release 단일 파일 바이너리 12 + SHA256SUMS + Desktop 2트랙. 2단 dispatch와 무조건 자산 검증(정확히 13개), 두 npm 패키지 계열(`oh-my-opencode-*` vs `lazycodex`의 `oh-my-openagent-*`), npm/Release 간 Windows ARM64 편차, stable 바이너리의 beta 엔진 문제 같은 실제 운 warts까지 문서화되어 있다. 다른 7개 하니스 중 이 수준의 릴리스 문서화를 갖고 있는 곳은 없다.
