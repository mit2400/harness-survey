# OmO 훅 시스템 · 룰 엔진 · AGENTS.md 주입

> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` · 버전 `5.1.13` · 2026-10-04
> 모든 수치는 해당 커밋의 실제 디렉터리 listing / grep 기준. 단정할 수 없는 값은 명시.
> 상위 문서: [harnesses/oh-my-openagent.md](oh-my-openagent.md) · 교차 비교: [topics/hooks.md](../topics/hooks.md)

## 1. 훅 규모와 5-tier 구성

OmO의 훅은 `packages/omo-opencode/src/hooks/`에 있다. 실측 기준:

| 항목 | 값 | 근거 |
|---|---|---|
| 하위 디렉터리 | **62개** | `find -maxdepth 1 -type d` 실측 (루트 `hooks/` 포함) |
| `.ts` 파일 | **643개** | 디렉터리별 `*.ts` 카운트 합산 (테스트·mock 포함) |
| 컴포저 파일 | 6개 | `src/plugin/hooks/create-{core,session,tool-guard,transform,continuation,skill}-hooks.ts` |
| OpenCode 핸들러 | **14개** | 12개 `plugin-interface.ts` + 2개 `src/testing/create-plugin-module.ts` |
| 규칙 문서 | `hooks/AGENTS.md` | "Generated: 2026-07-17" 으로 커밋보다 오래됨 |

가장 무거운 훅 디렉터리(파일 수): `runtime-fallback`(61) · `atlas`(60) · `ralph-loop`(51, 미배선) · `rules-injector`(40) · `anthropic-context-window-limit-recovery`(38) · `todo-continuation-enforcer`(35) · `claude-code-hooks`(31).

### Tier → 이벤트 → 개수

Tier 이름과 소속은 `hooks/AGENTS.md`의 Tier 표와 각 `create-*-hooks.ts`의 실제 반환 객체 키로 대조했다.

| Tier | 컴포저 | 슬롯 수 | 기본 활성 | 이벤트 | 대표 훅 |
|---|---|---|---|---|---|
| **Session** | `create-session-hooks.ts` (264줄) | 24 | 24 | `session.idle`/`created`/`error`, `chat.params`, `chat.message`, `tool.execute.after`, `event` | `preemptiveCompaction`, `sessionNotification`, `thinkMode`, `modelFallback`, `goal`, `prometheusMdOnly`, `runtimeFallback`, `legacyPluginToast` |
| **Tool Guard** | `create-tool-guard-hooks.ts` (180줄) | **18 (확정)** | 17 | `tool.execute.before` / `tool.execute.after` | `commentChecker`, `writeExistingFileGuard`, `bashFileReadGuard`, `rulesInjector`, `hashlineReadEnhancer`, `planFormatValidator`, `teamToolGating` |
| **Transform** | `create-transform-hooks.ts` (123줄) | **8 (확정)** | 5 | `experimental.chat.messages.transform` | `claudeCodeHooks`, `keywordDetector`, `contextInjectorMessagesTransform`, `btwSideContextInjector`, `toolPairValidator` (+team 2, +monitor 1) |
| **Continuation** | `create-continuation-hooks.ts` (98줄) | **7 (확정)** | 7 | `session.idle`, `session.compacted`, `event`, `chat.message` | `todoContinuationEnforcer`(boulder), `compactionContextInjector`, `compactionTodoPreserver`, `atlasHook` |
| **Skill** | `create-skill-hooks.ts` (50줄) | **2 (확정)** | 2 | `chat.message` | `categorySkillReminder`, `autoSlashCommand` |
| **직접 이벤트 핸들러** | `src/plugin/event.ts` (티어 아님) | +4 | 0 | `event` | `team-idle-wake-hint`, `team-lead-orphan-handler`, `team-member-error-handler`, `team-member-status-handler` |
| | | **59 슬롯** | **54 활성** | | team-mode 61 / +monitor 62 |

Tool Guard 18과 Transform 8은 파일을 직접 읽어 반환 객체 키를 세어 **확정**했다. `create-tool-guard-hooks.ts:33-52`의 `ToolGuardHooks` 타입과 `160-179`의 반환 객체가 정확히 18개 키다.

### 문서와 코드의 불일치 (실측)

`hooks/AGENTS.md`는 Tier 3을 "4 base"라고 쓰지만 반환 객체는 8키고, 그중 `btwSideContextInjector`는 `features/btw-side`에서 옵니다(`hooks/` 하위가 아님). 루트 `AGENTS.md`도 이를 명시한다: "the Transform tier also pulls `btwSideContextInjector` … and `contextInjectorMessagesTransform` … (neither is a `hooks/` dir)". 즉 **기본 비team Transform은 5개**이며 `hooks/AGENTS.md`의 "4"는 `hooks/` 디렉터리만 세어低估한 값이다.

Session tier에서 세 문서가 서로 다르게 말한다.

| 출처 | Session 개수 | 문제 |
|---|---|---|
| 코드 `create-session-hooks.ts` | **24** | `isHookEnabled("…")` 훅 이름 24개 (`create-session-hooks.ts:84`~`234`) |
| `hooks/AGENTS.md` 헤더 | 24 | 일치 |
| `hooks/AGENTS.md` Tier 1 표 행 수 | 23 | `native-edition-nudge` 누락 |
| `src/AGENTS.md` | 23 | `native-edition-nudge` 누락 |
| 루트 `AGENTS.md` | 24 | 일치 |

`hooks/AGENTS.md`는 "60 directories"라 쓰지만 실제는 62개다. 이 문서는 2026-07-17 생성본이라 커밋 `251cbfe`에 뒤처진 것으로 보인다.

`isHookEnabled("startup-toast")`(`create-session-hooks.ts:139`)는 독립 훅이 아니라 `autoUpdateChecker`에 넘기는 서브 플래그다. 슬롯 계산에서 제외해야 한다.

"54 활성" 산술도 문서 안에서 일관되지 않는다. 루트 `AGENTS.md`는 config-gated null을 8개 열거(|team 3 + monitor 1 + goal + model-fallback + preemptive-compaction + interactive-bash|)하면서도 59−**5**=54로 계산한다. `goal`/`model-fallback`/`preemptive-compaction`의 실제 기본값은 이 문서만으로 확정할 수 없다. **54는 문서 주장값이고 코드에서 재현하지 못했다.**

### 미배선 훅 2개

| 디렉터리 | 상태 |
|---|---|
| `task-reminder/` | `index.ts` + `createTaskReminderHook` 존재. barrel 미export, 어떤 컴포저도 import 안 함 |
| `ralph-loop/` | barrel에는 남았으나 컴포저 미배선. `goal/`가 대체(PR #6184). `ralph_loop` 설정은 deprecated passthrough |

둘 다 "orphaned, 배선 전까지 건드리지 말라"는 상태. `hooks/AGENTS.md` 기준 62개 디렉터리 중 `shared/`·`team-session-events/`·`zauc-mocks-*` 4개가 `index.ts` 없는 비훅 디렉터리다. `zauc-*` 4개는 `bun:test`가 알파벳순으로 먼저 로드되도록 이름 접두어를 붙인 테스트 목업이다.

## 2. Tier 합성 방식과 플러그인 API 경계

### 합성 트리

`packages/omo-opencode/src/create-hooks.ts`(103줄)가 최상위 진입점이다.

```
createHooks()                                     # src/create-hooks.ts
  ├─ createCoreHooks()                            # src/plugin/hooks/create-core-hooks.ts (55줄)
  │    ├─ createSessionHooks()                    #   24
  │    ├─ createToolGuardHooks()                  #   18
  │    └─ createTransformHooks()                  #    8
  ├─ createContinuationHooks()                    #    7
  └─ createSkillHooks()                           #    2
  ⇒ {...core, ...continuation, ...skill} + disposeHooks()
```

핵심 성질 3가지:

1. **티어 간 연결은 없다.** `createCoreHooks()`가 Session+ToolGuard+Transform을 호출해 단순히 `{...session, ...tool, ...transform}`로 펼친다. 이후 `create-hooks.ts`가 core/continuation/skill 3개를 다시 펼쳐 **단일 평면 객체**를 만든다. 반환 타입 `CreatedHooks = ReturnType<typeof createHooks>`라 키 이름이 곧 전역 유일 식별자다.
2. **객체 키 순서가 곧 호출 순서다.** `hooks/AGENTS.md` NOTES: "within Session tier the registration order in `create-session-hooks.ts` determines invocation order — earlier hooks see un-mutated input, later hooks see accumulated output." 핸들러(`plugin/tool-execute-before.ts` 등)는 이 레코드를 순회하며 `(input, output)`로 호출한다.
3. **모든 훅은 `createXXXHook(deps) -> HookFunction` 팩토리.** 비활성 시 `null`을 넣고, 생성 실패는 `safeCreateHook(hookName, factory, { enabled: safeHookEnabled })`가 예외를 격리한다. 즉 훅 하나가 죽어도 체인은 산다.

### 슬롯 게이트 (기본값 null이 되는 조건)

| 게이트 | 위치 |
|---|---|
| `isHookEnabled("team-tool-gating")` | `create-tool-guard-hooks.ts:144` |
| `isHookEnabled("team-mode-status-injector")` / `team-mailbox-injector` | `create-transform-hooks.ts:118-119` |
| `monitorConfig?.enabled && monitorManager && isHookEnabled("monitor-status-injector")` | `create-transform-hooks.ts:105` |
| `isHookEnabled("goal") && pluginConfig.goal?.enabled` | `create-session-hooks.ts:164` |
| `isModelFallbackConfigEnabled && isHookEnabled("model-fallback")` | `create-session-hooks.ts:111` |
| `isHookEnabled("preemptive-compaction") && …` | `create-session-hooks.ts:84` |
| `isHookEnabled("interactive-bash-session") && …` | `create-session-hooks.ts:159` |
| `isHookEnabled("rules-injector")`, 단 `claude_code.hooks === false`면 `skipClaudeUserRules` | `create-tool-guard-hooks.ts:102-109` |
| OpenCode 네이티브 지원 시 자동 비활성 | `create-tool-guard-hooks.ts:78-91` |

마지막 항목이 독특하다. `directory-agents-injector`는 `getOpenCodeVersion()`이 `OPENCODE_NATIVE_AGENTS_INJECTION_VERSION` 이상이면 **자기 자신의 로그를 남기고 null이 된다**. "호스트 버전에 따라 OmO 훅이 사라지는" 유일한 케이스다.

### OpenCode API 경계: 14개 핸들러

`packages/omo-opencode/src/plugin-interface.ts`가 **유일하게** OpenCode `Plugin` API를 만지는 파일이다(110줄). 반환 객체의 키 12개:

| 핸들러 키 | 위임 대상 |
|---|---|
| `tool` | `tools` (ToolRegistry 레코드) |
| `chat.params` | `applyAgentVariant()` + `createChatParamsHandler` |
| `chat.headers` | `createChatHeadersHandler` |
| `command.execute.before` | `createCommandExecuteBeforeHandler({ directory, hooks })` |
| `chat.message` | `createChatMessageHandler({ ctx, pluginConfig, firstMessageVariantGate, hooks })` |
| `experimental.chat.messages.transform` | `createMessagesTransformHandler({ hooks })` |
| `experimental.chat.system.transform` | `createSystemTransformHandler(default_mode, getUltraworkMessage, hooks.keywordDetector)` |
| `config` | `managers.configHandler` |
| `event` | `createEventHandler({ ctx, pluginConfig, firstMessageVariantGate, managers, hooks })` |
| `tool.definition` | `createToolDefinitionHandler({ hooks })` |
| `tool.execute.before` | `createToolExecuteBeforeHandler({ ctx, hooks, backgroundManager })` |
| `tool.execute.after` | `createToolExecuteAfterHandler({ ctx, hooks })` |

`src/testing/create-plugin-module.ts`가 추가로 2개를 직접 배선한다: `experimental.session.compacting`, `experimental.compaction.autocontinue`. 합계 14.

즉 **OmO의 59개 훅 슬롯은 14개의 OpenCode 훅 포인트 위에 겹쳐진 것**이다. 티어 이름(Session/Tool Guard/Transform/…)은 OmO 내부 분류이지 OpenCode API가 아니다. `chat.message` 하나에 Session tier 12개 + Continuation 1개 + Skill 2개가 몰려 있다.

`src/plugin/event.ts`는 티어 컴포저를 거치지 않고 `team-session-events/` 4개 핸들러를 직접 등록한다(팀 모드 활성 시).

## 3. 개별 훅 해설

### 컨텍스트/규칙 주입 계열

| 훅 | 하는 일 / 있는 이유 |
|---|---|
| `rules-injector` | `.omo/rules` 등 룰 파일을 매칭해 툴 결과 뒤에 주입. 아키텍처 규칙(`test-discipline.md` 등)을 에이전트가 매번 리마인드받게 해 강제한다 |
| `keyword-detector` (IntentGate) | 사용자 의도를 `ultrawork`/`ulw`, `search`, `analyze`, `team`으로 분류하고 모드별 프롬프트 주입. `ulw` 키워드 → `senpi-task/src/dag/` DAG 실행 트리거 |
| `compaction-context-injector` | `session.compacted` 직후 컨텍스트 재주입. 압축이 AGENTS.md/방향 컨텍스트를 삼키는 것을 막는다 |
| `compaction-todo-preserver` | 압축을 넘어 todo 목록 유지. 미완료 작업 상태가 증발하면 boulder 루프가 끊긴다 |
| `directory-agents-injector` | 디렉터리 로컬 `AGENTS.md`를 디렉터리 접근 시점에 주입. 호스트가 네이티브 지원하면 자동 정지 |
| `directory-readme-injector` | 디렉터리 로컬 `README.md`를 같은 방식으로 주입 |
| `hephaestus-agents-md-injector` | Hephaestus 깊은 작업 세션 전용 walk-up AGENTS.md 주입 (`chat.message`) |
| `contextInjectorMessagesTransform` | Transform 티어의 범용 메시지 컨텍스트 주입기 |
| `btwSideContextInjector` | `features/btw-side`의 임시 사이드 대화 컨텍스트 주입 |
| `categorySkillReminder` | 카테고리 호출 전 스킬 로드를 유도 |
| `team-mailbox-injector` | 팀 모드: 대기 중인 메일박스 메시지를 컨텍스트로 흡수 |

### 가드/차단 계열 (사용자 워크플로를 실제로 막는다)

| 훅 | 하는 일 / 있는 이유 |
|---|---|
| `write-existing-file-guard` | 기존 파일 Write/Edit 전 Read를 강제. 레포 `AGENTS.md`의 "Never write to existing files without reading them first"를 기계화한다 |
| `notepad-write-guard` | `Write`를 append 전용 notepad 경로(`.omo/notepads`, `.sisyphus/notepads`)에 차단. 오염 방지 |
| `prometheus-md-only` | Prometheus 에이전트의 쓰기를 `.md`로 제한. 금지 경로 = `packages/*/src/`, `package.json`, 설정 파일 |
| `bash-file-read-guard` | `cat`/`head`/`tail` 등 파일 읽기 bash 명령을 감시. Read 툴로 유도해 출력을 추적·절감 가능하게 만든다 |
| `comment-checker` | AI slop 주석 패턴 차단. 외부 바이너리 `@code-yeongyu/comment-checker`. `// @allow`(한 줄), `// comment-checker-disable-file`(파일 전체)로 우회 |
| `plan-format-validator` | boulder 계획 파일의 `Write`/`Edit`에서 체크박스 포맷 검증. 계획 문서가 기계 파싱 가능해야 boulder가 진행 가능 |
| `tasks-todowrite-disabler` | Sisyphus task 시스템 활성 시 네이티브 `TodoWrite` 비활성. 이중 todo 상태를 막는다 |
| `tool-pair-validator` | tool_call / tool_result 페어링 검증. Transform 티어라 모델이 만든 불완전 페어를 조기에 잡는다 |
| `webfetch-redirect-guard` | webfetch 리다이렉트 동작 감시 |
| `toolOutputTruncator` | 과도한 툴 출력 절단. 컨텍스트 비용의 1차 방어선 |
| `todo-description-override` | todo 항목 설명 덮어쓰기(`tool.definition`에도 적용) |
| `noSisyphusGpt` / `noHephaestusNonGpt` | Sisyphus/Hephaestus를 비-GPT 프로바이더에서 차단(+경고 토스트). 모델 요구사항 강제 |
| `commentChecker` 리소스 해제 | `disposeCreatedHooks()`가 7개 훅의 `dispose()`를 호출 (`create-hooks.ts:28-37`) |

### 복구/폴백 계열

| 훅 | 하는 일 / 있는 이유 |
|---|---|
| `model-fallback` | **선제적**. `chat.params`에서 프로바이더 레벨 모델 체인 교체. `shared/model-requirements.ts`의 에이전트별 체인. 설정 `model_fallback` 게이트 |
| `runtime-fallback` | **반응적**. `event`(세션 에러)에서 API 프로바이더 오류 감지 시 자동 전환. 카테고리/에이전트별 설정 가능. 훅 61파일 중 최대 규모 + `first-prompt-watchdog.ts`(90초 무진행 서브에이전트 탐지) |
| `edit-error-recovery` | 실패한 파일 편집 재시도 (`tool.execute.after`) |
| `delegate-task-retry` | 실패한 태스크 위임 재시도 (`tool.execute.after`) |
| `json-error-recovery` | JSON 파싱 오류 탐지 + 교정 리마인더 주입 |
| `anthropic-context-window-limit-recovery` | Anthropic 컨텍스트 한도 초과 시 다중 전략(절단·압축·중복 제거). 38파일 |
| `delegate-task-retry` vs `edit-error-recovery` | 둘 다 `tool.execute.after`지만 대상이 다름. 하나는 툴 산출물, 하나는 위임 결과 |

두 폴백은 **의도적으로 독립**이며 상호 통합이 없다(루트 `AGENTS.md` ARCHITECTURE INVARIANTS 명시). 하나는 요청 시점, 다른 하나는 실패 시점.

### 연속 실행/압축/알림 계열

| 훅 | 하는 일 / 있는 이유 |
|---|---|
| `todo-continuation-enforcer` | **boulder**. `session.idle`에서 미완료 todo가 있으면 강제 연속 실행. 계획이 멈추지 않게 하는 핵심 장치 |
| `atlasHook` | boulder/ralph/서브에이전트 세션의 마스터 오케스트레이터. `todoContinuationEnforcer`와 **둘 다 `session.idle`에 걸리지만 세션 타입을 먼저 확인**해 분기한다 |
| `preemptive-compaction` | 한도 도달 전 선제 압축. 파일이 `preemptive-compaction.ts` + `-trigger` + `-types` + `-degradation-monitor` + `-no-text-tail`로 분할 |
| `session-notification` | OS 알림. **20개 최상위 파일**로 쪼개짐: `-linux`/`-macos`/`-windows`/`-desktop-sidecar`/`-platform`/`-sound`/`-sender`/`-scheduler`/`-runner`/`-formatting`/`-content`/`-log`/`-init`/`-utils`/`-event-properties` + 테스트 |
| `hashline-read-enhancer` | 모든 `Read` 출력에 `LINE#ID` 내용 해시 태그. `hashline_edit`가 적용 전 해시를 검증하고 불일치 시 거부 (`ZPMQVRWSNKTXJBYH` 알파벳, xxHash32) |
| `readImageResizer` | 큰 이미지 리사이즈로 컨텍스트 절감 |
| `goal` | 세션별 지속 목표 + idle 연속 실행 + 사용량 계량. `ralphLoop` 대체 |
| `stop-continuation-guard` | `/stop-continuation` 커맨드 핸들러 |
| `ulwExecute` | `/ulw-execute` 커맨드 핸들러 (`chat.message`) |
| `autoSlashCommand` | 사용자 메시지에서 매칭되는 `/command` 자동 실행 |
| `agentUsageReminder` / `taskResumeInfo` / `interactiveBashSession` / `nonInteractiveEnv` / `thinkMode` / `autoUpdateChecker` / `astGrepSgProvision` / `questionLabelTruncator` / `sisyphusJuniorNotepad` / `legacyPluginToast` / `unstableAgentBabysitter` / `backgroundNotificationHook` / `claudeCodeHooks` | 각기 에이전트 리마인더, resume 컨텍스트, tmux 세션 수명주기, `run` 명령 조정, 모델 variant 전환, npm 업데이트 확인, `sg` 바이너리 프로비저닝, 긴 라벨 절단, 서브에이전트 노트패드 주입, 레거시 플러그인 경고, 불안정 에이전트 감시, 백그라운드 완료 알림, Claude Code `settings.json` 호환 |

## 4. rules-engine

`packages/rules-engine/` — **3,054 LoC** (테스트 제외 실측). `rules-core`에서 개명된 패키지.

### 파일 구조: 신형 엔진 + 구형 shim

```
rules-engine/
├── index.d.ts / engine.d.ts        # 1줄 stub
└── src/
    ├── engine/                     # 현행 구현 27파일
    │   ├── constants.ts  finder*.ts  scanner.ts  parser*.ts
    │   ├── matcher.ts  distance.ts  ordering.ts  formatter.ts  truncator.ts
    │   ├── engine.ts  engine-{static,dynamic}-{loader,cache}.ts
    │   ├── cache.ts  types.ts  errors.ts  sources.ts  project-root.ts
    └── (11개 평면 re-export shim)   # finder.ts matcher.ts ordering.ts
                                    # parser.ts scanner.ts project-root.ts
                                    # types.ts index.ts constants.ts cache.ts
                                    # distance.ts agents-md.ts
```

평면 shim 11개는 하네스 중립화를 위한 재export다. 루트 `AGENTS.md`가 금지하는 "catch-all 파일"(`utils.ts`/`helpers.ts`/`service.ts`) ban의 예외이자, 동시에 리팩터링 잔재다. `src/agents-md.ts`가 §5의 walk-up 알고리즘을 실제로 들고 있다.

### 룰 발견 경로 (코드 확인)

`src/engine/constants.ts`:

| 분류 | 상수 | 값 |
|---|---|---|
| 프로젝트 루트 마커 | `PROJECT_MARKERS` | `.git`, `pnpm-workspace.yaml`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `.venv` |
| 프로젝트 룰 서브디렉터리 | `PROJECT_RULE_SUBDIRS` | `.omo/rules`, `.claude/rules`, `.cursor/rules`, `.github/instructions` |
| 프로젝트 단일 파일 | `PROJECT_SINGLE_FILES` | `.github/copilot-instructions.md`, `CONTEXT.md` |
| 사용자 홈 서브디렉터리 | `USER_HOME_RULE_SUBDIRS` | `.omo/rules`, `.opencode/rules`, `.claude/rules` |
| 번들 플러그인 | `BUNDLED_RULE_SUBDIR` | `bundled-rules` |
| 확장자 | `RULE_FILE_EXTENSIONS` | `.md`, `.mdc` |

사용자가 가정한 4개 경로(`.omo/rules`, `.claude/rules`, `.cursor/rules`, `.github/instructions`)는 **전부 코드에 존재**한다. 그 외에 코드에만 있는 것이 4개 더 있다: `.opencode/rules`(사용자 홈), `.github/copilot-instructions.md`, `CONTEXT.md`, `plugin-bundled`.

`src/engine/finder-sources.ts`는 위 목록에 없는 소스를 `UnsupportedRuleSourceError`로 던지는 타입 가드다. 즉 경로 목록은 상수 배열이 아니라 타입 유니온(`RuleSource`)과 이 스위치문이 함께 고정한다.

### 우선순위

`SOURCE_PRIORITY` (낮을수록 먼저, `src/engine/constants.ts:56-68`):

| 값 | 소스 |
|---|---|
| 0 | `.omo/rules` |
| 1 | `.claude/rules` |
| 2 | `.cursor/rules` |
| 3 | `.github/instructions` |
| 4 | `.github/copilot-instructions.md` |
| 7 | `CONTEXT.md` |
| 100 | `~/.omo/rules` |
| 101 | `~/.opencode/rules` |
| 102 | `~/.claude/rules` |
| 200 | `plugin-bundled` |

5와 6은 비어 있다(삭제된 소스 잔재).

정렬은 `src/engine/ordering.ts`의 `compareCandidates`가 5단계로:

1. `compareBoolean(isGlobal)` — 로컬(false)이 글로벌(true)보다 먼저
2. `compareNumber(distance)` — 근접도. 글로벌은 `GLOBAL_DISTANCE = 9999`
3. `compareNumber(SOURCE_PRIORITY)` — 위 표
4. `compareString(relativePath)`
5. `compareString(realPath)`

최종 동점은 원래 인덱스(stable sort). 즉 **우선순위가 같으면 거리, 거리가 같으면 소스 종류**로 결정되고, 완전히 같으면 스캔 순서를 따른다.

`DEFAULT_AUTO_DISABLED_SOURCES = ["AGENTS.md", "~/.claude/rules", "~/.claude/CLAUDE.md"]` (`engine/sources.ts:4`). OpenCode가 AGENTS.md를 네이티브로 주입하므로 OmO가 중복 주입하지 않는다. `directory-agents-injector`의 버전 게이트와 같은 설계 철학이다.

### 포맷과 파싱

- 프론트매터는 `engine/parser-frontmatter.ts` + `engine/parser-yaml.ts`가 분리 처리
- 단일 파일 소스(`copilot-instructions.md`)는 프론트매터 **선택**(`engine/types.ts:55` 주석)
- 매칭은 `matcher.ts` + `distance.ts`(Glob/경로 유사도)
- 절단은 `truncator.ts`, 마커 `TRUNCATION_NOTICE = "\n\n[Truncated. Full: {path}]"`

문자 예산이 단계별로 갈린다. 이게 실비용을 결정한다.

| 상수 | 값 | 용도 |
|---|---|---|
| `DEFAULT_MAX_RULE_CHARS` | 12,000 | 기본 룰 본문 상한 |
| `DEFAULT_MAX_RESULT_CHARS` | 40,000 | 툴 결과 전체 상한 |
| `DEFAULT_MAX_SCAN_FILES` | 1,000 | 스캔 파일 수 상한 |
| `DEFAULT_POST_COMPACT_MAX_RULE_CHARS` / `_RESULT_CHARS` | 3,500 / 4,000 | 압축 직후 축소 예산 |
| `DEFAULT_DYNAMIC_MAX_RULE_CHARS` / `_RESULT_CHARS` | 4,000 / 10,000 | 세션 중 동적 매칭 (의도적으로 매우 낮음) |
| `DEFAULT_PROMPT_MAX_RULE_CHARS` / `_RESULT_CHARS` | 6,000 / 16,000 | 프롬프트 시점 정적 주입 |
| `SCANNER_EXCLUDED_DIRS` | `node_modules`, `.git`, `dist`, `build`, `.turbo`, `.next`, `coverage` | 재귀 스캔 제외 |

레거시 호환: `src/finder.ts:16`에 `.sisyphus/rules` deprecate 메시지(v4.3.0 제거 예정).

### 3중 캐시

`src/cache.ts`, `engine/cache.ts`, `engine-dynamic-cache.ts`, `engine/finder-cache.ts`. 비교 문서가 "parsed-rule LRU + match decision LRU + scan cache"라고 정리한 것이 이 구조다. `engine-dynamic-cache.ts`는 파일 상위 수집 경로를 캐시한다.

### `docs/reference/rules-injection-cross-module-comparison.md` 대조

이 문서(116줄)는 `codex-rules` / `pi-rules` / `omo rules-injector` 3개 구현의 포팅 기록이다. 커밋 참조가 `fbe423a2d`(omo), `9789f12`(pi), `9f49c68`(codex)라 분석 기준 `251cbfe`보다 오래된 **과거 스냅샷**이다.

| 항목 | 문서 | 코드/현황 |
|---|---|---|
| omo 룰 경로 | `packages/omo-opencode/src/hooks/rules-injector` | 일치. 해당 디렉터리에 `.ts` 40개 |
| 공유 엔진 | "shared `@oh-my-opencode/rules-engine` workspace package" | 일치. omo·pi가 공유하고 codex-rules만 자체 finder 보유 |
| 주입 훅 위치 | "Multiple injection hooks: omo = **tool.execute.after** only" | **불일치.** `hooks/AGENTS.md` Tier 2 표는 `rulesInjector`를 `tool.execute.before`로 등재. 어느 쪽이 맞는지 코드에서 확정하지 못했다 |
| 트랜스크립트 hydrate | `transcript-hydration.ts` (~140 LoC), `createTranscriptHydrationStore({ client })` | 파일 존재 미검증. 문서 값으로만 기재 |
| hydrate 마커 | `[Rule: <relativePath>]\n[Match: <reason>]`, 200 메시지 / 1,000,000자 캡, single-flight | 문서 §5 기준 |
| 동적 타깃 핑거프린팅 | omo는 "deferred", omo는 이미 LRU로 흡수 | `engine-dynamic-cache.ts` 존재하나 핑거프린팅 레이어는 미확인 |
| post-compact 전략 | "clearSessionState(sessionID) (full reset)" | 문서 기록 |
| 테스트 | "rules-injector 70/70 bun test" | 커밋 `fbe423a2d` 시점 값. 현 커밋 값은 미측정 |

문서 자체가 §7에 미해결 항목을 4개 남겨 두었다: omo 동적 핑거프린팅 도입, 공통 벤치마크 하네스, pi-mono에 `transcript_path` 노출, omo 훅 테스트 스위트 크로스 스위트 상태 누수. 문서 마지막 문단에 이 flakiness를 인정한다("the most recent merge on dev is literally named `fix/test-isolation-cross-test-state-leak`").

## 5. agents-md-core: walk-up 알고리즘

`packages/agents-md-core/` — **160 LoC** (테스트 143줄 제외, 실측).

| 파일 | LoC | 역할 |
|---|---|---|
| `injector.ts` | 72 | 주입 실행 |
| `finder.ts` | 26 | 경로 정규화 + 자일(jail) 검사 |
| `types.ts` | 19 | `AgentsMdContextOutput`, `AgentsMdTruncator`, `AgentsMdInjectedPathsStorage` |
| `formatter.ts` | 15 | 컨텍스트 블록 포맷 |
| `injection-cache.ts` | 14 | 세션별 캐시 접근 |
| `index.ts` | 10 | barrel |
| `constants.ts` | 4 | 절단 알림 접두/접미 |
| `injector.test.ts` | 143 | 테스트 |

**중요한 사실: walk-up 알고리즘 자체는 agents-md-core에 없다.** `findAgentsMdUp`은 `packages/rules-engine/src/agents-md.ts`에 있고, agents-md-core는 `@oh-my-opencode/rules-engine`에서 import한다. 즉 AGENTS.md 발견 로직이 룰 엔진에 산다.

### `findAgentsMdUp` (`rules-engine/src/agents-md.ts`)

```
입력: { startDir, rootDir, skipRoot?, cache? }
1. startDir, rootDir 를 realpathSync 로 정규화 (실패 시 resolve 폴백)
2. isSameOrChildPath(startDir, rootDir) 실패 → [] 반환   ← 경로 이탈 차단
3. cacheKey = [startDir, rootDir, skipRoot?"1":"0"].join("\0") → AgentsMdCache 조회
4. current = startDir 에서 루프:
     - skipRoot(기본 true) && current === rootDir 면 이 레벨 건너뜀
     - join(current, AGENTS_FILENAME) → existsSync + realpathSync + isFile + isSameOrChildPath
       중 하나라도 실패면 null (즉 디렉터리는 무시, 심볼릭 링크 이탈도 무시)
     - 루트 도달 / parent === current / parent가 root 밖이면 break
5. found.reverse()   ← 루트에 가까운 것부터
6. cache.set(cacheKey, result)
```

`skipRoot` 기본값이 `true`라는 점이 핵심이다. **루트 AGENTS.md는 이 훅이 주입하지 않는다.** 호스트(OpenCode)가 이미 주입하므로 `DEFAULT_AUTO_DISABLED_SOURCES`에 `AGENTS.md`가 들어 있는 것과 같은 이유다. 결과 배열은 루트→리프 순서로 뒤집혀 **상위 컨텍스트가 먼저** 들어간다.

### `processFilePathForAgentsInjection` (`agents-md-core/src/injector.ts`)

```
1. typeof output.output !== "string" → 즉시 no-op
2. resolveFilePath(rootDirectory, filePath)
     - 상대경로면 root 기준으로 resolve
     - 양쪽 realpathSync 정규화 후 isSameOrChildPath 검사
     - root 밖이면 null → 스킵
3. dir = dirname(resolved)
4. cache = getSessionCache({ sessionCaches, sessionID, storage })
5. agentsPaths = findAgentsMdUp({ startDir: dir, rootDir, cache })
6. 각 path 에 대해:
     - cache.has(dirname(path)) → 이미 주입됨, 스킵
     - fsPromises.readFile(path, "utf-8").catch(() => null) → 실패 시 continue
     - cache.add(dirname(path))
     - truncator.truncate(sessionID, content) → { result, truncated }   ← 주입 의존성
     - output.output += formatAgentsMdContextBlock({ agentsPath, content, truncated })
7. dirty면 storage.saveInjectedPaths(sessionID, cache)
```

포맷(`formatter.ts`):

```
\n\n[Directory Context: ${agentsPath}]\n${content}${truncationNotice}
```

절단 시 `truncationNotice`:

```
\n\n[Note: Content was truncated to save context window space. For full context, please read the file directly: ${agentsPath}]
```

### 성질과 소비처

- **경로 구동 방식이다.** 세션 시작이 아니라 **에이전트가 그 디렉터리 안의 파일을 만났을 때** 발동한다. 실제로 만난 디렉터리 체인만 주입된다.
- 세션당 `Set<dirname>`로 중복 주입을 막고, `storage.saveInjectedPaths`로 지속 저장한다.
- `truncator`와 `storage`는 주입 의존성이라 어댑터가 교체한다(하네스 중립).
- 소비 훅: `directory-agents-injector`(`tool.execute.before`, 버전 게이트로 자동 정지 가능), `hephaestus-agents-md-injector`(`chat.message`), `rules-injector`.

## 6. 비용과 위험

### 매 툴 호출마다 도는 티어

Tool Guard는 18개 슬롯이 `tool.execute.before`와 `tool.execute.after`에 각각 붙는다. 즉 **툴 호출 1회당 최대 36회 훅 실행 기회**가 있고, 지연은 직렬이다(`safeCreateHook` 격리지만 순차 호출). `chat.message`는 Session 12 + Continuation 1 + Skill 2 = 15개가 몰린다.

`safeCreateHook`는 예외를 격리하지만 비용을 없애지 않는다. 각 훅은 호출될 때마다 입력 정규화·경로 해석·캐시 조회를 한다.

### 배치/단락 로직

| 기법 | 위치 | 효과 |
|---|---|---|
| 세션당 `Set` 중복 차단 | `agents-md-core/injector.ts` `cache.has()` | 같은 디렉터리 AGENTS.md 재주입 방지 |
| 3중 LRU 캐시 | `rules-engine/src/engine/cache.ts`, `engine-dynamic-cache.ts`, `finder-cache.ts` | 파싱·매칭·스캔 비용 흡수 |
| 문자 예산 단계 분리 | `engine/constants.ts` | 동적 주입 4,000/10,000으로压制. 세션 중 룰 주입을 의도적으로 얇게 유지 |
| 지속 세션 캐시 + 트랜스크립트 hydrate | 비교 문서 §5 | 재시작/`session.compacted` 후에도 중복 주입 회피. 200 메시지·1MB 캡, transport 오류는 fail-open, 동시 호출 single-flight |
| 출력 절단 | `tool-output-truncator` | 결과 크기 상한 |
| 이미지 리사이즈 | `read-image-resizer` | 비텍스트 컨텍스트 비용 |
| `SCANNER_EXCLUDED_DIRS` | `engine/constants.ts` | `node_modules`/`dist` 등 재귀 스캔 제외 |

비교 문서가 omo에 **포팅하지 않은** 최적화: 동적 타깃 핑거프린팅. "cache miss인데 대상 파일이 안 바뀌었으면 `findRuleFiles` 재탐색을 생략"하는 계층이다. 문서는 "LRU가 이미 대부분 흡수하므로 당일 ROI 낮음"으로 판단해 §7에 미해결로 남겼다. 핑거프린팅 상수가 코드에 없는 것으로 보이지만 미검증이다.

### 사용자 워크플로를 깨뜨릴 수 있는 지점

| 훅 | 위험 |
|---|---|
| `comment-checker` | AI slop 주석으로 판정되면 빌드가 막힌다. `// @allow` / `// comment-checker-disable-file`이 유일한 우회 |
| `prometheus-md-only` | Prometheus가 `packages/*/src/`나 설정 파일을 못 건드린다. 의도된 제약이지만 계획 범위가 넓으면 조용히 막힌다 |
| `write-existing-file-guard` | Read 없이 Edit 시 차단. 세션 재개 후 대상 파일을 안 읽었으면 실패한다 |
| `todo-continuation-enforcer` (boulder) | 미완료 todo가 있으면 `session.idle`에서 강제 재시작. 종료 조건이 잘못 잡히면 무한 반복 + 예상치 못한 지출 |
| `preemptive-compaction` | 세션 도중 컨텍스트를 강제로 압축. 진행 중 작업이 증발할 수 있다 |
| `noSisyphusGpt` / `noHephaestusNonGpt` | 비-GPT 프로바이더 선택 시 Sisyphus/Hephaestus가 **하드 실패**한다. 실수로 다른 프로바이더를 고르면 작업이 시작도 못 한다 |
| `model-fallback` / `runtime-fallback` | 모델이 조용히 바뀐다. `runtime-fallback`는 에러 뒤 자동 전환이라 사용자가 모르는 사이 비용·품질이 변한다. 두 시스템은 독립이라 서로의 상태를 모른다 |
| `directory-agents-injector` | 호스트 버전이 기준선을 넘으면 스스로 비활성화. **호스트를 업그레이드하면 동작이 조용히 바뀐다** |
| `interactive-bash-session` | tmux 없으면 비활성. TUI 조작 요구가 있는 흐름이 못 동작 |
| `session-notification` | OS별 분기가 14개 파일. Linux 알림 미들없이는 무음 |
| `notepad-write-guard` | `.omo/notepads` 아래 쓰기 차단. 경로 규칙을 모르고 파일을 만들면 막힌다 |
| `plan-format-validator` | boulder 계획 체크박스 포맷 오류로 `Write`/`Edit` 자체가 거부된다 |
| `bash-file-read-guard` | `cat`/`head`/`tail`을 막으므로 셸 스크립트에 의존하는 기존 워크플로가 깨진다 |
| `tasks-todowrite-disabler` | Sisyphus task 시스템이 켜져 있으면 네이티브 `TodoWrite`가 사라진다 |

`safeCreateHook`가 예외를 삼켜주므로 **크래시 대신 무음 실패**가 된다. 훅이 조용히 비활성화된 걸 사용자가 알기 어렵다.

## 7. 다른 하니스 대비

`topics/hooks.md` 기준 8개 하니스 훅 규모:

| 하니스 | 훅 규모 | 형태 | OmO 대비 |
|---|---|---|---|
| claude-code | 32 클래식 + ~90 function | `hooks.json` + TS 모듈 | 표면은 최대. 단 바이너리 폐쇄라 내부 검증 불가 |
| **oh-my-openagent** | **59 슬롯 / 54 활성 (5-tier)** | OpenCode 플러그인 + Codex/Senpi 어댑터 | — |
| hermes-agent | 4종 ~50 `VALID_HOOKS` | `plugin.yaml` + shell subprocess | 개수는 비슷. hermes는 호출당 subprocess 비용, OmO는 인프로세스 클로저 |
| openclaw | 3종 분리, typed ~40 | `openclaw.plugin.json` + HTTP webhook | OmO는 webhook 없음. 팀 모드가 그 자리를 대신 |
| oh-my-pi | 40+ | Extension = Hook 통합 | marketplace·외부 16포맷 인식 |
| pi-mono | 32 | TS 모듈 (jiti) | OmO는 replaceable 개념 없음 |
| opencode | 20 hook 포인트 + 27 이벤트 | TS 모듈 | 오모가 OpenCode 위에 59개를 얹는 구조 |
| codex | 12 | `config.toml`/`hooks.json` | 가장 작음. MCP 기반 훅 + trust hash |

단순 개수만 보면 OmO는 claude-code·hermes·openclaw과 같은 3rd 티어다. 차이가 있는 지점은 세 군데다.

**첫째, 내부 합성 규범.** 8개 중 유일하게 훅을 **5-tier 컴포저 + 단일 플러그인 경계**로 구조화했다. `createHooks()` → `createCoreHooks()` → 티어별, 그리고 59개 슬롯이 정확히 14개 OpenCode 훅 포인트 위로 접힌다. 다른 하니스는 훅을 플랫한 레지스트리로 노출한다. `hooks/AGENTS.md`가 티어·슬롯 수·와이어 위치를 문서화하고, 루트 `AGENTS.md`는 `disabled_hooks` allowlist(`config/schema/hooks.ts` `HookNameSchema`)를Tier 문서와 동기화하라고 강제한다.

**둘째, 훅 외 계층을 별도 코어 패키지로 뽑았다.** `rules-engine`(3,054 LoC), `agents-md-core`(160), `comment-checker-core`, `hashline-core`, `boulder-state`, `memory-core`가 하네스 중립 `*-core` 20개에 속한다. 같은 룰 엔진이 OpenCode 어댑터·Codex 플러그인(`components/rules`)·pi-rules 확장 3곳에서 재사용된다. 비교 문서의 "shared `@oh-my-opencode/rules-engine`" 행이 이 구조다. 다른 하니스의 훅은 자기 하네스 안에 갇혀 있다(claude-code의 rules는 Claude Code용 `.claude/rules`에 묶여 있다).

**셋째, 실패 모델.** 다른 하니스는 훅 timeout/fail-closed/fail-open 정책을 이벤트별로 명시한다(openclaw의 `before_tool_call` 15s fail-closed, hermes의 `fail_closed` 플래그, claude-code의 `continueOnBlock`). OmO는 예외 격리(`safeCreateHook`) 하나뿐이고 **타임아웃 개념이 없다**. 대신 비용을 캐시·절단·문자 예산으로 관리한다. 장단점이 명확하다. 프로세스 격리가 필요 없고 지연이 낮지만, 느린 훅이 세션을 붙잡을 수 있다.

## 8. 확장 방법

### 1단계: 훅 디렉터리 복제

가장 단순한 템플릿은 `notepad-write-guard/`(`.ts` 2개, 테스트 없음), 패턴이 완전한 것은 `write-existing-file-guard/`(`.ts` 6개)다.

```bash
cp -r packages/omo-opencode/src/hooks/write-existing-file-guard \
      packages/omo-opencode/src/hooks/my-guard
```

### 2단계: `index.ts`에 팩토리

모든 훅은 `createXXXHook(deps) -> HookFunction` 형태를 따른다. 반환값은 `(input, output) => void`이며 `output`을 변형한다.

```ts
// packages/omo-opencode/src/hooks/my-guard/index.ts
export function createMyGuardHook(ctx: PluginContext) {
  return async (input: unknown, output: unknown) => { /* output 변형 */ }
}
```

의존성은 주입한다(`ctx`, `pluginConfig`, `modelCacheState`, `truncator` 등). 규칙 문서 금지 사항: `index.ts`에 비즈니스 로직 금지(barrel export만), catch-all 파일 금지(`utils.ts`/`helpers.ts`/`service.ts`), 주석은 기록적으로.

### 3단계: 티어 컴포저에 등록

경로: `packages/omo-opencode/src/plugin/hooks/create-*-hooks.ts` 중 해당 티어.

```ts
// 예: Tool Guard 티어 (create-tool-guard-hooks.ts)
const myGuard = isHookEnabled("my-guard")
  ? safeHook("my-guard", () => createMyGuardHook(ctx))
  : null
// 반환 객체에 myGuard 추가 (ToolGuardHooks 타입에도 추가)
```

선택 기준:

| 하고 싶은 일 | 등록할 파일 |
|---|---|
| 세션 수명주기 / 모델 / 커맨드 | `create-session-hooks.ts` |
| 툴 실행 전후 검사 | `create-tool-guard-hooks.ts` |
| 메시지 변환·컨텍스트 주입 | `create-transform-hooks.ts` |
| idle·압축·연속 실행 | `create-continuation-hooks.ts` |
| 스킬 인식 | `create-skill-hooks.ts` |
| 팀 모드 전용 | 해당 티어의 team-mode 조건부 블록 |

같은 페이즈 안에서는 **컴포저에 등록된 순서가 실행 순서**다. 앞의 훅이 원본 입력을 보고 뒤의 훅이 누적된 출력을 본다.

### 4단계: 설정 allowlist 등록

`packages/omo-opencode/src/config/schema/hooks.ts`의 `HookNameSchema`에 훅 이름을 추가해야 `disabled_hooks`로 끌 수 있다. 이 단계를 빠지면 `isHookEnabled("my-guard")`가 항상 참이 되어 설정으로 비활성화가 불가능하다.

### 5단계: 테스트

같은 디렉터리에 `*.test.ts`를 두고 given/when/then 스타일(중첩 `describe` + `#given`/`#when`/`#then`, 또는 인라인 `// given` / `// when` / `// then`)로 작성한다. Arrange-Act-Assert 주석은 금지. 프롬프트·마크다운 산문을 단언하는 테스트는 금지이고, 파싱·라우팅·디스패치·상태 같은 관측 가능 동작만 단언한다.

### 규칙 소비자(룰)를 만드는 경우

`.omo/rules/*.md`에 쓰고 `rules-injector`가 자동 주입하게 한다. hooks/가 아니라 룰 엔진으로 가는 편이 훨씬 싸다. 프론트매터 스키마는 `packages/rules-engine/src/engine/types.ts`와 `parser-frontmatter.ts`를 따른다.

### 검증 관례

레포는 훅 변경 시 "QA 없이는 머지 금지"를 강제한다(`AGENTS.md` "STOP. QA IS MANDATORY"). `packages/omo-opencode/` 하위 변경은 `.agents/skills/opencode-qa` 스킬을 돌리고, 증거를 `.omo/evidence/<YYYYMMDD>-<slug>/`에 남겨야 한다. `bun test` 통과나 타입체크 통과는 QA가 아니다.