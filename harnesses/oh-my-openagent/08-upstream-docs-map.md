# 08 — oh-my-openagent 공식 문서 지도 + 문서화되지 않은 것

> **이 문서의 지위**: 전체 deep-dive 세트의 **검증 기준선(verification baseline)**.
> 아래에 있는 모든 파일 경로와 모든 카운트는 실제 `find` / `wc -l` / `grep -r` 출력에서 얻은 값이다.
> 이 문서에서 "문서화되어 있다"고 말하는 것은 전적으로 *공식 `docs/` 트리에 그 파일이 존재한다*는 뜻이며,
> **내용의 정확성까지 보증하는 것이 아니다.**

- **대상 저장소**: `/home/minkoo/study/repos/oh-my-openagent`
- **Pin**: commit `251cbfe` (`251cbfe0381ab5cced2034cd87566b5891fd3d67`), `package.json` version **5.1.13**
- **문서 기준일**: 2026-10-04
- **측정 대상**: `docs/**/*.md` — **45개 파일, 10,882줄**

> **주의 (측정 방법론)**: 이 문서에서 "언급 횟수"는 두 가지를 함께 적는다.
> `files` = 그 문자열이 등장하는 **문서 파일 개수**, `mentions` = `grep -o` 기준 **실제 출현 횟수**.
> 두 값이 다르면, 즉 한 문서가 서술형 단락으로 몇 번 반복 서술하는지가 다르다는 뜻이다.
> 검증 시 **files 카운트를 기준으로** "공식 문서에 존재하는가"를 판정할 것.

---

## 1. 전체 파일 목록 (45개, 디렉터리별)

### 1.1 루트 레벨 (3개, 321줄)

| 파일 | 줄 | 이 문서가 답하는 질문 |
|---|---:|---|
| `docs/AGENTS.md` | 85 | "어떤 문서를 읽어야 하는가" — docs/ 자체의 인덱스. `## WHERE TO LOOK` 표가 30개 파일 기준(2026-08-24 생성, f3642fcda 기준)이라 **현재 45개와 이미 어긋나 있다** |
| `docs/manifesto.md` | 203 | "이 프로젝트가 왜 이런 설계를 택했는가" — Human Intervention is a Failure Signal, Indistinguishable Code 등 설계 신조 |
| `docs/model-capabilities-maintenance.md` | 33 | "모델 별칭(alias)을 언제 어떻게 추가/갱신하는가" — 내부 정책 문서 |

### 1.2 `docs/guide/` — 사용자 안내 (16개, 3,142줄)

| 파일 | 줄 | 이 문서가 답하는 질문 |
|---|---:|---|
| `docs/guide/overview.md` | 269 | "Oh My OpenAgent란 무엇인가" — 빠른 시작, 프로파일 선택, 아키텍처 다이어그램(Kibitzer 포함) |
| `docs/guide/install.md` | 150 | "원라인 설치로 OmO를 어떻게 넣는가" — 설치 확인, PATH 수정 |
| `docs/guide/installation.md` | **1057** | "플러그인 에디션을 어떻게 설치하고 무엇이 들어 있나" — **최대 파일**. `Which edition should I pick?` / For Humans. 슬래시 커맨드·에이전트·훅 대량 목록 포함 |
| `docs/guide/binary-install.md` | 108 | "컴파일된 `omo` 바이너리를 어떻게 설치하는가" — 원라인 설치(권장), OS별 수동 설치 |
| `docs/guide/migrating-from-opencode.md` | 195 | "기존 OpenCode 사용자를 OmO로 옮기려면" |
| `docs/guide/orchestration.md` | 325 | "에이전트들이 어떻게 협업하는가" — TL;DR 판단표, 아키텍처, Kibitizer nudges, boulder 기반 ulw 흐름 |
| `docs/guide/agents.md` | 34 | "OmO에 어떻게 위임하는가" — 위임 가능한 작업의 종류, Desktop Agents 패널 |
| `docs/guide/agent-model-matching.md` | 293 | "에이전트별로 어떤 모델을 붙여야 하나" — 4개 lane, 프롬프트 프리셋이 있는 모델 |
| `docs/guide/team-mode.md` | 151 | "Team Mode(옵트인 멀티에이전트)를 언제 어떻게 쓰나" |
| `docs/guide/senpi-task.md` | 156 | "Senpi 태스크 위임" — 자식 스폰, in-process vs process |
| `docs/guide/computer-use.md` | 167 | "OmO Native에서 컴퓨터 조작을 켜는가" — 운영체제별 설정 |
| `docs/guide/btw.md` | 93 | "부업 대화(side conversation)를 임시로 여는가" — 동작, 조작 |
| `docs/guide/workflows.md` | 40 | "워크플로(mass ulw)를 언제 쓰나" — 시작 방법 |
| `docs/guide/keywords.md` | 27 | "예약 키워드가 무엇인가" — ulw, ulw loop, ulw plan, ulw research, mass ulw |
| `docs/guide/telemetry.md` | 45 | "텔레메트리가 무엇을 보내고 어떻게 끄는가" |
| `docs/guide/desktop-updates.md` | 32 | "OmO Desktop를 어떻게 업데이트하는가" — 릴리스 노트, 트랙 |

### 1.3 `docs/reference/` — 참조 문서 (23개, 7,222줄)

| 파일 | 줄 | 이 문서가 답하는 질문 |
|---|---:|---|
| `docs/reference/configuration.md` | **1344** | "설정 필드 전체는 무엇인가" — **최대 파일**. 목차 + Getting Started. 에디션별 설정 필드 총람 |
| `docs/reference/features.md` | **1207** | "기능 하나하나가 정확히 무엇을 하는가" — **第二大 파일**. Agents, Launch in background 등 기능 단위 |
| `docs/reference/omo-thread.md` | **575** | "스크립트/커넥터에서 세션 게이트웨이(`omo thread`)를 어떻게 호출하나" — JSON 형태, exit code, 스토어 확장 |
| `docs/reference/omo-daemon.md` | **540** | "세션별 엔진 호스트는 어떻게 생기고 어디에 있나" — `omo daemon`, 마이그레이션, 롤백 |
| `docs/reference/omo-json.md` | **515** | "하네스 중립 `omo.json`의 모든 스키마는 무엇인가" — 파일 위치와 우선순위, `$schema` |
| `docs/reference/opencode-config.md` | **445** | "OpenCode 에디션 설정(레거시)은 무엇인가" — Agents, Task System 등 레거시 경로 |
| `docs/reference/senpi-telemetry.md` | **336** | "Senpi 텔레메트리 이벤트 스키마는 무엇인가" — 이벤트별 필드 표 |
| `docs/reference/cli.md` | **312** | "CLI 명령 전체는 무엇인가" — 바이너리 명령, 기본 사용법, exit code 규칙 |
| `docs/reference/mass-ulw-protocol.md` | 292 | "mass-ulw 뷰어 프로토콜은 무엇인가" — 전송 계층/도달성, 푸시 채널 |
| `docs/reference/known-issues.md` | 213 | "알려진 버그와 우회 방법은 무엇인가" — 이슈 번호별(worktree merge 상태, `ulw` 플래너 모드 등) |
| `docs/reference/prompt-async-gate-rfc.md` | 235 | "`prompt_async_gate`의 예약 기반 중복 주입 방지 설계는 무엇인가" — ADR |
| `docs/reference/re-export-shim-inventory.md` | **284** | "재수출 shim이 어디에 있고 무엇을 되받는가" — 타깃 패키지별 집계 + 전수 경로 |
| `docs/reference/monitor.md` | 176 | "Monitor를 어떻게 켜고 어떻게 설정하나" |
| `docs/reference/omo-ai-publishing.md` | 143 | "omo-ai 퍼블리싱 런북은 무엇인가" — 부트스트랩 상태, 채널 선택 |
| `docs/reference/rules-injection-cross-module-comparison.md` | 116 | "rules 주입은 모듈별로 어떻게 다른가" — 커밋 현황, 성능 베이스라인 |
| `docs/reference/codex-telemetry.md` | 88 | "Codex 경량 텔레메트리는 무엇을 보내나" — 이벤트, 소스 |
| `docs/reference/release-process.md` | 74 | "릴리스 절차는 무엇인가" — 표준 게이트, 실패한 퍼블리시 재개 |
| `docs/reference/web-terminal-visual-qa.md` | 75 | "웹 터미널 비주얼 QA의 증거 계약은 무엇인가" — 라이브 캡처 |
| `docs/reference/computer.md` | 64 | "네이티브 `computer` 도구의 계약은 무엇인가" — 액션 목록 |
| `docs/reference/shared-core-multi-pr.md` | 59 | "공유 코어 멀티-PR 추출 시 QA는 무엇인가" — PR 매트릭스. `hashline-core`, `boulder-state`, `telemetry-core` 명시 |
| `docs/reference/github-attachment-upload.md` | 51 | "GitHub PR 증거 첨자는 어떻게 올리는가" — 계약, 직접 업로드 흐름 |
| `docs/reference/omob-dev-binary.md` | 47 | "`omob` 개발용 바이너리는 어떻게 만드는가" — 최신 커밋에서 빌드 |
| `docs/reference/lazycodex-npm-reservation.md` | 31 | "`lazycodex-ai` npm 이름 점유 퍼블리싱 절차" — 버전 네임스페이스, 사전 점검/재시도 예산 |

### 1.4 `docs/legal/` — 법적 (2개, 155줄)

| 파일 | 줄 | 이 문서가 답하는 질문 |
|---|---:|---|
| `docs/legal/privacy-policy.md` | 98 | "어떤 정보를 수집하고 텔레메트리는 어떻게 동작하나" |
| `docs/legal/terms-of-service.md` | 57 | "사용 약관과 라이선스는 무엇인가" |

### 1.5 `docs/troubleshooting/` (1개, 42줄)

| 파일 | 줄 | 이 문서가 답하는 질문 |
|---|---:|---|
| `docs/troubleshooting/ollama.md` | 42 | "Ollama의 도구 호출 스트리밍 실패는 왜 일어나고 어떻게 우회하나" — 현재 상태 포함 |

---

## 2. 총량 및 guide vs reference 분할

| 디렉터리 | 파일 수 | 줄 수 | 전체 대비 |
|---|---:|---:|---:|
| `docs/reference/` | 23 | 7,222 | 66.4% |
| `docs/guide/` | 16 | 3,142 | 28.9% |
| 루트 (3개) | 3 | 321 | 3.0% |
| `docs/legal/` | 2 | 155 | 1.4% |
| `docs/troubleshooting/` | 1 | 42 | 0.4% |
| **합계** | **45** | **10,882** | 100% |

읽어낼 사실 세 가지:

1. **reference가 guide의 2.3배다.** 줄 수로도(7,222 vs 3,142), 파일 수로도(23 vs 16).
2. **문서가 5개 파일에 몰려 있다.** `configuration.md`(1,344) + `installation.md`(1,057) + `features.md`(1,207) = **3,608줄 = 전체의 33.2%**. 세 문서만 읽어도 3분의 1이 커버된다.
3. **troubleshooting는 파일 1개 42줄.** 실사용자가 부딪히는 문제를 다루는 면적이 가장 얇다. 반면 `docs/reference/known-issues.md`(213줄)가 사실상 troubleshooting을 수행하고 있다 — 즉 **문제 대응 문서가 `reference/` 밑에 흩어져 있는 구조**다.

---

## 3. [핵심] 공식 문서에서 빠진 것

측정 명령: `grep -rl --include='*.md' "<패키지명>" docs/` (files)와 `grep -ro --include='*.md' "<패키지명>" docs/ | wc -l` (mentions).

### 3.1 언급 카운트 표

| 패키지/서브시스템 | files (문서 파일 수) | mentions (출현 횟수) | 판정 |
|---|---:|---:|---|
| `memory-core` | **0** | **0** | **완전 무문서화** |
| `isolation-core` | **0** | **0** | **완전 무문서화** |
| `agent-message-board-client` | **0** | **0** | **완전 무문서화** (존재 확인: docs/에 문자열 자체가 없음) |
| `telemetry-core` | 1 | 1 | 명목뿐 (파일명 나열 수준) |
| `boulder-state` | 2 | 5 | 기능 서술은 있으나 패키지 아키텍처 문서 없음 |
| `kibitzer` | 1 | 24 | 1개 파일(`orchestration.md`)에 집중 서술 |
| `delegate-core` | 2 | 3 | 거의 없음 |
| `lsp-daemon` | 2 | 2 | 명목뿐 |
| `agents-md-core` | 2 | 4 | 거의 없음 |
| `mcp-client-core` | 2 | 23 | 한 문서가 집중 서술 |
| `hashline-core` | 2 | 8 | 이름만 등장 |
| `rules-engine` | 3 | 8 | 이름 위주 |
| `prompt-async-gate` | 3 | 10 | 그나마 전용 ADR 존재 |
| `skills-loader-core` | 3 | 38 | 슬슬 실사용 가이드 존재 |
| `team-core` | 3 | 42 | 팀 기능은 실사용 가이드 존재 |
| `omo-native` | 7 | 12 | 네이티브 타깃은 일부만 |
| `omo-codex` | 8 | 64 | Codex 타깃이 가장 많이 다뤄짐 |
| `senpi-task` | 10 | 22 | 가장 넓게 산재 |
| `omo-senpi` | 13 | 25 | 가장 넓게 산재 |

> 비교 기준선: `core`라는 문자열 자체는 docs/ 전체에서 245회, `OpenCode` 232회, `Codex` 151회 등장한다.
> 즉 **"core"를 붙인 공유 코어 패키지들이 전혀 다른 문서 층위(설정/기능/원격지)에서 다루어지는 것과 정반대**다.

### 3.2 ZERO/low-mention 서브시스템 → 독자가 공식 출처를 갖지 못하는 것

한 줄 요약. 각 항목의 "공식 출처 없음"은 **docs/ 안에 그 내용을 설명하는 파일이 없다**는 의미이고, 소스코드를 직접 읽는 것 외에 공식 근거가 없다는 뜻이다.

**완전 무문서화 (files=0)**

- **`memory-core`** — git 기반 MemFS(메모리를 git 저장소에 두는 설계), BM25 계열 "Kibitzer" 리콜 사이드카의 구현, Reflection 상태 머신(remember/reflect 루프의 전이 규칙)을 설명하는 공식 문서가 **하나도 없다**. Kibitzer의 *행동* 서술은 `docs/guide/orchestration.md:81`과 `docs/guide/overview.md:126` 두 문장에 압축돼 있고, **언제 무엇이 저장되고 어떻게 회수되고 언제 버려지는지에 대한 규칙은 문서에 없다.** 플래그시프 메모리 기능이 공식 문서 커버리지 0이라는 점이 가장 큰 격차다. 켜는 방법은 `memory.recall.enabled: false` 한 줄뿐이다.
- **`isolation-core`** — 에이전트 격리(샌드박스/분리 프로세스) 경계의 설계 문서가 없다. 어느 작업이 격리 안에서 도는지에 대한 공식 설명이 존재하지 않으므로, 격리를 가정하는 것은 순전히 추측이다.
- **`agent-message-board-client`** — 문자열이 docs/ 어디에도 없다. 에이전트 간 메시지 보드 클라이언트의 계약·전송 형식·재시도 규칙에 대한 공식 문서가 없다.

**명목 격차 (files=1~2, 사실상 이름만)**

- **`telemetry-core`** (1파일 1회) — 텔레메트리 **이벤트 스키마**는 `docs/reference/senpi-telemetry.md`(336줄)와 `codex-telemetry.md`(88줄)에 정밀하게 적혀 있지만, **이벤트를 만들고 버퍼링하고 전송하는 코어의 계약 문서는 없다.** 즉 "무엇이 보내지는가"는 알 수 있고 "무엇이 언제 보내는지(배치/샘플링)는 모른다."
- **`boulder-state`** (2파일) — `.omo/boulder.json` 스키마는 `docs/guide/orchestration.md:169`(works + active_work_id 구조)와 `docs/reference/cli.md:52`(boulder 서브옵션)에 흩어져 있을 뿐, **공식 스키마 문서가 없다.** 멀티 워크 레지스트리의 상태 전이 규칙은 전적으로 코드 읽기 대상이다.
- **`delegate-core`** (2파일, 3회) — 위임( delegation) 자체는 `guide/agents.md`, `guide/senpi-task.md`, `reference/features.md`에 사용자 관점에서 설명돼 있으나, **`delegate-core`의 계약은 없다.** 프롬프트 조립·에이전트 등록·타임아웃 정책 같은 구현 규칙을 공식 근거로 삼을 수 없다.
- **`lsp-daemon`** (2파일, 2회) — LSP 도구 계약은 `features.md` 쪽에 사용자 레벨로만 있고, **daemon의 수명주기/프로토콜 버전/재시작 정책 문서가 없다.**
- **`agents-md-core`** (2파일, 4회) — `AGENTS.md` 로딩 규칙은 문제없어 보이지만, **파싱·병합·상속 우선순위를 정의한 공식 사양이 없다.**
- **`hashline-core`** (2파일, 8회) — `shared-core-multi-pr.md:9`에 이름만 나열. 해시라인 도구의 입력 계약/충돌 규칙 공식 문서 없음.

**약한 문서화 (files=3)**

- **`rules-engine`** (3파일, 8회) — 규칙 주입의 모듈 간 비교는 `rules-injection-cross-module-comparison.md`에 있으나, 이 문서는 116줄짜리 **성능/커밋 스냅샷 보고서**에 가깝고 규칙 문법·우선순위·해석 순서의 정식 레퍼런스가 아니다.
- **`prompt-async-gate`** (3파일, 10회) — `prompt-async-gate-rfc.md`(235줄)가 유일한 진짜 설계 문서다. **ADR 한 편에 의존하는 유일한 서브시스템**이므로, 이 ADR이 틀렸다면 공식적으로 막을 곳이 없다.

### 3.3 문서가 있는 쪽 (비교군)

- `team-core` (3파일 42회) / `skills-loader-core` (3파일 38회) — 팀·스킬은 서술 문서가 실사용 가이드 수준까지 내려와 있다. 대조군으로 충분하다.
- `omo-codex`(8/64), `senpi-task`(10/22), `omo-senpi`(13/25), `omo-native`(7/12) — 타깃 패키지는 문서 존재 여부의 차이가 아니라 **문서 **관점**의 차이다**(§5).

---

## 4. 문서화 성격의 결함 (파일을 넘어선 것)

파일 개수만으로는 드러나지 않지만 검증 기준선으로 반드시 기록해야 할 사항:

1. **`docs/AGENTS.md`가 자기 자신보다 오래되었다.** 헤더가 `Generated: 2026-08-24 / f3642fcda`이고 "30 tracked Markdown files across 6 subdirectories (guide 7, reference 18, examples 3 JSONC, legal 2, templates 1, troubleshooting 1)"라고 서술한다. 현재 실측은 **45개 md 파일, md 하위 디렉터리 4개**다. `examples/`와 `templates/` 하위의 JSONC/파일은 이번 측정의 md 범위 밖이므로 별도 확인이 필요하지만, **인덱스 자체가 15개 파일 차이로 뒤처져 있다.** → `docs/AGENTS.md`의 WHERE TO LOOK 표를 검증 기준으로 쓰면 안 된다.
2. **`orchestration.md`에 소스 주석이 남아 있다.** `docs/guide/orchestration.md:1` HTML 주석에 `packages/senpi-task/src/...`, `packages/omo-senpi/skills/ulw-plan/SKILL.md`, `packages/omo-config-core/src/schema/agent.ts` 등 내부 소스 경로가 나열돼 있다. 즉 이 문서는 **코드에서 추출한 문서**이며, 소스 경로를 통해 원본을 역추적할 수 있는 유일한 문서 중 하나다. (이 정보는 §4-1의 인덱스 불일치와 짝을 이루는 다리 역할을 한다.)
3. **`docs/reference/`가 `troubleshooting/`를 대체하고 있다.** §2-3에서 확인한 구조적 문제.

---

## 5. Senpi/Desktop 타깃 vs OpenCode 플러그인 타깃

문서가 다루는 대상(target/edition)이 서로 다르며 **기능이 타깃별로 갈린다.** 한 문서를 읽고 다른 타깃의 동작을 추측하면 안 된다.

### 5.1 명시적으로 타깃이 다른 파일들

| 구분 | 파일 근거 | 내용 |
|---|---|---|
| **OpenCode 플러그인 타깃** | `docs/guide/installation.md` (헤딩 `## Which edition should I pick?`), `docs/reference/opencode-config.md:1` (헤딩 `# OpenCode edition configuration (legacy)`) | "OpenCode 에디션 설정"이 **legacy**로 명시됨. `OpenCode` 문자열 232회 — 문서 전체의 기본 가정 |
| **컴파일된 바이너리 / CLI 타깃** | `docs/guide/binary-install.md`, `docs/reference/cli.md`, `docs/reference/omo-daemon.md`, `docs/reference/omo-thread.md`, `docs/reference/omo-json.md`, `docs/reference/omob-dev-binary.md` | `omo` / `omob` 바이너리와 스크립트·커넥터용 게이트웨이. `omo.json`은 **하네스 중립(harness-neutral)** 이라 명시(`reference/omo-json.md`) |
| **OmO Native / Desktop 타깃** | `docs/guide/computer-use.md:1` (`# Computer use in OmO Native`), `docs/guide/desktop-updates.md`, `docs/guide/agents.md` (Desktop Agents 패널), `docs/reference/computer.md` (`# The native computer tool contract`) | 데스크톱 앱에 붙는 네이티브 전용 기능. `omo-native` 문자열 12회 |
| **Codex 타깃** | `docs/reference/codex-telemetry.md`, `docs/reference/lazycodex-npm-reservation.md`, `docs/reference/omo-ai-publishing.md` | Codex 경량 텔레메트리와 lazycodex-ai npm 채널. `Codex` 문자열 151회 |
| **Senpi 타깃** | `docs/guide/senpi-task.md`, `docs/reference/senpi-telemetry.md` | 세션 ID가 `senpi:<session_id>`로 접두되는 Senpi 런타임(`guide/orchestration.md:169`). `Senpi Native`라는 문구는 0회 — 문서는 "Senpi"와 "OmO Native"을 **분리된 이름으로** 쓴다 |

### 5.2 타깃 분기를 넘나드는 문서

- `docs/guide/installation.md`(1,057줄) — **분기점의 중심 문서.** `Which edition should I pick?`로 시작해 여러 에디션을 나눠 제시하고, 이후 슬래시 커맨드·에이전트·훅 목록을 나열한다. 다만 **목록이 어느 타깃 유효한지 표기되지 않은 채 나열되는 구간이 있어**, 타깃 판별은 이 문서의 최우선 확인 대상이다.
- `docs/reference/configuration.md`(1,344줄) — `edition` 문자열 4회. 설정 키가 타깃별로 갈리는지 확인 필요.
- `docs/guide/overview.md` — `omo-native` 참조와 Kibitzer 다이어그램을 함께 담고 있어 두 타깃을 한 문서에서 다룬다.
- `docs/reference/re-export-shim-inventory.md`(284줄) — **타깃이 가장 명확한 문서.** 타깃 패키지별 집계와 전수 shim 경로를 표로 제공. 타깃 분기를 파야 한다면 여기가 출발점이다.

### 5.3 검증 규칙

> OpenCode 타깃의 서술을 Senpi/Desktop/Codex 타깃에 그대로 적용하지 않는다. 반대도 마찬가지다.
> 교차 검증이 필요할 때는 **타깃(edition)별 대상 패키지(`omo-native`(`omo-native` / `omo-codex` / `omo-senpi`) 구현 소스**를 `docs/reference/re-export-shim-inventory.md`의 경로 표로 따라가 직접 확인한다.

---

## 6. 이 문서如何使用 — deep-dive 세트의 사용법

**역할**: 이 파일은 나머지 deep-dive 문서들이 "이 주장을 어디까지 믿어도 되는가"를 판정하는 **단일 기준선**이다.

1. **출처 우선순위 규칙**
   - Tier 1 (공식): 위 §1에 나열된 실제 파일. 문장 단위로 인용할 때 파일 경로 + 줄 번호를 함께 쓴다.
   - Tier 2 (자기 기술 문서): `docs/reference/prompt-async-gate-rfc.md`, `shared-core-multi-pr.md`, `re-export-shim-inventory.md`처럼 소스 경로를 담은 문서. Tier 1과 소스가 충돌하면 **소스가 이긴다**(코드 대조 필수).
   - Tier 3 (무문서화): §3의 files=0/low 항목. **여기는 오직 소스코드로만 기술할 수 있다.** 어느 deep-dive 문서든 이 영역에 대해 "공식 문서에 따르면"이라는 표현을 쓰면 **오류다.** 반드시 "소스 기준"이라고 명시한다.

2. **§3 표는 인용 횟수가 아니라 존재 횟수로 판정한다.** `memory-core = 0`은 "간헐적으로 언급되지 않는다"가 아니라 **"공식 문서가 존재하지 않는다"**를 뜻한다. 이 세 문서의 격차를 두고 "문서화된 기능"으로 서술하면 검증 실패다.

3. **§1의 줄 수·파일 수는 스냅샷 값이다.** 다른 문서가 파일 수나 분할 비율을 인용하면 `45개 / 10,882줄 / reference 66.4% / guide 28.9%`와 일치하는지 확인한다. 불일치하면 base commit(`251cbfe`)이 다르다는 뜻이므로, 각 deep-dive 문서 맨 앞에 기준 commit을 적을 것.

4. **§5 타깃 규칙을 모든 기능 서술에 적용한다.** "X 기능이 동작한다"는 서술에는 반드시 타깃(OpenCode / OmO Native / Codex / Senpi / CLI)을 명시해야 한다. 타깃 미명시 서술은 검증 대상이다.

5. **§4의 인덱스 불일치를 경고로 유지한다.** `docs/AGENTS.md`의 WHERE TO LOOK 표는 stale이므로, 다른 deep-dive 문서가 이를 근거로 "공식 인덱스에 없다"고 쓰면 안 된다. 경로가 `docs/`에 실제 존재하는지는 항상 `find`로 재확인한다.

6. **이 문서가 바뀌는 경우는 base commit이 바뀌었을 때뿐이다.** 그 외의 갱신(설명문 다듬기)은 허용하지 않는다 — 기준선이 변동하면 세트 전체의 검증 결과가 무효가 된다.
