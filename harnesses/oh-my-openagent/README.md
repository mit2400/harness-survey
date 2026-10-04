# oh-my-openagent (OmO) 심층 분석

`code-yeongyu/oh-my-openagent` 내부 구현을 8개 파일로 나눠 파고든 문서 세트다. 상위 문서 [`../oh-my-openagent.md`](../oh-my-openagent.md)가 "무엇이 있는가"를 나열했다면, 이 세트는 "어떻게 붙어 있는지"를 다룬다.

**왜 이 세트가 필요한가**: OmO는 자체 공식 문서를 레포 안에 45개 파일 / **10,882줄**(`docs/`)로 동봉한다 — 이 설문 저장소 전체(약 29개 문서)보다 3배가량 크다. 즉 *사용 설명*은 이미充分하다. 이 세트는 중복하지 않고, 공식 문서가 다루지 않은 층(패키지 내부 연결 구조, 어댑터 간 실제 차이, 미문서화된 서브시스템)을 채운다.

## 분석 기준 버전

| 항목 | 값 |
|---|---|
| 레포 | `code-yeongyu/oh-my-openagent` |
| 커밋 | `251cbfe` |
| 버전 | `5.1.13` |
| 수집일 | `2026-10-04` |

> ⚠️ **OmO는 매우 빠르게 움직인다.** 주간/일간 릴리스라 위 수치(패키지 50개, LoC, 훅 5-tier 구성)가 한두 릴리스 만에 부패한다. 인용된 파일 경로를 확인할 때는 반드시 위 커밋을 먼저 대조할 것.

## 문서 목록

| 파일 | 내용 |
|---|---|
| [01-architecture.md](01-architecture.md) | 50개 패키지 맵, 공용 core 18개와 4개 어댑터의 관계, 부트 플로우, 패키지 의존 그래프 |
| [02-targets.md](02-targets.md) | `omo-opencode` / `omo-codex` / `omo-senpi` / `omo-native` 이 실제로 무엇이 다른가 — 이름이 같은 것을 착각하지 말 것 |
| [03-memory.md](03-memory.md) | `memory-core` + Kibitzer BM25 recall + Reflection 상태머신 — **공식 문서 미문서화 구간** |
| [04-orchestration.md](04-orchestration.md) | `senpi-task` DAG 엔진 + `boulder-state` + ulw/goal 루프가 어떻게 결속되는가 |
| [05-hooks-rules.md](05-hooks-rules.md) | 5-tier 훅 시스템, `rules-engine`, AGENTS.md 주입 경로 |
| [06-skills-mcp-tools.md](06-skills-mcp-tools.md) | 7-source 스킬 발견, 3-tier MCP, hashline edit, LSP 경로 |
| [07-config-distribution.md](07-config-distribution.md) | 설정 해석(profiles/migration/doctor) + 12개 플랫폼 바이너리 배포 체계 |
| [08-upstream-docs-map.md](08-upstream-docs-map.md) | 공식 문서 45개 인덱스 + **공식 문서에 없는 것** 목록 |
| [09-native-runtime.md](09-native-runtime.md) | **OmO Native를 하니스로** — 런처(2.6k)/엔진(122k) 경계, compile·supervisor 생명주기, doctor 4종, bunshin 능력 설치 모델 |
| [10-vision-computer-use.md](10-vision-computer-use.md) | `look_at`(툴) vs `multimodal-looker`(에이전트) 3층 분리 + 비전 모델 폴백 체인 + `computer` 안전 모델(2티어 권한·스톕 차드·감사 로그) |

## 읽는 순서

### OmO를 쓰려는 사람
`01-architecture.md` → `02-targets.md` → `06-skills-mcp-tools.md` → `07-config-distribution.md`
어떤 타깃을 고르고(어댑터), 무엇을 쓸 수 있으며(스킬/MCP/툴), 어떻게 설정하는지까지만. 공식 `docs/guide/`가 이미 해주지만, 여기서는 "왜 이 설정이 그렇게 생겼는지"를 연결해준다.

### OmO를 改造하려는 사람 (포크/확장/디버깅)
`01-architecture.md` → `02-targets.md` → `05-hooks-rules.md` → `04-orchestration.md` → `03-memory.md`
LoC가 큰 `omo-opencode`(126,239) / `omo-senpi`(76,079) / `senpi-task`(53,551) 세 곳이 변경 표면의 대부분이다. 훅·에이전트·스킬 추가점은 모두 `omo-opencode` 내부 컴포저(`hooks/`, `agents/`, `src/tools/`, `src/mcp/`)에 몰려 있다.

### 다른 하니스와 비교하려는 사람
`02-targets.md` → `04-orchestration.md` → `03-memory.md` → `08-upstream-docs-map.md`
차이의 축은 세 가지다. (1) `omo-native`는 src LoC 0 — 재구현이 아니라 런처다. (2) 오케스트레이션이 DAG 기반이다. (3) 메모리가 git-backed MemFS + BM25 사이드카 구조다. 저장소 전체 교차 비교는 [`../../topics/`](../../topics/) 참조.

## 가치가 가장 높은 파일

공식 문서가 침묵하는 구간이라, 이 설문 저장소 안에서 유일하게 1차 출처로 읽을 수 있는 내용이 많다.

| 파일 | 이유 |
|---|---|
| [03-memory.md](03-memory.md) | `grep docs/` 결과 **`memory-core` 언급 0회**. 14,784줄짜리 메모리 엔진(Kibitzer BM25, Reflection 상태머신, Soul/Facts/People/Dream 서브엔진, worktree 실행, orphan sweep)이 전부 미문서화. 실제로 읽어야만 다룰 수 있는 부분 |
| [08-upstream-docs-map.md](08-upstream-docs-map.md) | 공식 45개 문서의 어디에 무엇이 있는지와 **어디가 비어 있는지**를 한 장에서 정리. 문서 커버리지 갭을 먼저 알고 읽어야 중복 시간을 안 쓴다 |
| [09-native-runtime.md](09-native-runtime.md) | Native를 고른 이유와 런처/엔진 경계. 컴퓨터 유즈처럼 **플러그인에서는 구조적으로 불가능한 기능**을 왜 native에서만 가능한지 |
| [10-vision-computer-use.md](10-vision-computer-use.md) | 비전/컴퓨터 유즈 실전 동작. `look_at` 7단계 실행 흐름, 왜 이미지를 서브세션으로 위임하는지, 스톕 차드가 왜 "중단 불가능한 실행"을 refuses하는지 |

추정 커버리지 갭(문서 내 언급 횟수 기준): `memory-core` 0, `isolation-core` 0, `kibitzer` 1, `boulder-state` 2, `senpi-task` 10, `omo-senpi` 13. 즉 대용량 서브시스템일수록 공식 문서 언급이 적다 — 밀도가 아니라 존재 이유로 문서가 배분된 듯하다.

## 수치 요약

| 항목 | 값 |
|---|---|
| 패키지 디렉터리 | 50개 (플랫폼 바이너리 12개 제외 시 38개) |
| 구성 | 공유 `*-core` 18 · MCP 5 · 어댑터 4 · senpi-desktop 5 · 기타(`boulder-state`·`senpi-task`·`rules-engine`·`lsp-daemon`·`shared-skills`·`utils`·`web`·`get-worker`) |
| 플랫폼 바이너리 | `oh-my-opencode-{darwin,linux,windows}-{arm64,x64,x64-baseline,musl}` 12개 |
| 큰 패키지 (LoC) | `omo-opencode` 126,239 · `omo-senpi` 76,079 · `senpi-task` 53,551 · `shared-skills` 39,139 · `memory-core` 14,784 · `omo-codex` 8,237 · `skills-loader-core` 7,560 · `team-core` 4,210 · `omo-config-core` 4,190 · `rules-engine` 3,052 |
| 작은 패키지 (LoC) | `isolation-core` 2,397 · `lsp-daemon` 2,347 · `boulder-state` 1,656 · `hashline-core` 1,407 · `delegate-core` 417 · `agents-md-core` 160 |
| 공식 문서 | `docs/` 45개 마크다운 / 10,882줄 (guide 17 · reference 22 · legal · troubleshooting) |
| 툴체인 | Bun 1.4.0 + Rust crates(`senpi-desktop` computer-use) |

> `omo-native`는 `src/` LoC가 0이다. 네 번째 "어댑터"로 세면 안 되고, 네 번째 타깃으로 세어야 한다(→ [02-targets.md](02-targets.md)).
