# OmO Native — 하니스 그 자체로 본 네이티브 런타임

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13`
> 상위 문서: [oh-my-openagent.md](../oh-my-openagent.md) · 타깃 비교: [02-targets.md](02-targets.md) · DAG 엔진: [04-orchestration.md](04-orchestration.md)
>
> 이 문서는 **"Native가 플러그인 기능 하나가 아니라 하니스 그 자체"** 라는 질문에만 답한다. 4개 타깃의 기능 매트릭스는 02에 있고, 여기서는 반복하지 않는다.

## 한 줄 요약

OmO Native는 3개 층으로 분리된다. 그중 **프로세스 소유 층은 이 레포 밖에 있다.**

| 층 | 위치 | 규모 (지정 기준 수치) | 이 레포에 소스가 있는가 |
|---|---|---|---|
| **엔진 (호스트)** | `@code-yeongyu/senpi@2026.10.3` (npm) | 미상 — 레포 밖 | ❌ **외부 패키지. exact pin** |
| **확장 (런타임의 OmO 부분)** | `packages/omo-senpi/` | **122,133 LoC / 879 파일** | ✅ 25개 라이브 컴포넌트 |
| **DAG 엔진** | `packages/senpi-task/` | **53,551 LoC** | ✅ (04에서 다룸) |
| **런처/슈퍼바이저** | `packages/omo-native/` | **2,633 LoC / 27 파일** | ✅ `bin/` + 컴파일 진입점 |

핵심 사실 두 가지:

1. `@code-yeongyu/senpi`는 **이 레포에 벤더링되지 않은 외부 npm 패키지**다. `bun.lock`에 레지스트리 해석 결과(sha512 무결성 해시 포함)로 존재하고, `package.json`은 exact pin(`"2026.10.3"`)이다.
2. `packages/omo-senpi/`는 **엔진이 아니라 엔진 안에서 도는 확장**이다. 즉 이 레포가 소유하는 것은 "하니스의 확장과 배포 계층"이지 "프로세스"가 아니다.

---

## 1. "OmO Native"는 실제로 무엇인가

### 1.1 이름이 감추는 것

`installation.md`는 이 제품군을 "**three editions** of the same product: two plugins that load into a host you already run, plus one standalone edition"로 규정한다. "standalone edition"라는 말이 오해를 부른다 — **standalone인 것은 배포 형태일 뿐, 런타임이 아니라서가 아니다.** Native 에디션도 결국 어떤 엔진 프로세스 안에서 도는 확장이다. 그 엔진이 npm으로 들어오는 외부 패키지다.

### 1.2 외부 엔진 `@code-yeongyu/senpi` — 벤더링 아님 (검증됨)

`packages/omo-native/package.json`:

```json
{
  "name": "omo-ai",
  "bin": { "omo": "bin/omo.js" },
  "files": ["bin", "plugin"],
  "dependencies": { "@code-yeongyu/senpi": "2026.10.3" }
}
```

`bun.lock`의 해당 엔트리(611행 부근)를 읽으면:

```
"@code-yeongyu/senpi": ["@code-yeongyu/senpi@2026.10.3", "", {
  "dependencies": { "@anthropic-ai/claude-agent-sdk": "0.3.286", ...,
                    "@code-yeongyu/senpi-codemode": "2026.10.3",
                    "@earendil-works/pi-agent-core": "npm:@code-yeongyu/senpi-agent-core@2026.10.3",
                    "@earendil-works/pi-ai": "npm:@code-yeongyu/senpi-ai@2026.10.3", ... },
  "bin": { "pi": "dist/bundle/cli.js", "senpi": "dist/bundle/cli.js" }
}, "sha512-Uv8vzBaW…"]
```

이것이 말해주는 것:

- **레지스트리 패키지다.** 세 번째 필드(빈 문자열 = tarball 경로 없음)와 네 번째 필드(sha512 무결성 해시)가 붙어 있는 정규 레지스트리 해석이다. `packages/` 하위에 `senpi` 소스가 없고, `scripts/`에도 이걸 내려받아 소스로 편입하는 코드가 없다.
- **exact pin이다.** caret/tilde가 아니라 `"2026.10.3"`. `packages/omo-native/AGENTS.md`도 "the **exact-pinned** `@code-yeongyu/senpi` CLI"라고 명시한다.
- **bin이 두 개다:** `pi`와 `senpi` → 둘 다 `dist/bundle/cli.js`. 의존성 이름이 `@earendil-works/pi-*`인데 `npm:@code-yeongyu/senpi-*`로 alias되어 있다 — 즉 Senpi는 **pi 계열의 포크/리브랜드**다. 레포의 루트 AGENTS.md가 "Senpi/Pi packages"라고 함께 부르는 이유가 이것이다.

**`docs/reference/re-export-shim-inventory.md`는 이 질문의 답이 아니다.** 그 문서는 `@oh-my-opencode/omo-opencode/src`와 `omo-codex/src`의 구 import 경로를 살리는 shim 241개를 세는 문서이고, `senpi`라는 문자열이 한 건도 없다. 즉 이 레포는 senpi를 re-export로 감싸지 않는다. **외부 의존으로 그대로 쓴다.**

**정직한 결론**: 이 레포만 읽고는 "Senpi가 세션 수명주기를 어떻게 소유하는지", "TUI가 어떻게 렌더링하는지", "`agent_settled`이 엔진 내부에서 언제 발화하는지"를 알 수 없다. 02의 Senpi 훅 설명은 `packages/omo-senpi/AGENTS.md`의 **계약 설명을 인용한 것**이고, 엔진 구현을 읽은 결과가 아니다. 이 문서에서 그것을 다시 확인했다.

### 1.3 레포 안의 런타임 세 계층

```
┌─ 프로세스 소유 ────────────────────────────────────────────────┐
│  @code-yeongyu/senpi  (외부 npm, exact pin 2026.10.3)          │
│  · 프로세스 · 세션 수명주기 · TUI · 모델 루프 · RPC · 데몬       │
│  · 소스: 이 레포에 없음                                          │
└────────────────────────────────────────────────────────────────┘
        ▲ --extension <pkgRoot>/plugin
        │  (npm 경로)                        │ (컴파일 경로: 정적 링크로 번들)
┌───────┴──────────────┐        ┌────────────┴─────────────────┐
│ packages/omo-senpi/  │        │ packages/omo-native/         │
│ 122,133 LoC          │        │  2,633 LoC                   │
│ 25개 컴포넌트        │◀──────▶│  bin/ 런처 + compile-* 진입점  │
│ + skills/ 10개       │ 源码 참조│  유일한 dep = senpi          │
└───────┬──────────────┘        └────────────┬─────────────────┘
        │                                    │
        ▼                                    ▼
┌───────────────────────┐         ┌──────────────────────────────┐
│ packages/senpi-task/ │         │ bin/lib/*.js (런타임 정책)     │
│ 53,551 LoC DAG 엔진  │         │ bun 재실행·shim·doctor·setup │
│ (04-orchestration)    │         └──────────────────────────────┘
└───────────────────────┘
```

**경계선은 세 군데에서 확인된다.**

1. `bin/omo.js` → `runLauncher()` → `spawn: senpi --extension <pkgRoot>/plugin`. `plugin/`은 gitignored이고 `bun run build:omo-native`가 스테이징한다.
2. `compile-entry.ts` → `import("../../node_modules/@code-yeongyu/senpi/dist/cli.js")` — 컴파일 바이너리 경로에서는 **엔진이 정적으로 번들된다**.
3. `computer.*` 설정이 `["native"]` 하네스로 제한된다 (4절).

### 1.4 컴포넌트 수 주의

`packages/omo-senpi/src/components/`의 실제 디렉터리 수는 **31개**다. `AGENTS.md`가 `component-list.ts` 등록 순서로 열거하는 **등록 컴포넌트는 25개**이고, `config-resolution`·`agent-home`은 등록되지 않은 헬퍼다. 31 − 25 − 2 = 4개가 남는데, 그중 `claude-code`는 `compile-entry.ts`가 `../omo-senpi/src/components/claude-code/index`로 **직접 import**한다(`applyCachedClaudeCode`, `readClaudeCodePin`, `cachedExecutablePath`). `browser-bridge`·`formatter`·`post-mutation`의 등록 상태는 이번 패스에서 확인하지 못했다.

---

## 2. 런처/슈퍼바이저 수명주기

### 2.1 두 개의 배포 형태

`packages/omo-native`에는 **서로 다른 두 형태의 실행 코드가 함께 있다.**

| | 해석(interpreted) 경로 | 컴파일(compiled) 경로 |
|---|---|---|
| 진입점 | `bin/omo.js` | `compile-entry.ts` → `bun build --compile` |
| 배포 단위 | npm `omo-ai`의 `files: ["bin","plugin"]` | 단일 실행파일 ~114MB |
| 엔진解決 | `resolveSenpi()`가 설치된 `senpi` bin을 PATH에서 찾음 | `node_modules/@code-yeongyu/senpi/dist/cli.js` 정적 번들 |
| 파일시스템 | 실제 디스크 | bun의 가상 FS(`$bunfs`) |
| 진단 | `bin/lib/doctor.js`, `bin/lib/config-doctor.js` … | `compiled-doctor.ts`, `config-doctor-runtime.ts` … |

`compiled-*` 접두사는 **"컴파일 바이너리 안에서만 동작하는 코드"** 다. 이 구분이 왜 필요한지는 `compiled-diagnostic-runtime.ts`의 주석이 정확히 답한다:

> "The compiled binary has no npm package layout: `bin/lib/package-paths.js` resolves `packageRoot` inside the binary's virtual filesystem and `resolveSenpi()` finds no installed engine, so the npm loaders behind doctor and setup coverage fail open and silently drop their lines."

즉 같은 기능(doctor, update, config 진단)에 **구현이 두 벌**이고, 컴파일 경로의 것은 "가상 FS에서 npm 로더가 조용히 라인을 버린다"는 실패를 메꾸기 위해 존재한다.

### 2.2 `bin/omo.js` 부트 시퀀스 (실제 코드)

```js
const scriptPath = fileURLToPath(import.meta.url)
ensureBunBinShim({ scriptPath })       // POSIX sh shim 복구 (fail-open)
const reexeced = await maybeReexecUnderBun({ scriptPath })
if (!reexed) {
  if (process.argv[2] === "setup") await runSetup(process.argv.slice(3))
  else await runLauncher()
}
```

주석이 설계 의도를 남긴다. `bun add -g` 설치는 `~/.bun/bin`의 심볼릭 링크를 거치는데 node가 main 모듈을 realpath로 해석하므로, 그 URL이 이미 bun 글로벌 트리 안을 가리키고 그 bun은 신뢰해도 된다고 판단한다. 그 외 설치(npm·프로젝트 로컬·bunx)는 발견한 bun을 한 번 프로브해 `BUN_MIN_VERSION`(1.4.0) 이상이면 인계한다. `OMO_RUNTIME=node`는 항상 node 유지, `OMO_RUNTIME=bun`은 항상 re-exec(하한 없음). POSIX 인계는 `execve`(argv[0] 포함)로 잔류 래퍼를 없앤다. `spawnSync`는 이 경로에 쓰지 않는다 — 수명주기 긴 인계에 동기 spawn은 금기.

`runLauncher()`(`bin/lib/launcher.js`)는 먼저 `earlyCommands = {install, remove, list, config, auth, app-server, host}`를 먼저 처리하고, 나머지를 `SENPI_BRAND` 프로필(name, `~/.omo/agent` 홈, `OMO_*` env 접두, wire identity, 업데이트 채널)을 주입해 `senpi --extension <pkgRoot>/plugin`으로 넘긴다. `--version`과 모든 자체 업데이트 표기는 런처가 답한다.

### 2.3 컴파일 경로의 각 파일

**`compile-entry.ts` (373줄, 패키지 최대 파일)** — 컴파일 바이너리의 메인 디스패치. `buildSenpiArgs`, `runCompiledDoctor`, `runInternalSupervisor`, `registerEngineRuntimeModules`, `ensureEnginePrepared`, `applyCachedClaudeCode`, `migrateHostSessionSockets` 등을 배선한다.

이 파일의 가장 중요한 주석(엔진 import가 왜 상대 리터럴인지):

```
//  - `@code-yeongyu/senpi/dist/cli.js` is not in senpi's exports map (only ".",
//    "./bun-runtime", "./rpc-entry", "./client"), so the bare subpath fails
//    exports enforcement at build time.
//  - bun's bundler only traces import() whose argument is a literal: a
//    module-level const or a runtime-resolved URL (import.meta.resolve +
//    pathToFileURL) drops the entire engine graph from the binary (1 module
//    bundled instead of ≈4000) and the latter also fails to resolve inside
//    $bunfs. Do NOT refactor these two literals into an indirection.
```

**즉 리팩터링 금지 규칙이다.** 이 두 리터럴을 변수로 뽑으면 번들된 모듈이 1개로 떨어진다.

**`compile-runtime.ts` (141줄)** — 임베디드 런타임 프로비저닝.

- `EmbeddedManifest` = `{ omoAiVersion, enginePin, manifestSha, entries[{relPath, sha256, mode, size}], buildInfo?, engineBuild?, releaseTarget? }`
- `selectRuntimeManifest()` — `omo-runtime/runtime-manifest.json`을 먼저 찾고, 없으면 이름이 비슷한 파일을 열어 `omoAiVersion`/`enginePin` 문자열이 있는지 확인한다. **같은 이름이면서 내용이 다른 에셋은 후보에서 제외.**
- `provisionEmbeddedRuntime()` — `.provisioned` 마커에 `manifestSha`가 있으면 즉시 return(멱등). 각 엔트리마다 **바이트 길이 + sha256 검증** 후 `mode` 그대로 쓰고 `chmodSync`, 마지막에 마커 기록. 하나라도 불일치면 throw.
- `materializeProvisionedExecutable()` — 바이너리가 ~114MB라 매번 재복사하면 벽시계 비용이 실감난다. 목적지 파일 크기만 같으면 건너뛴다(목적지는 같은 출처에서만 쓰이므로 크기 비교로 충분). POSIX는 `.tmp-<pid>`에 복사 → `chmod 0o755` → `rename` 원자 교체, `finally`에서 임시 파일 정리.
- `runningExecutablePath()` — Linux에서 ` (deleted)` 접미사 제거(자기 업데이트 중 실행 중 파일 처리), Windows에서 `.exe`면 argv[0].

**`provisioned-handoff.ts` (67줄)** — `planProvisionedLaunch`와 `handOffToProvisionedRuntime`을 제공해 "지금 실행 중인 이미지를 계속 쓸 것인가, 프로비저닝된 사본으로 넘어갈 것인가"를 결정한다. `isProvisionedExecutable()`은 `realpathSync` 양쪽을 비교해 같은 파일인지 판정한다.

관련된 사실 하나를 `compile-entry.ts` 주석에서: 프로비저닝은 **re-exec 없이 끝날 수 있다**(`materializeProvisionedExecutable`의 크기 가드가 그 경로를 흔하게 만든다). 그래서 `execPath`는 사용자의 설치 경로에 머무르고 페이로드만 `execDir` 아래에 놓인다. 그래서 `remapSenpiEnvironment()`가 `OMO_PACKAGE_DIR`을 `execDir`로 **명시적으로 박아 둔다** — "trusting the running image"하지 않기 위해.

**`supervisor-fast-path.ts` (32줄)** — 이 파일의 존재 이유가 코멘트에 다 있다:

> "A task shard's supervisor only owns the public socket and restarts its host, yet entering the engine through `dist/cli.js` evaluates the whole `main.js` graph before `main()` dispatches the route, once per shard. The binary enters the engine's own route module instead."

- `INTERNAL_SUPERVISOR_FLAG = "--internal-rpc-host-supervisor"` — **엔진의 `INTERNAL_SUPERVISOR_ROUTE_FLAG`(`modes/rpc/supervisor-route.ts`)와 반드시 같아야 한다.**
- `runInternalSupervisor()`는 엔진 `cli.ts`/`cli-main.ts`가 `main()` 전에 하던 프로세스 설정을 그대로 재현한다: `valid-cwd.js` import → `process.title = APP_NAME` → `PI_CODING_AGENT=true` → `AI_AGENT=APP_NAME` → `process.emitWarning` 무음 → `dispatchInternalSupervisor(args)`.
- **false를 반환하면 엔진이 그 argv를 거부한 것**이므로 호출자는 평소처럼 전체 CLI로 계속 진행한다. fast path는 우회로가 아니라 우선 경로다.

즉 **태스크 shard 수만큼 비용이 곱해지는 문제를 파일 하나로 죽인 것**이다. 세션마다 전체 `main.js` 그래프를 평가할 이유가 없다는 판단.

**`engine-runtime-modules.ts` (11줄)** — 스테틱 번들된 OAuth 플로우와 Bedrock/Cursor/Devin 프로바이더 모듈을 **senpi CLI 그래프가 로드되기 전에** 등록한다. Bun 컴파일 FS가 이들의 불투명한 동적 로더를 해석하지 못하기 때문. 호출은 CLI import 직전이며 **절대 모듈 스코프가 아니다**: "evaluating the provider graph costs every launch CPU time, and the answers omo gives itself (--version, update, doctor, setup, the RPC supervisor route) never reach pi-ai".

**`compile-args.ts` (41줄)** — `buildSenpiArgs`, `shouldPrintCompiledBanner`, `launchOptions`. `launchOptions`는 `--` 뒤의 사용자 메시지를 무시하고 그 **앞의 옵션만** 읽는다. `daemon adopt` 재플레이에서 `--no-extensions` 같은 플래그로 세션을 재생성해 플러그인 없이 뜨는 것을 막는 장치다(02의 §5.1과 연결).

**`build-info.ts` (174줄)** — 컴파일된 **개발용** 바이너리에 박히는 빌드 출처.

```ts
interface OmoBuildInfo { command: string; omo: {commit, committedAt, branch}; engine: {…} }
type EngineBuildScheme = "epoch" | "nodef"
```

`versionLine()`의 세 갈래가 이 문서에서 재현된다:

| 조건 | 출력 |
|---|---|
| `omoBuild` 파싱 성공 (개발 빌드) | `versionLines(info).join("\n")` |
| `engineBuild` 스탬프 `scheme === "epoch"` | `omo <ver> (engine: senpi <pin>+<epoch>.<sha7>; scheme epoch)` |
| 그 외 | `omo <ver> (engine: senpi <pin>; scheme nodef)` |

`nodef`의 뜻은 코드 주석에 있다: "**no epoch derived, so I2 never hands off**" — 컴파일 타임 epoch가 없을 때. `parseComponent`는 40자 SHA·ISO 날짜·비어있지 않은 branch를 강제하고, 하나라도 어긋나면 `undefined`로 떨어뜨린다(느슨한 파싱 안 함). `updateHint()`도 같은 분기: 개발 빌드면 "rebuild with: `bun run <command>`", 릴리스면 GitHub Release에서 에셋을 받으라고 안내한다.

**`changes.md` (774줄)** — 날짜 역순 엔트리 + 이슈 번호로 구성된 **엔진Native 런처 레벨의 변경 로그**. 예: `2026-10-02 - Gate each platform publish on that platform's release-binary smoke (#9385)`, `2026-10-01 - The gateway hook's integrity check and its tests agree on Windows paths (follow-up to #9243)`.

> ⚠️ 정직한 단서: 런처가 실제로 읽는 changelog는 `bin/lib/launcher.js`의 `join(pluginRoot, "CHANGELOG.md")` (= 스테이징된 플러그인 페이로드 쪽)다. **`changes.md`를 `bin/`에서 참조하는 코드는 이번 패스에서 찾지 못했다.** 즉 유지보수자용 기록으로 보인다.

**나머지 컴파일 진입점들**: `compiled-update.ts`(GitHub Releases 조회, `releaseAssetName`, `RELEASES_URL`), `compiled-diagnostic-runtime.ts`(2.6절), `category-coverage-entry.ts`/`task-config-entry.ts`(빌드 시 `plugin/runtime/category-coverage/index.js`로 번들되는 진입점).

### 2.4 컴파일 바이너리 vs npm 경로의 런타임 정책

| | npm `omo-ai` | 단일 바이너리 |
|---|---|---|
| 엔진 로드 | `resolveSenpi()` → PATH의 `senpi` | 정적 번들 (≈4000 모듈) |
| 첫 실행 | postinstall `bin/senpi-patch.mjs`가 엔진 트리에 `.omo-engine-prepared` 스탬프 | 엔진·플러그인 페이로드·런타임 리소스 전부 내장 |
| 프로비저닝 | 불필요 | `~/.omo/binary-runtime/<version>/`에 sha256 검증과 함께 언팩 |
| postinstall 실패 대응 | `ensureEnginePrepared`가 **매 실행 전** 엔진 준비를 시도 — `ignore-scripts=true`나 Bun의 blocked postinstall로 스크립트가 안 돌았어도 첫 실행에 복구 (#8713) | 해당 없음 |
| engine preparation | Claude Code UA floor · compile-safe css-tree 데이터 · RPC stream 가드 | 동일 |

`launch-spec-mode.js`도 같은 결론에 도달한다: `plugin/daemon-launch-spec.json`에서 group/world 쓰기 비트를 벗겨낸다. npm/bun이 설치자의 umask로 추출하기 때문에 **task host가 쓰기 가능한 스펙을 거부한다**(#9208). 실패하면 `omo doctor`가 FAIL 줄과 수리를 같이 준다.

---

## 3. Doctor 패밀리 — 자기 진단 하니스

이건 02에서 언급되지 않은, **진짜 차별화 축**이다. 플러그인은 자기 호스트의 설치를 진단할 수 없다. Native는 자기가 소유한 엔진 바이너리까지 내려가 진단한다.

### 3.1 각 파일이 진단하는 것

| 파일 | 진단 대상 | 산출 |
|---|---|---|
| `bin/lib/doctor.js` (`runDoctor`) | npm 경로 루트 진단. 설치 상태·버전·업데이트 대상·설정 경고·마이그레이션·stale 엔진·retired 페이로드·일시 메모리·daemon·게이트웨이·computer-use·launch-spec | `PASS`/`FAIL`/`WARN`/`INFO` |
| `compiled-doctor.ts` (`runCompiledDoctor`) | 위와 **같은 목록을 바이너리 안에서**. 아티팩트 경로가 `execDir` 기준 고정 | 동일 |
| `config-doctor-runtime.ts` | omo.json 로더가 버린 키, 로드하지 못한 파일 | `WARN config: <file>: <key> ignored (…)` |
| `computer-use-doctor-runtime.ts` | 데스크톱 엔진 **실제 기동 + ABI 핸드셰이크** | 6종 리포트 |
| `computer-use-engine-probe.ts` | 위가 쓰는 프로브 (1줄 re-export) | — |
| `claude-code-doctor.ts` | Claude Code 실행 파일의 **해석 순서 4단** | `INFO`/`PASS` |
| `compiled-diagnostic-runtime.ts` | 바이너리 안에서 npm 로더가 죽는 문제를 메꾸는 어댑터 | — |
| `bin/lib/launch-spec-mode.js` (`launchSpecDoctorLines`) | `daemon-launch-spec.json` 권한이 task host를 거부하는가 | `FAIL` + 수리 지시 |
| `bin/lib/category-coverage.js` (`doctorCoverageLines`) | 태스크 카테고리 커버리지 | fail-open |
| `bin/lib/doctor-migration.js` (`migrationReport`) | 설정 마이그레이션 상태 (`standalone: true`) | — |
| `bin/lib/doctor-pi-config.js` (`piConfigReport`) | pi/senpi 설정 리포트 | — |
| `bin/lib/daemon-doctor-report.js` | daemon 엔드포인트 | — |

### 3.2 `runCompiledDoctor`가 실제로 하는 일

아티팩트 3개를 하드코딩해 existence로 확인한다:

```
["plugin manifest", "plugin/package.json"]
["extension",       "plugin/extensions/omo.js"]
["lsp-daemon runtime", "plugin/runtime/lsp-daemon/dist/cli.js"]
```

그 뒤 순서대로 누적한다: 버전 라인(`INFO`) · 업데이트 명령(`INFO`) · launch-spec(`FAIL`이면 exit 1) · daemon 리포트 · 마이그레이션 리포트 · 설정 경고 · config 진단 · pi 설정 · stale 엔진 · 일시 메모리 → 마지막으로 **컴퓨터유저·claude-code·커버리지 라인을 `Promise.all`로 병렬 수집**.

`FAIL`이 하나라도 있으면 `process.exitCode = 1`. 그리고 `--reap` 인자가 첫 자리에 있으면 진단 대신 **stale 엔진 회수 모드**로 들어간다.

### 3.3 `computer-use-doctor-runtime.ts` — 가장 깊은 하나

이 파일은 doctor가 **실제로 엔진을 띄워서 ABI 핸드셰이크까지 수행한다.** 순서는:

1. `getDesktopEngineHost(platform, arch)` + `isSupportedHost(platform)` — 미지원이면 `skipped/unsupported`.
2. `loadOmoConfig({ cwd, env, harness: "native" })` → `resolveOmoComputerSettings(config.computer, platform)`. `enabled: false`면 `skipped/disabled`.
3. `resolveInstalledEngine(settings.enginePath, env, version, locatorOptions)` — 후보 경로는 `runtimeDir = env.OMO_PACKAGE_DIR`, `execDir = dirname(process.execPath)`, `packageDir = resolve(packageRoot, "..", "senpi-desktop-engine")`, `repoRoot = resolve(packageRoot, "..", "..")`(개발 빌드 폴백).
4. `launchDesktopEngine(…, { allowDownload: false, cacheDir: ~/.omo/cache/senpi-desktop-engine, installDir: ~/.omo/engines/senpi-desktop-engine })` — **`allowDownload: false`. 진단 중에는 절대 다운로드하지 않는다.**
5. 프로브 → `hello { protocolVersion, engineVersion, buildSha, abi }` + `capabilities`.

리포트 6종:

| kind | 코드 | 뜻 |
|---|---|---|
| `ready` | — | 엔진 기동 + 핸드셰이크 성공. 경로·출처·hello·capabilities |
| `skipped` | `disabled` \| `unsupported` | 설정 off / 미지원 호스트 |
| `unavailable` | `native-unavailable` \| `quarantined` | 엔진 없음 / macOS 격리 속성. `attemptedPaths` 전부 노출 |
| `not-installed` | — | 설치된 엔진도 설정된 경로도 없음. **첫 사용 시 다운로드된다** |
| `failed` | `abi-mismatch` \| `handshake-failed` \| `timeout` | 프로브 실패 |

`quarantined`(macOS 다운로드 격리)와 `abi-mismatch`(다른 릴리스의 엔진)는 서로 다른 실패를 구분한다 — 사용자에게 전부 "엔진 없음"으로 보이는 게 아니라, 각각 원인이 다르다. `attemptedPaths`를 전부 찍는 것도 이 때문이다.

### 3.4 `config-doctor-runtime.ts` — 자기.view로 진단

```ts
const { diagnostics } = loadOmoConfig({ cwd, env, harness: "senpi" })
return omoConfigDiagnosticLines(diagnostics, { homeDir: resolveHomeDir(env) }).map(l => `WARN ${l}`)
```

주석이 중요도를 말한다: "**read through the same view the extension loads** (omo-senpi config-resolution: harness "senpi")". 즉 doctor가 **자기만의 설정 파서를 갖지 않는다.** 확장이 실제로 로드하는 뷰를 그대로 호출해, 확장이 버릴 키와 못 읽을 파일을 그대로 보고한다. 파서가 둘로 갈라질 수 없다.

> 🔍 **검증된 비대칭**: `config-doctor-runtime.ts`는 `harness: "senpi"`를 쓰고 `computer-use-doctor-runtime.ts`는 `harness: "native"`를 쓴다. `COMPUTER_HARNESS_SUPPORT`가 `computer.*`를 `["native"]`로 매핑하므로 `computer` 블록은 native 뷰에서만 해석되는 것이 의도된 설계다. 일반 키 진단은 확장 뷰(`senpi`), `computer.*` 진단은 네이티브 뷰(`native`). 우연이 아니라 하네스 ID 2개 체계를 쓰기 때문에 생긴 갈림이다.

### 3.5 `claude-code-doctor.ts` — 해석 순서 4단

`readClaudeCodePin(runtimeDir)`이 `undefined`를 주면 **아무 줄도 내지 않는다**(핀 자체가 없으면 진단할 대상이 없다). 이후:

1. `env.CLAUDE_CODE_EXECUTABLE` → `INFO Claude Code: <경로>`
2. PATH의 `claude` → `INFO Claude Code: <경로> (on PATH)`
3. `runtimeDir/claude-code/<pin>` 캐시 존재 → **`PASS Claude Code <version>: <경로>`**
4. 그 외 → `INFO ... not downloaded yet; it is fetched (integrity-checked) on the first anthropic-subscription turn`

`PASS`가 되는 유일한 단계가 캐시命中이라는 점이 정확하다. 사용자 관점에서 "설치됐다"의 정의가 그 경로이기 때문이다.

### 3.6 stale 엔진 회수 — 패턴 킬 금지

`bin/lib/doctor.js` `classifyEngineProcesses`가 살아있는 엔진을 세 갈래로 나눈다: **stale**(interactive, PPID 1) / attached / **managed**(`--mode` 플래그). 그리고 `reapStaleEngines`는 **명시적으로 넘긴 pid 중, 회수 요청 시점에도 여전히 stale인 것만** 종료한다.

거부 사유가 코드에 열거돼 있다:

| 거부 | 사유 |
|---|---|
| pid가 아님 | `--reap takes process ids` |
| omo 엔진 프로세스가 아님 | "it is not an omo engine process" |
| managed 엔진 | "`owned by whatever started it`" |
| launcher(ppid)가 살아 있음 | "it is a live session" |

그리고 `changes.md`/AGENTS.md가 명시: **"Pattern-kill is forbidden."** 자식이 자기 부모를 죽이면 남는 게 있다는 판단이다.

여기에 더해 `classifyRetiredPayloadEngines`가 "이 페이로드를 설치하기 전에 시작된 엔진"(=`WARN ... it still runs the previous plugin copy`)을 찾아내고 "**a running engine is never rewritten in place**"를 출력한다. `transientMemoryReport`는 메모리 identity를 durable/transient로 세고 transient run root 수를 알려준다.

### 3.7 왜 이게 차별화인가 (정리)

플러그인 타깃의 `doctor`는 **자기가 올라간 호스트의 설치**를 본다. Native의 doctor는 **자기가 소유한 엔진 바이너리의 ABI**까지 본다. 이건 02의 기능 매트릭스에 없는 축이다. 그만큼 자기가 소유한 프로세스가 많다는 뜻이고, 그게 4절의 논리로 이어진다.

---

## 4. 왜 플러그인은 못 하고 네이티브는 할 수 있는가

### 4.1 구체적 증거: `computer.*` → `["native"]`

`packages/omo-config-core/src/schema/computer.ts` — **하네스 중립 core 패키지 안의 설정 스키마**에 이 지도가 박혀 있다.

```ts
export const COMPUTER_HARNESS_SUPPORT: Record<ComputerSettingPath, readonly OmoHarnessId[]> = {
  "computer.enabled": ["native"],
  "computer.display": ["native"],
  "computer.max_width": ["native"],
  "computer.max_height": ["native"],
  "computer.screenshot_max_bytes": ["native"],
  "computer.stop_hotkey": ["native"],
  "computer.allow_host_relay_only_stop": ["native"],
  "computer.macos_canary": ["native"],
  "computer.audit_log": ["native"],
  "computer.screenshot_gc": ["native"],
  "computer.engine_path": ["native"],
  "computer.cua_adapter": ["native"],
} as const
```

12개 키 **전부** `["native"]`. 예외 없음. 그리고 같은 파일의 `enabled` 설명문: "register the computer tool in **OmO Native** sessions".

이것이 왜 결정적인 이유는, **제한이 런타임 체크가 아니라 설정 스키마 단계에 있기 때문이다.** `computer.*` 키는 모든 에디션의 `omo.jsonc`에 존재할 수 있다. 그런데 공유 core 스키마가 **"이 키를 해석할 수 있는 하네스는 native 하나뿐"** 이라고 선언한다. 다른 에디션의 어댑터는 이 키를 받으면 조용히 버린다. 즉 권한이 코드로 박혀 있다.

### 4.2 왜 플러그인은 구조적으로 불가능한가

`packages/omo-opencode/src/index.ts` 전체가 이것이다:

```ts
import type { PluginModule } from "@opencode-ai/plugin"
const pluginModule: PluginModule = createPluginModule()
export default pluginModule
```

플러그인이 얻는 것:

| | 플러그인 | 네이티브 |
|---|---|---|
| 프로세스 | **호스트의 프로세스** (자기 자신의 자식이 아님) | **자기 자신의 프로세스**. `omo doctor`가 형제 엔진 pid를 판별하고 죽일 수 있다 |
| 개입 지점 | 호스트가 정의한 이벤트(OpenCode 14개 핸들러 / 20 hook 포인트) | 호스트 내부 + 그 바깥. `agent_settled` 같은 엔진 내부 이벤트에 직접 등록 |
| 툴 등록 | 호스트의 툴 레지스트리. 호스트가 모르는 권한 등급은 만들 수 없음 | 자기 권한 등급(`permissionParser`, `kernelPrelude`)을 정의 |
| 자식 프로세스 | 호스트의 셸 툴을 "借用"해야 함 | 직접 `spawn`. 예: `spawn: senpi --extension ...` |
| 세션 수명주기 | 없음. 세션은 호스트의 것 | `omo daemon run/adopt/status/stop`, `p-<key>.sock` 태스크 호스트, `i-*` Desktop 스레드 호스트 |
| OS 권한 | 없음 | 스톱 코드(단축키) 암기, 스크린 레코딩/접근성 권한 요청 |
| 자기 진단 | 호스트 설치만 가능 | 엔진 바이너리 ABI까지 |

**일반 원칙**: 플러그인은 호스트의 샌드박스 **안에서** 행동을 추가한다. 하니스는 호스트 아래에 **한 층을 더 놓는다.** capability 표면 = (호스트가 노출하는 것) × (호스트가 허용하는 것)이므로, 호스트가 없는 자리에 서는 순간 그 곱이 0이 아니라 **무한대**가 아니라 **호스트가 정한 값**이 된다. Native가 process를 소유하므로 process 표면의 값은 "호스트가 정한 값"이 아니라 "직접 정한 값"이 된다.

### 4.3 native가 실제로 소유하는 것 (검증된 목록)

- **엔진 프로세스 감독**: `.omo-engine-prepared` 스탬프, `classifyEngineProcesses`(stale/attached/managed), `reapStaleEngines`(명시 pid만), `classifyRetiredPayloadEngines`
- **데스크톱 세션**: Rust 엔진의 `--serve <socket>` 데몬 + `--oneshot` 브리지, `~/.omo/engines/senpi-desktop-engine` 설치 디렉터리, `~/.omo/cache/senpi-desktop-engine` 캐시
- **ABI 협상**: `hello {protocolVersion, engineVersion, buildSha, abi}`, 실패 코드가 `abi-mismatch`/`handshake-failed`/`timeout`으로 구분
- **RPC 호스트**: `rpc.sock` 운영자 데몬, 세션별 `p-<key>.sock`, `OMO_RPC_SHARD_ROOT` 샤드 라우팅, `supervisor-fast-path`로 shard마다 엔진 그래프 평가 회피
- **에이전트 상태 단일 위치**: `canonicalAgentDir()` = `~/.omo/agent`. 런처·doctor·setup·로컬 설치 런처가 **모두 그곳에서 해석**하며, 자기 기본값을 조립하는 곳은 없다. `adoptLegacyFlatState()`가 평면 `~/.omo` 레이아웃을 1회 이관. 루트 AGENTS.md가 이걸 명시적 금지 조항으로 둔다: "Composing a private default is what made settings look erased on update."

---

## 5. `senpi-desktop-*` 5패키지 분해 — 가설 검증

사용자가 제시한 가설("engine = 실제 데스크톱 엔진, `--stdio`/`--serve`/`--oneshot`/`--mcp` 모드 보유")은 **오답이다.** `package.json` description과 실제 플래그 위치로 검증한다.

| 패키지 | `package.json` description | 의존 | 역할 |
|---|---|---|---|
| `senpi-desktop-protocol` (576) | "Engine JSON-RPC types, session snapshot, error codes, and computer-call approval tiers" | (없음) | 리프. 와이어 타입 |
| `senpi-desktop-engine` (1,253) | "**Locator, ABI handshake, and dev-build fallback** for the senpi-desktop-engine binary" | protocol | **엔진이 아니라 엔진의 로케이터** |
| `senpi-desktop-service` (2,251) | "JSON-RPC client over the senpi-desktop-engine **child process** and the computer.run runtime" | engine, protocol | 자식 프로세스 클라이언트 |
| `senpi-desktop-tool` (1,428) | "The senpi **computer tool definition, permission tiers, settings, and `/computer` command**" | prelude, protocol, service, typebox | 모델 대면 |
| `senpi-desktop-prelude` (623) | "Codemode **prelude assets and model documentation** for the senpi computer tool" | (없음) | 부트스트랩 자산 |

**정정 1 — `--stdio`/`--serve`/`--oneshot`/`--mcp`는 TS 패키지 게다가 아니라 Rust 바이너리의 플래그다.** `crates/senpi-desktop-engine/src/cli.rs`:

```rust
#[command(group(ArgGroup::new("mode").args(["stdio","serve","oneshot","mcp","resume","selftest","schema"])))]
```

**정정 2 — 실제 엔진은 Rust다.** `packages/senpi-desktop-engine/`는 그 바이너리의 **발견기**에 불과하다. 실제 데스크톱 엔진은 `crates/senpi-desktop-engine`(그리고 형제 크레이트). 데스크톱 백엔드가 크레이트 단위로 분리돼 있다:

| 크레이트 | 측정 LoC (`.rs`) |
|---|---|
| `senpi-desktop-backend-macos` | 7,988 |
| `senpi-desktop-backend-wayland` | 7,155 |
| `senpi-desktop-backend-win32` | 6,350 |
| `senpi-desktop-backend-x11` | 5,859 |
| `senpi-desktop-session` | 5,254 |
| `senpi-desktop-engine` | 4,224 |
| `senpi-desktop-core` | 3,852 |
| `senpi-desktop-backend-atspi` | 2,036 |
| `senpi-desktop-backend-fake` | 1,748 |
| `senpi-desktop-safety` | 1,420 |
| `crates/desktop` | **0** (`.rs` 없음 — 존재하는 디렉터리일 뿐) |

Wayland/x11/macOS/win32를 백엔드로 분리한 이유: `docs/guide/computer-use.md`의 알려진 제한이 그대로 이 분할을 설명한다. "Linux: Wayland에서는 단일 윈도우 캡처·윈도우별 입력·`raise()`가 없고, 스톱 코드에 GlobalShortcuts 포털이 필요하다. macOS: 백그라운드 클릭이 프론트 윈도우 바로 아래로 윈도우를 올린다. Windows: UIPI." `backend-fake`는 테스트용(`fake_listener.rs`의 주석: "a backend that is live once started and never fires, so a fake `--serve`").

**왜 5개인가 — 4가지 이유가 코드에서 확인된다.**

1. **의존 그래프가 단방향 DAG다**: `protocol ← engine ← service ← tool`, `prelude → tool`. 순환이 없다. wire 타입이 리프이므로 프로토콜을 바꾸면 engine/service/tool이 전부 따라온다.
2. **프로토콜이 생성 산출물이다**: `protocol`에 `generate` 스크립트(`scripts/generate-engine-schema.ts` → `src/engine-schema.generated.ts`). `prelude`도 `generate`(`scripts/generate-assets.ts` → `assets.generated.json`). 02가 기록한 "memoized getter로 1회 읽기"가 이 파일의 상대다.
3. **Rust/TS 언어 경계는 `engine` 패키지 한 곳에서만 넘는다**: `engine`는 Rust 바이너리를 찾고(`locator.ts`) ABI를 잡는다(`hello`). 그 위는 전부 TS. 그래서 `service`가 "child process over JSON-RPC"라고 설명할 수 있다.
4. **모델 대면부를 격리한다**: `tool`만 `typebox`를 의존한다. 프로토콜 타입만 다루는 `protocol`이 스키마 생성 도구는 `typebox`를 몰라도 된다. `tool`은 `./registration` 서브패스 export를 따로 낸다 — 등록 경로와 정의 경로의 분리.

---

## 6. "bunshin" — 무엇이고, 왜 여기 있나

### 6.1 bunshin이 뭔가 (레포가 말하는 것만)

`grep -rn "bunshin"` 결과는 여섯 군데에 흩어져 있다. 그중 결정적인 두 줄:

```
crates/senpi-desktop-engine/src/oneshot.rs:1
  //! `--oneshot`: bunshin's sidecar contract. One JSON-RPC line in on stdin,

packages/senpi-desktop-engine/test/bunshin-descriptor.test.ts:6
  // Copied from bunshin packages/machine-sdk/src/handlers-sidecar.ts:44-70 at commit 0a21e2c1
  // (the `superRefine` …)
```

알 수 있는 것:

- bunshin은 **sidecar 계약**을 가진 별도 에이전트 런타임이다. 자체 `packages/machine-sdk/src/handlers-sidecar.ts`에 sidecar descriptor 검증기(zod `superRefine`)가 있고, 이 테스트는 그 검증 규칙을 커밋 `0a21e2c1`에서 **복사**해 왔다.
- 이 레포는 bunshin을 **벤더링하지 않는다.** 존재하는 산출물은 descriptor 생성기(`descriptor.mjs`), 참조용으로 체크인된 `desktop.capability.json`, 그리고 검증 규칙 복사본뿐이다. bunshin 본체·에이전트·권한 부여 로직은 모두 밖이다.
- 경로: `BUNSHIN_HOME ?? ~/.bunshin`, `BUNSHIN_CAPABILITY_DIR ?? $BUNSHIN_HOME/capabilities`, 대상 파일 `capabilities/desktop.json`. 테스트 픽스처는 `bunshinHome: "/home/user/.bunshin"`.

### 6.2 capability 설치 모델

`script/install-bunshin-desktop-capability.mjs`:

```bash
bun script/install-bunshin-desktop-capability.mjs                  # 로컬 엔진 탐색
bun script/install-bunshin-desktop-capability.mjs --engine <path>   # 특정 바이너리
bun script/install-bunshin-desktop-capability.mjs --remove          # 제거
```

동작은 **파일 하나 쓰는 것**이다:

1. `--remove`면 `rmSync(target, {force:true})` 하고 끝.
2. `--engine`가 없으면 `locateDesktopEngine({ repoRoot })`(개발 빌드 폴백), 있으면 `resolve(values.engine)`.
3. 버전은 `packages/senpi-desktop-engine/package.json`에서 읽는다.
4. `mkdirSync(capabilityDir)` → `desktop.json`에 `desktopDescriptor(...)` 결과를 JSON으로 씬다.
5. `BUNSHIN_CAPABILITY_DIR`가 env에 없으면 "`start the bunshin agent with BUNSHIN_CAPABILITY_DIR=...` so it loads the descriptor"를 출력한다.

**설치 = bunshin 에이전트가 시작 시 읽는 디렉터리에 선언 파일을 하나 넣는 것.** 프로세스를 띄우지 않고, 에이전트를 띄우지 않고, 데스크톱 권한도 부여하지 않는다. `docs/guide/computer-use.md:167`이 명시한다: "**Installing a descriptor does not itself start an agent or grant desktop permissions.**" 실제 시작·적재·권한 판단은 전부 bunshin 쪽이다.

### 6.3 descriptor의 내용 — 그리고 "드리프트를 구조적으로 막는다"

`packages/senpi-desktop-engine/bunshin/descriptor.mjs`:

```js
const SCHEMA_URL = new URL("../../../crates/senpi-desktop-core/schema/engine.schema.json", import.meta.url)
```

**Rust 크레이트의 JSON 스키마를 런타임에 읽는다.** 그래서 op 목록·효과·파라미터 스키마가 엔진과 갈라질 수 없다. 필터 규칙:

- `spec.hostOnly` / `spec.testOnly` / `method.startsWith("$/")` → 제외
- 예외 2개: `desktop.stop` → `stopPath.stop`, `desktop.stopPath.status`
- `spec.effect === "exec"` → `mutate`, 그 외 → `read`
- `$ref`는 `reachableDefinitions()`로 전이 폐쇄를 계산해 인라인으로 넣는다(각 op이 자기에게 필요한 것만 담는다)

결과 객체:

```js
{
  name: "desktop",
  kind: "sidecar",
  version,                                   // 0.1.0
  description: "senpi desktop engine: screenshots, native input, accessibility, and the stop path",
  ops, effects, schemas,
  sidecar: {
    executable,
    args: ["--oneshot", "--audit-path", `${bunshinHome}/desktop-audit.jsonl`,
           "--stop-chord", stopChord],
    protocol: "json-rpc-stdio",
    timeoutMs: 65000,
  },
  sandbox: { grant: "full_access" },
}
```

- `stopChord`: darwin은 `ctrl+opt+cmd+escape`, 그 외는 `ctrl+alt+shift+escape`. **호스트별로 다르다** — 데스크톱 전역 단축키는 OS 네이티브라 한 번 정하면 안 된다.
- `sandbox.grant: "full_access"` — `computer-use.md`의 "`computer.run` code runs with full host access"와 일치. descriptor는 이를 **요청**하고 bunshin이 결정한다. 설치가 권한을 부여하지 않는 이유다.

### 6.4 Rust 쪽 계약 — 노출 등급 강제

`crates/senpi-desktop-engine/src/oneshot.rs`:

```rust
pub const METHOD_PREFIX: &str = "desktop.";
match name {
  "stop" => Ok(("stopPath.stop".to_owned(), json!({ "source": "api" }))),
  "stopPath.status" => Ok(("stopPath.status".to_owned(), params)),
  _ => match Method::from_name(name).map(|m| m.spec().exposure) {
      Some(Exposure::Public)  => Ok((name.to_owned(), params)),
      Some(Exposure::HostOnly) => Err(MethodRejection::HostOnly),
      Some(Exposure::TestOnly) => Err(MethodRejection::TestOnly),
      None                     => Err(MethodRejection::Unknown),
  },
}
```

문서 주석이 나머지를 말해준다: "host-only engine methods never cross this bridge, except `desktop.stop` (`stopPath.stop {source:"api"}`) and `desktop.stopPath.status`, and **`stopPath.resume` never does.**" 즉 스톱은 줄 수 있지만 **스톱 해제(`resume`)는 에이전트에 절대 주어지지 않는다.** 한 방향으로만 열려 있는 안전장치다.

### 6.5 왜 이게 "하니스로서의" 성질인가

이건 세 번째 통합 표면이다. OmO Native는 데스크톱 엔진을 자기 세션 안에서만 쓰지 않는다. **자기 개념(에이전트·세션·메시지)이 전혀 없는 다른 에이전트 프레임워크에 그 엔진을 capability로 내보낸다.** 그 경계에 놓이는 건 코드 호출이 아니라 **JSON 선언 파일 하나**다.

설계 원칙이 선명하다:
- **선언은 스키마에서 파생한다** — 드리프트가 구조적으로 불가능하다.
- **권한 부여는 호스트 몫이다** — descriptor는 `grant`를 요청할 뿐 결정하지 않는다.
- **감사는 경로를 넘긴다** — `--audit-path`를 bunshin 홈으로 지정해, bunshin 에이전트의 감사 로그 뒤에붙인다.
- **스톱 경로는 노출하고 스톱 해제는 숨긴다** — 단방향.

---

## 7. 실전 가이드

### 7.1 설치 경로 3종 교차 확인 (`docs/guide/`)

`install.md` / `binary-install.md` / `installation.md`는 **모순되지 않는다. 같은 바이너리 설치의 다른 층위다.**

| 문서 | 다루는 것 | 실제 내용 |
|---|---|---|
| `docs/guide/install.md` | **권장 단일 명령** (설치의 기본값) | `curl -fsSL https://get.omo.dev/install.sh \| bash` / Windows는 `irm … install.ps1 \| iex`. OS·CPU·libc에 맞는 빌드 선택 → `SHA256SUMS` 검증 → `~/.local/bin/omo` 설치. **root/Administrator 실행 거부.** PATH 미설정 시 셸 프로필에 마크된 블록(`# >>> omo installer >>>`) 추가. 검증 후 `omo --version` |
| `docs/guide/binary-install.md` | 위의 **수동·미러 상세** | GitHub Release 단일 파일 `omo-<os>-<arch>`. 자체 완결형(엔진+플러그인 페이로드+런타임 리소스 내장), 첫 실행 시 `~/.omo/binary-runtime/<version>/` 프로비저닝. `OMO_INSTALL_DIR`, `OMO_NO_MODIFY_PATH=1`. 미러(get.omo.dev = GitHub Release의 CDN 사본) 미달 시 GitHub Releases에서 **같은 파일을 받아 같은 검증** |
| `docs/guide/installation.md` | **3 에디션 판정** | Ultimate(OpenCode) / Light(Codex) / Native. `bunx oh-my-openagent install`, `npx lazycodex-ai install`, `curl … install.sh` 또는 `bun add -g omo-ai`. "**Do not install plain `omo` from npm: it is an unrelated package by a different author.**" |

여기에 npm 경로가 하나 더 있다 — `omo-ai` 패키지 자체. `package.json` description이 "bun add -g omo-ai"이고, `postinstall`이 `node bin/senpi-patch.mjs`로 엔진 트리에 `.omo-engine-prepared`를 찍는다.

`install.md`는 "One command installs the native `omo` binary for your OS and CPU. **You don't need Node.js, npm or Bun.**"이라 명시한다. 이것이 `packages/omo-native/package.json`의 `engines.node >= 24`와 모순되지 않는다 — **npm 경로는 Node 24를 요구하고, 바이너리 경로는 요구하지 않는다.**

### 7.2 native vs OpenCode 플러그인 — 뭘 얻고 뭘 잃는가

| | **Native** | **OpenCode 플러그인 (Ultimate)** |
|---|---|---|
| 선행 준비 | 없음. `omo` 하나 | OpenCode 필요 |
| 에이전트 레지스트리 | Senpi 자체 모델. OpenCode 판의 11개 에이전트 없음(`config-startup`이 차이를 한 번만 notice로 보고) | **11개** |
| `team_*` 툴 | lead 전용 6개 | **12개** |
| 훅 | Senpi 이벤트 10개 위 컴포넌트 25개 | **5-tier, 54~62개** |
| 편집 | 표준 | **Hashline `LINE#ID`** 검증 |
| 위임 카테고리 | senpi-task 리졸버 | **9종** 명시 모델 |
| 데스크톱(`computer.*`) | **가능** (12키 전부 native) | 불가 (스키마가 차단) |
| 스레드 | `thread_*` 크로스세션 + `omo thread` CLI | 없음 |
| 데몬/호스트 | `omo daemon …`, `omo host status --all` | 없음 |
| LSP | 직접 툴 6개 + 공유 `lsp-daemon` 아웃프로세스 | MCP 8 alias 인라인 stdio |
| ast-grep | **MCP** (`search`/`rewrite`/`scan`) | 스킬 + `sg` 프로비저닝 |
| 에이전트 상태 위치 | `~/.omo/agent` 단일 정본 강제 | (플러그인이라 강제 없음) |
| 설치 변형 | `~/.omo/agent` 통일, flat 레이아웃 1회 이관 | 최소 침습 |
| 엔진 고정 | **exact pin** `@code-yeongyu/senpi@2026.10.3` | OpenCode 버전에 종속 |

**결론**: OpenCode를 이미 굴리고 있고 에이전트 11종·`team_*` 12개·hashline edit·54~62 훅이 필요하면 Ultimate. 호스트 없이 단일 명령이 필요하거나, 데스크톱을 제어하거나, 스레드/데몬/셸 억지가 필요하면 Native.

### 7.3 Native 설정을 점검하는 법

```bash
omo doctor                 # 루트 진단 (npm 경로: runDoctor / 컴파일 경로: runCompiledDoctor)
omo doctor --reap <pid…>   # stale 엔진 회수 — 명시 pid만, 패턴 킬 금지
omo setup                  # 하네스 감지 · SQLite read-only import · provider 매핑
omo daemon status --all    # rpc.sock 운영자 + p-* 태스크 호스트 + i-* Desktop 스레드 호스트
omo host status --all      # 엔진 inventory (+ tui 행의 last_activity_at)
omo thread report          # 바인딩 상태
```

`omo doctor`에서 **특별히 볼 줄**:

| 줄 접두사 | 의미 |
|---|---|
| `FAIL plugin manifest / extension / lsp-daemon runtime` | 페이로드 3종 중 하나가 없다. 설치가 불완전 |
| `FAIL launch spec: …` | `daemon-launch-spec.json`에 group/world 쓰기 비트. npm/bun umask 문제(#9208). 줄에 수리 명령이 따라온다 |
| `WARN config: <file>: <key> ignored (…)` | 설정 로더가 그 키를 버렸다 |
| `PASS Claude Code <v>: <경로>` | Claude Code가 캐시에서 해석됨(PATH가 아니라면) |
| computer-use `ready` | 엔진 기동 + ABI 핸드셰이크 성공 (`hello` + `capabilities`) |
| computer-use `quarantined` | macOS 다운로드 격리 속성 |
| computer-use `abi-mismatch` | `computer.engine_path`가 다른 릴리스의 엔진 |
| `WARN engine pid … started before this payload was installed` | 이전 페이로드로 도는 중. in-place 재작성은 절대 하지 않는다 |
| `WARN stale engine pid … has no launcher` | 신호로 죽은 런처가 남긴 orphan. `omo doctor --reap <pid>` |
| `INFO memory identities: N durable, M transient` | 메모리 identity 분류 |

컴파일 바이너리에서는 `omo doctor`가 `runCompiledDoctor`를 타고, 아티팩트 경로가 `plugin/…` 3종으로 고정되며, 진단 로더가 `compiled-diagnostic-runtime.ts`의 임베디드 경로로 바뀐다. **사용자용 명령은 동일하다.**

### 7.4 bunshin capability를 쓰는 경우

```bash
# 로컬 엔진(개발 빌드 포함)으로 설치
bun script/install-bunshin-desktop-capability.mjs

# 특정 바이너리 지정
bun script/install-bunshin-desktop-capability.mjs --engine ~/.omo/engines/senpi-desktop-engine/<ver>/senpi-desktop-engine

# 제거
bun script/install-bunshin-desktop-capability.mjs --remove
```

설치 후 출력되는 `BUNSHIN_CAPABILITY_DIR=...` 리마인더대로 bunshin 에이전트를 그 환경변수로 띄워야 descriptor가 로드된다. 그 전까지는 파일이 있어도 아무 일도 일어나지 않는다. 그리고 descriptor 설치는 **데스크톱 권한 부여가 아니다** — 스톱 코드와 감사 경로만 준비된 상태다.

---

## 8. 최종 표 — capability → native / opencode-plugin / codex

| Capability | native | opencode-plugin | codex |
|---|:--:|:--:|:--:|
| 프로세스 소유 | ✅ 자체 프로세스 | ➖ 호스트 프로세스 | ➖ 호스트 프로세스 |
| 호스트 선행 설치 불필요 | ✅ | ❌ | ❌ (Codex CLI 필요) |
| 단일 바이너리 배포 | ✅ `omo-<os>-<arch>` | ➖ npm dist | ➖ npm `lazycodex-ai` |
| 엔진 exact pin | ✅ `@code-yeongyu/senpi@2026.10.3` | ➖ | ➖ |
| 데스크톱 `computer.*` | ✅ 12/12 | ❌ 스키마 차단 | ❌ 스키마 차단 |
| Rust 데스크톱 엔진 (`crates/senpi-desktop-*`, 백엔드 6종) | ✅ | ❌ | ❌ |
| 엔진 ABI 핸드셰이크 | ✅ `hello{protocolVersion,engineVersion,buildSha,abi}` | ❌ | ❌ |
| 스톱 코드 / OS 권한 제어 | ✅ | ❌ | ❌ |
| 자기 엔진 진단 | ✅ doctor + `--reap` | ❌ (호스트 설치만) | ❌ |
| stale/retired 엔진 감독 | ✅ | ❌ | ❌ |
| 세션 격리 task 호스트 (`p-*.sock`) | ✅ | ➖ | ❌ |
| 운영자 데몬 (`rpc.sock`) | ✅ `omo daemon` | ❌ | ❌ |
| Desktop 스레드 호스트 (`i-*`) | ✅ | ❌ | ❌ |
| `omo thread` 크로스세션 CLI | ✅ 12 서브커맨드 | ❌ | ❌ |
| 외부 런타임에 capability 내보내기 (bunshin sidecar) | ✅ | ❌ | ❌ |
| 에이전트 레지스트리 (11종) | ❌ (Senpi 자체 모델) | ✅ | ✅ agent TOML 12 |
| `team_*` 툴 | ✅ 6 (lead 전용) | ✅ 12 | ❌ |
| 훅 | 25 컴포넌트 | ✅ 5-tier 54~62 | 23 JSON (7 이벤트) |
| Hashline edit | ❌ | ✅ | ❌ |
| delegation 카테고리 9종 명시 모델 | ❌ (senpi-task 리졸버) | ✅ | ❌ |
| LSP | 직접 툴 6 + 공유 데몬 | ✅ MCP 8 alias | MCP |
| ast-grep | ✅ MCP | 스킬 + `sg` | 스킬 + `sg` |
| 내장 MCP | 2 (`context7`, `grep_app`) | ✅ 4 | ✅ 4 |
| 설치가 기존 홈을 변형 | ✅ `~/.omo/agent` 통일 강제 | ➖ | `~/.codex/` 변형 |
| 최소 침습 | ❌ | ✅ | ✅ |

---

## 9. 이 문서의 검증 상태 — 확인된 것과 정정·미확인 것

이 문서를 쓰면서 **직접 확인한 것과, 확인하지 못한 것을 분리**한다.

### 집계 수치에 대한 정직한 단서

지정된 기준 수치(`omo-native` 2,633 LoC / 27 파일, `omo-senpi` 122,133 / 879, `senpi-task` 53,551)와 직접 집계한 값이 **일치하지 않는다.** 이 문서는 지정 수치를 본문의 기준으로 삼았으되, 재현 가능한 집계 명령과 그 결과를 함께 남긴다:

| 패키지 | 지정 수치 | 직접 집계 (`.ts`/`.js`/`.mjs`, `node_modules`·`dist` 제외) |
|---|---|---|
| `senpi-task` | 53,551 | `src`에서 `*.test.*` 제외 → **53,439 / 472** (가장 가까움) |
| `omo-senpi` | 122,133 | 패키지 전체에서 `*.test.*` 제외 → **117,614 / 856** / `src`만 → 70,855 / 596 |
| `omo-native` | 2,633 / 27 | 패키지 전체 → 23,511 / 161 / `*.test.*` 제외 → 9,496 / 83 / 최상위 `*.ts` 17개 → **1,327** |

**결론**: `omo-senpi`의 122,133은 "테스트 제외" 계열의 집계와 방향이 맞고, `omo-native`의 2,633/27은 **재현하지 못했다.** 이 패키지는 최상위 `.ts`가 1,327줄에 불과한 반면 `bin/lib/`에 JS 모듈이 55개, `test/`에 테스트 파일이 86개 있다. 2,633/27이 어떤 범위인지 이번 패스에서 특정할 수 없다. **집계 규칙이 다른 수치를 비교하지 말 것.**

### 확인 못 한 것 (추정하지 않고 남긴다)

1. **`@code-yeongyu/senpi`의 내부 구현 전부.** 세션 수명주기, TUI, 모델 루프, RPC 데몬의 실제 코드는 이 레포에 없다. `AGENTS.md`의 서술은 **계약 문서**다.
2. **`bunshin` 자체.** sidecar descriptor 계약과 `machine-sdk`의 검증 규칙 복사본(커밋 `0a21e2c1`)만 있고, bunshin 에이전트·권한 부여·startup 적재 로직은 전부 외부다.
3. **`changes.md`의 소비자.** 런처가 읽는 changelog는 `plugin/CHANGELOG.md`이고, `changes.md`를 `bin/`에서 참조하는 코드는 찾지 못했다.
4. **`browser-bridge`·`formatter`·`post-mutation` 컴포넌트의 등록 상태.** `src/components/`는 31개 디렉터리, `AGENTS.md`가 열거하는 등록 컴포넌트는 25개. 그 차이의 나머지를 확인하지 못했다(`claude-code`는 `compile-entry.ts`가 직접 import 함을 확인).
5. **`provisioned-handoff.ts`의 분기 로직 상세.** `planProvisionedLaunch`/`handOffToProvisionedRuntime`이라는 API 표면과 호출 위치(`compile-entry.ts`)까지는 확인했고, 분기 조건의 완전한 목록은 읽지 않았다.
6. **Rust 크레이트 LoC는 직접 집계값**(`.rs` 기준)이며, 지시된 TS 패키지 LoC와 측정 규칙이 다르다.
7. **`crates/desktop/`는 `.rs`가 0줄**로 측정됐다. 빈 디렉터리인지, 다른 언어로 되어 있는지는 확인하지 않았다.

### 확실히 정정한 것

| 흔한 오해 | 실제 |
|---|---|
| "Native는 런타임이다" | 런처(2.6k) + 외부 엔진(`@code-yeongyu/senpi`) + 레포 측 확장(`omo-senpi` 122k) 3층이다. **프로세스 소유 층은 레포 밖에 있다** |
| "`@code-yeongyu/senpi`가 이 레포에 포함된다" | ❌ npm 레지스트리 패키지. `bun.lock`에 sha512 무결성 해시와 함께 있고, `package.json`이 exact pin. `re-export-shim-inventory.md`에도 senpi 언급 0건 |
| "`senpi-desktop-engine`가 데스크톱 엔진이다" | ❌ TS 측 로케이터 + ABI 핸드셰이크 + 개발 빌드 폴백. **실제 엔진은 `crates/senpi-desktop-engine`의 Rust 바이너리** |
| "`--stdio`/`--serve`/`--oneshot`/`--mcp`는 TS 패키지의 모드다" | ❌ `crates/senpi-desktop-engine/src/cli.rs`의 `ArgGroup` 플래그 |
| "설치 가능한 capability를 설치하면 데스크톱 권한이 열린다" | ❌ descriptor는 `sandbox.grant: "full_access"`를 **요청**할 뿐이다. 설치는 JSON 파일 하나를 쓰는 것이고, 판단은 bunshin이 한다 |
| "bunshin에 에이전트 스톱 해제(`resume`)가 노출된다" | ❌ `stopPath.resume`은 이 브리지를 절대 지나지 않는다. 스톱만 노출 |