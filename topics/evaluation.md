# AI 코딩 하니스 평가/테스트 인프라 비교

## 요약 비교

| 하니스 | eval 시스템 | 플러그인 테스트 | 스킬 검증 | CI |
|--------|-------------|-----------------|-----------|-----|
| **pi-mono** | `packages/evals/` 전용 패키지 | 패키지별 테스트 | 스키마 기반 검증 | package.json 스크립트 |
| **hermes-agent** | `evals/` 69개 시나리오, `batch_runner.py`, `mini_swe_runner.py` | 플러그인 격리 테스트 | `agent/verify/recipes.py` | pytest 설정 |
| **codex** | 전용 eval 크레이트 없음, `core/tests/` 통합 | 플러그인 시스템 | 스킬 시스템 | GitHub Actions |
| **claude-code** | `mods/` 내장 플러그인 테스트 | `claude plugin test mods/<name>` | 플러그인 검증 에이전트 | GitHub Actions + 방화벽 |
| **opencode** | 패키지별 vitest | 플러그인 SDK 테스트 | 문서 기반 가이드 | 루트 차단, 패키지별 실행 |
| **oh-my-pi** | `ci-test-ts.ts coding-agent-heavy --full` | 패키지별 테스트 | 문서 기반 | 커스텀 테스트 러너 |
| **oh-my-openagent** | `bun test --timeout 20000` | 패키지별 테스트 | 문서 기반 | 표준 bun test |
| **openclaw** | `scripts/test-projects.mts` | 프로젝트 테스트 | 문서 기반 | 커스텀 테스트 스크립트 |

---

## pi-mono

**eval 시스템**: `packages/evals/` 디렉토리에 전용 평가 패키지 존재. 모델 품질 및 에이전트 성능 벤치마크를 위한 구조화된 eval 코드.

**테스트 인프라**: 모노레포 구조로 각 패키지별 테스트 스크립트 정의. package.json의 test 스크립트를 통해 실행.

**특징**: 평가를 별도 패키지로 분리하여 관리.

---

## hermes-agent

**eval 시스템**: 가장 체계적인 평가 인프라. `evals/` 디렉토리에 69개 이상의 시나리오 파일 존재:
- `batch_runner.py`: 배치 평가 실행기
- `mini_swe_runner.py`: SWE-bench 스타일 평가
- `agent/verify/recipes.py`: 에이전트 검증 레시피
- pytest 설정을 통한 테스트 자동화

**시나리오 커버리지**:
- 코드베이스 탐색 (`codebase_navigability/`)
- 컴팩션 (`compaction/`)
- 게이트웨이 (`gateway/`)
- 토큰 계량 (`token_accounting/`)
- 서브에이전트 (`subagent_process_handoff/`)
- 브라우저 사용 (`browser_use/`)
- 데스크톱 통합 (`desktop_bug_campaign/`, `desktop_mcp_oauth/`)

**플러그인 테스트**: `plugin_isolation/` 디렉토리에서 플러그인 격리 테스트.

**특징**: 가장 포괄적인 평가 시스템으로, 실제 사용 시나리오 기반의 회귀 테스트에 중점.

---

## codex

**eval 시스템**: 전용 eval 크레이트 없음. `codex-rs/core/tests/`에 통합 테스트 존재. Rust 생태계의 표준 테스트 관행 따름.

**테스트 인프라**: Cargo 기반 테스트, `codex-rs/` 하위 각 크레이트별 테스트 모듈.

**CI**: GitHub Actions 워크플로우.

**특징**: 별도 벤치마크 시스템보다 단위/통합 테스트에 집중. 평가가 필요한 경우 외부 도구에 의존.

---

## claude-code

**eval 시스템**: `mods/` 내장 플러그인 테스트 프레임워크:
- `claude plugin test mods/<name>` 명령
- 각 모드의 `tests/` 폴더에 테스트 파일
- `claude-code/testing`에서 `describe`, `expect`, `mock`, `test`, `tier` 제공
- 목 객체를 통한 외부 의존성 격리

**플러그인 테스트**: 가장 정교한 플러그인 테스트 시스템:
- 훅 기반 테스트 (`on`, `$` 인터페이스)
- 타입 안전한 모의 객체
- 타입체크 통합 (`tsc -p mods/tsconfig.json`)

**스킬 검증**: `plugins/plugin-dev/agents/plugin-validator.md` + `skill-reviewer.md` 에이전트, `plugins/plugin-dev/skills/` 개발 스킬.

**CI**: GitHub Actions + 보안 강화:
- `ubuntu-24.04-firewall` 러너
- 이그레스 방화벽 (`.github/egress-firewall.yaml`)
- 자동 권한 모드 (`--permission-mode auto`)
- `workflow-hardening.yml` 검증

**특징**: 보안이 강화된 CI 환경에서 플러그인 테스트를 실행. 내장 모드와 외부 플러그인 모두 동일한 테스트 인프라 사용.

---

## opencode

**eval 시스템**: 전용 eval 패키지 없음. 패키지별 vitest 테스트.

**테스트 인프라**:
- 루트 `package.json`: `"test": "echo 'do not run tests from root' && exit 1"` (의도적 차단)
- 각 패키지 디렉토리에서 테스트 실행
- `packages/opencode/` 및 기타 패키지별 테스트

**플러그인 테스트**: 플러그인 SDK를 통한 테스트 지원. 문서화된 가이드 존재.

**특징**: 모노레po 구조에서 테스트 실행 위치를 명확히 구분. 루트 레벨 테스트 실행을 방지하여 의도하지 않은 전체 테스트 실행 방지.

---

## oh-my-pi

**eval 시스템**: 커스텀 테스트 러너 사용:
- `bun ../../scripts/ci-test-ts.ts coding-agent-heavy --full`
- 코딩 에이전트 중심의 테스트

**테스트 인프라**: `packages/coding-agent/package.json`에 테스트 스크립트 정의.

**특징**: 코딩 에이전트 특화 테스트 실행 방식. 무거운 테스트를 별도로 분리하여 실행.

---

## oh-my-openagent

**eval 시스템**: 표준 bun 테스트:
- `bun test --timeout 20000`

**테스트 인프라**: 모노레po 내 표준 TypeScript 테스트.

**특징**: 간결한 테스트 설정. 별도 평가 인프라 없이 표준 테스트 도구 활용.

---

## openclaw

**eval 시스템**: 프로젝트 테스트 스크립트:
- `node --import ./scripts/tsx.mjs scripts/test-projects.mts`

**테스트 인프라**: 커스텀 테스트 스크립트를 통한 프로젝트 검증.

**특징**: 프로젝트 기반 테스트 접근. 실제 프로젝트를 통한 통합 테스트.

---

## 관찰 및 시사점

1. **평가 인프라의 성숙도 차이**: hermes-agent가 가장 체계적인 평가 시스템을 갖추고 있으며, 69개 이상의 시나리오로 실제 사용 커버리지를 확보. codex와 opencode는 별도 평가 시스템 없이 단위/통합 테스트에 의존.

2. **플러그인 테스트 표준화**: claude-code의 `claude plugin test` 명령은 플러그인 테스트를 표준화하는 좋은 사례. 타입 안전한 목 객체와 훅 기반 테스트 모델은 다른 하니스도 참고할 만함.

3. **CI 보안**: claude-code의 이그레스 방화벽과 자동 권한 모드는 AI 코딩 에이전트의 CI 보안 모델로 적합. 특히 외부 네트워크 접근 제한은 민감한 코드베이스에서 중요.

4. **테스트 실행 전략**: opencode의 루트 테스트 차단은 모노레po에서 의도하지 않은 전체 테스트 실행을 방지하는 실용적 접근.

5. **토큰 계량**: hermes-agent의 `token_accounting/` 평가는 모델 사용량과 비용 측정에 중점. 다른 하니스도 토큰 사용량 모니터링 메커니즘을 고려할 필요.

6. **커스텀 테스트 러너**: oh-my-pi와 openclaw는 각자의 필요에 맞는 커스텀 테스트 러너를 사용. 표준 테스트 도구가 모든 요구사항을 충족하지 못할 때의 대안적 접근.
