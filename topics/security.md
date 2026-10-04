# AI 코딩 하니스 보안 기능 비교

> 범위: 시크릿 관리 · 프롬프트 인젝션 방어 · 트러스트 모델 · 권한 모델 · 엔터프라이즈 정책.
> 샌드박스/퍼미션 실행 메커니즘은 `sandbox.md`에서 다룸. 여기서는 **시크릿/트러스트/인젝션/정책**에 집중.

## 요약 비교

| 하니스 | 시크릿 관리 | 인젝션 방어 | 트러스트 모델 | 정책/관리자 |
|---|---|---|---|---|
| **opencode** | `cli/cmd/debug/redact.ts`(키 패턴 기반 마스킹), `core/src/credential/sql.ts`, `auth/index.ts`, `mcp/oauth-provider.ts` | 로그/디버그 출력 마스킹(`redactConfig`), URL 내 사용자명·비밀번호·쿼리 시크릿 자동 `***` 처리 | permission 모듈(`permission/{index,evaluate,arity}.ts`), `install` 권한 게이트 | plugin 검증은 `plugin/`에分散, 별도 managed 정책 파일 없음 |
| **oh-my-openagent** | `omo-config-core` `mcp_env_allowlist` | Tier-3 session isolation (`omo-opencode/src/mcp/`) | — | — |
| **pi-mono** | `auth.json` (`getAuthPath()`), `settings.json` `defaultProjectTrust` | — | 프로젝트 trust prompt, `mcp.json` trust gating | — |
| **oh-my-pi** | `${VAR}` substitution, `!shell-command` secrets, registry credential entries (`config/api-key-resolver.ts`, `config/registry.ts`) | — | — | — |
| **codex** | `cli_auth_credentials_store`, `mcp_oauth_credentials_store` (keyring/file/auto) | `guardian/` auto-reviewer, `auto_review.policy` | `ProjectConfig.trust_level` (Trusted/Untrusted) | `[permissions]` profiles, `requirements.toml`, project-local denylist, `strict_config.rs` |
| **claude-code** | — (managed settings hooks chain) | `strictKnownMarketplaces`, `allowManagedHooksOnly` | marketplace trust | `PermissionMode` (`auto` = model classifier), managed settings hooks |
| **openclaw** | credential broker (write-only), `secrets/resolve.ts`, `secrets/store/`, `secret-mask.ts` | `security/external-content.ts`, `audit-deep-code-safety.ts`, `secret-equal.ts` | `security/audit-plugins-trust.ts`, `trusted-plan-path.ts`, provenance-gated memory | `security/dangerous-config-flags.ts`, `install-policy.ts`, `windows-acl.ts` |
| **hermes-agent** | `.env` 라우팅 (`config_env_routing.py`), `credential_lifecycle.py`, `vault.py`, `auth_oauth_pkce_plugin.py` | `debug_redaction.py`, `secret_prompt.py`, `plugins/security-guidance/patterns.py` | `tools/plugin_guard.py`, `plugins_provenance.py`, `plugin_isolation.py` | `mcp_security.py`, `managed_scope.py`, `security_audit.py` |

---

## opencode

**시크릿 관리.** `packages/opencode/src/auth/index.ts`가 인증 정보를 다루고, `packages/core/src/credential/sql.ts`가 자격증명 저장 계층(DB)이다. MCP 쪽은 `src/mcp/oauth-provider.ts`가 OAuth 토큰 교환을 담당한다. **전용 credential broker(	write-only 참조 모델)는 없다** — openclaw의 `src/secrets/` 계열과 혼동 주의.

**인젝션/노출 방어.** `packages/opencode/src/cli/cmd/debug/redact.ts`의 `redactConfig()`가 디버그 출력만 마스킹한다(설정 자체는 provider가 쓸 수 있도록 유지). 정규식 `/(?:api.?key|secret|password|token$|authorization$|cookie$|credential|private.?key)/i`로 키를 걸러 `***`로 대체하고, URL 값은 `username`/`password`/시크릿명 쿼리 파라미터가 있으면 전체를 마스킹하며, `headers` 블록은 하위 키까지 전파해 마스킹한다. `provider/transform.ts`, `server/routes/.../middleware/schema-error.ts` 등에도 시크릿 관련 처리 존재.

**권한 모델.** `packages/opencode/src/permission/` 3파일 — `index.ts`, `evaluate.ts`, `arity.ts`. `arity.ts`는 bash 커맨드 접두사→토큰 수 사전을 생성해 "사람이 읽을 수 있는 명령"(예: `git checkout main`)을 식별하고,以此 스코프 단위로 permission 규칙을 평가한다. 즉 **커맨드 단위 세분 권한**이 opencode의 핵심 보안 단위.

**플러그인.** `src/plugin/`에 산재하며(예: `plugin/xai.ts` 등 개별 플러그인 파일 수준) `src/security/` 같은 중앙 감사 모듈은 **존재하지 않는다**. 신뢰 승격/매니페스트 검증 로직은 plugin 로더 분산 구현.

**정책.** 별도 managed-settings/엔터프라이즈 정책 파일 없음(opencode는 단일 사용자 CLI 관점).

---

## oh-my-openagent

**시크릿 관리.** `omo-config-core`의 `mcp_env_allowlist`가 MCP 서버에 전달되는 환경변수를 화이트리스트로 제한한다. 허용되지 않은 환경변수는 MCP 프로세스에 노출되지 않는다.

**인젝션 방어.** `packages/omo-opencode/src/mcp/`의 Tier-3 session isolation이 MCP 서버를 세션 격리 환경에서 실행하여 인jection 시도가 다른 세션에 영향을 주지 못하게 한다.

**트러스트 모델 / 정책.** 별도의 트러스트 레지스트리나 관리자 정책 레이어는 확인되지 않음. 보안은 주로 격리와 환경변수 필터링에 의존.

---

## pi-mono

**시크릿 관리.** `packages/coding-agent/src/config.ts`의 `getAuthPath()`가 `~/.pi/agent/auth.json`을 가리키며, 인증 정보는 이 파일에 저장된다. `getSettingsPath()`는 `settings.json`, `getModelsPath()`는 `models.json`을 관리한다.

**트러스트 모델.** `settings.json`의 `defaultProjectTrust`가 신규 프로젝트의 기본 트러스트 레벨을 설정하고, 프로젝트 trust prompt가 사용자 확인을 요청한다. `mcp.json` trust gating은 프로젝트별 MCP 서버 접근을 제한한다.

**인jection 방어 / 정책.** 별도의 redaction 모듈이나 관리자 정책 파일은 확인되지 않음. 보안은 프로젝트 단위 trust 결정에 집중.

---

## oh-my-pi

**시크릿 관리.** `config/` 디렉터리의 `api-key-resolver.ts`가 API 키를 다양한 소스(환경변수, 파일, registry)에서 해석하고, `registry.ts`는 자격증명 항목을 관리한다. `${VAR}` 구문으로 설정값에서 환경변수를 참조하고, `!shell-command` 구문으로 셸 명령 실행 결과를 시크릿으로 주입할 수 있다. `resolve-config-value.ts`가 값 해석 파이프라인을 조정한다.

**인jection 방어 / 트러스트 / 정책.** 별도의 인jection 방어 모듈, 트러스트 레지스트리, 관리자 정책 레이어는 확인되지 않음. 시크릿 관리가 보안의 전부.

---

## codex

**시크릿 관리.** `config_toml.rs`의 `cli_auth_credentials_store`가 CLI 인증 정보 저장 방식을 `file` / `keyring` / `auto`로 설정한다. `mcp_oauth_credentials_store`가 MCP OAuth 자격증명에 동일한 옵션을 제공하며, `mcp_enterprise_managed_auth`가 엔터프라이즈 IdP 공유를 구성한다.

**인jection 방어.** `codex-rs/core/src/guardian/`가 auto-reviewer로 동작하여 모델 출력을 검토한다. `AutoReviewToml`의 `policy`와 `extra_policy` 필드가 Guardian 프롬프트에 정책 지시를 주입하고, `circuit_break_action`이 인터럽트 시 구조화된 에러 노출 여부를 제어한다.

**트러스트 모델.** `ProjectConfig.trust_level`이 `TrustLevel::Trusted` / `TrustLevel::Untrusted`를 정의하고, `is_trusted()` / `is_untrusted()`로 판별한다. `derive_permission_profile()`은 프로젝트 트러스트 상태에 따라 기본 샌드박스 모드를 결정한다.

**정책.** `[permissions]` 테이블이 명명된 권한 프로필을 정의하고, `default_permissions`가 기본 프로필을 선택한다. `requirements.toml`이 조직 정책을 선언하고, project-local denylist가 프로젝트별 차단 목록을 관리한다. `strict_config.rs`가 정책 준수를 강제한다. `approval_policy`와 `approvals_reviewer`가 명령 실행 승인 파이프라인을 구성한다.

---

## claude-code

**권한 모델.** `mods/types/claude-code.d.ts`의 `PermissionMode`는 `'default' | 'acceptEdits' | 'bypassPermissions' | 'plan' | 'dontAsk' | 'auto'`를 정의한다. `auto` 모드는 모델 분류기가 퍼미션 프롬프트를 자동 승인/거부한다. `permission_mode` 필드가 세션별 모드를 지정하고, `permissionDecision`이 PreModelSwitch 훅의 결과를 전달한다.


**트러스트 / 정책.** `strictKnownMarketplaces`(알려진 마켓플레이스만 허용)와 `allowManagedHooksOnly`(관리자 훅만 실행)는 **settings 키**이며, 레포에는 `mods/types/claude-code.d.ts`가 아니라 **`CHANGELOG.md`에서 확인**된다(`strictKnownMarketplaces` 7회, `allowManagedHooksOnly` 2회, `sandbox` 119회 언급). d.ts는 이 키들을 노출하지 않으므로 CHANGELOG가 유일한 1차 출처다. 훅 실행 순서는 `[managed settings hooks, ...hooks modules, other settings hooks as core]`로, 관리자 정책이 사용자 훅보다 우선한다.

**샌드박스.** `sandbox.*` 설정 그룹이 존재하며 `CHANGELOG.md`에 119회 등장 — bubblewrap/socat 기반 네이티브 격리, 네트워크·파일시스템·자격증명 주입을 다루는 독립 축이다(실행 메커니즘은 `sandbox.md` 참조).

**시크릿 관리.** 전용 credential broker는 없음. `settings.json`의 `env`와 권한 규칙으로 대체하며, 하위 프로세스가 `OTEL_*`를 상속하지 않게 하는 환경 변수 누출 방어가 CHANGELOG에 기록돼 있다(`:4732`).

---

## openclaw

**시크릿 관리.** `src/secrets/`는 openclaw 자체의 credential broker다(opencode에는 이 모듈이 **없다**). opencode가 파생된 관계이므로 구조는 유사하되, 이 파일들은 openclaw 구현이다. `resolve.ts`가 참조 해석, `store/`가 값 보관, `provider-credential-values.ts`가 프로바이더별 필드 정의, `runtime-auth-store-admission.ts`가 스토어 진입 제한, `secret-mask.ts`가 로그 마스킹을 담당한다. `audit.ts`가 시크릿 사용을 추적하고, `trusted-plan-path.ts`가 신뢰 경로를 제한한다.

**인jection 방어.** `src/security/external-content.ts`가 외부 콘텐츠를 분류하고, `audit-deep-code-safety.ts`가 코드 안전성을 검사한다. `secret-equal.ts`가 상수시간 비교를 제공한다.

**트러스트 모델.** `security/audit-plugins-trust.ts`가 플러그인 트러스트를 감사하고, `trusted-plan-path.ts`가 신뢰 계획 경로를 제한한다. 메모리는 provenance-gated되어 출처가 확인된 소스만 기록된다.

**정책.** `dangerous-config-flags.ts`가 위험 플래그를 차단하고, `install-policy.ts`가 설치 정책을 적용하며, `windows-acl.ts`가 Windows ACL을 관리한다.

---

## hermes-agent

**시크릿 관리.** `hermes_cli/config_env_routing.py`는 **키의 모양(UPPER_SNAKE)**으로 `.env` vs `config.yaml` 라우팅을 결정한다. `_ENV_SHAPE_RE = re.compile(r"^[A-Z][A-Z0-9_]*$")`가 환경변수 이름을 매칭하고, `is_env_setting_key()`가 `.env` 저장 여부를 판별한다. `save_env_setting()`은 `.env`에 쓰고 `config.yaml`의 중복 복사본을 제거한다. `credential_lifecycle.py`가 프로바이더 자격증명 순환을 관리하고, `vault.py`가 보안 저장소를 제공하며, `auth_oauth_pkce_plugin.py`가 MCP OAuth PKCE를 구현한다.

**인jection 방어.** `debug_redaction.py`가 디버그 출력에서 시크릿을 제거하고, `secret_prompt.py`가 시크릿 프롬프트를 처리한다. `plugins/security-guidance/patterns.py`가 보안 가이드 패턴을 정의한다.

**트러스트 모델.** `tools/plugin_guard.py`가 플러그인 실행을 가드하고, `plugins_provenance.py`가 플러그인 출처를 추적하며, `plugin_isolation.py`가 플러그인 격리를 적용한다.

**정책.** `mcp_security.py`가 MCP 보안을 관리하고, `managed_scope.py`가 관리자 범위를 정의하며, `security_audit.py`가 보안 감사를 수행한다.

---

## 관찰 및 시사점

1. **Credential broker 패턴.** opencode와 openclaw는 write-only broker로 시크릿을 참조로만 다루는 강한 설계를 취한다. 반면 pi-mono(`auth.json`), oh-my-pi(`${VAR}`/`!shell-command`), codex(keyring/file)는 파일/환경변수 기반의 단순한 저장에 머문다. hermes-agent는 `.env` 라우팅으로 시크릿과 설정을 분리하지만 값 자체는 평문으로 저장한다.

2. **트러스트 모델의 세분화.** codex(`ProjectConfig.trust_level`)와 pi-mono(`defaultProjectTrust` + `mcp.json` gating)는 프로젝트 단위 트러스트를 명시한다. opencode/openclaw는 플러그인 트러스트와 provenance-gated memory로 세분화한다. claude-code는 마켓플레이스 트러스트로 접근한다. oh-my-openagent와 oh-my-pi는 별도의 트러스트 레이어가 없다.

3. **인jection 방어.** opencode/openclaw(`external-content.ts`, `audit-deep-code-safety.ts`)와 hermes-agent(`debug_redaction.py`, `secret_prompt.py`)가 명시적 redaction/분류 모듈을 갖는다. codex는 Guardian auto-reviewer로 출력 검사를 수행한다. claude-code는 훅 체인 순서와 마켓플레이스 제한으로 방어한다. pi-mono, oh-my-pi, oh-my-openagent는 인jection 방어 모듈이 확인되지 않는다.

4. **엔터프라이즈 정책.** codex(`[permissions]`, `requirements.toml`, `strict_config.rs`)와 claude-code(`allowManagedHooksOnly`, managed settings hooks)가 가장 강력한 관리자 정책을 제공한다. hermes-agent(`managed_scope.py`, `security_audit.py`)가 그 뒤를 잇는다. 나머지 하니스는 조직 수준 정책 레이어가 부재하다.

5. **OAuth.** hermes-agent(`auth_oauth_pkce_plugin.py`)와 codex(`mcp_oauth_credentials_store`, `mcp_enterprise_managed_auth`)만이 MCP OAuth를 명시적으로 지원한다. 나머지는 자격증명 저장 방식만 제공하고 OAuth 플로우는 외부에 위임한다.

6. **공통 약점.** oh-my-openagent와 oh-my-pi는 시크릿 관리와 격리에 집중하지만 트러스트 모델, 인jection 방어, 엔터프라이즈 정책이 부재하다. pi-mono는 트러스트는 있지만 redaction과 정책 레이어가 없다. 이들 하니스가 엔터프라이즈 환경에서 사용되면 외부 정책 레이어로 보완해야 한다.
