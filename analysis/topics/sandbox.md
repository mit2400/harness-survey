# 샌드박스 / 격리 비교

## 요약

| 하니스 | OS 샌드박스 | 권한/승인 | 컨테이너/원격 실행 | FS/네트워크 격리 |
|---|---|---|---|---|
| opencode | 없음 | **Permission 서비스** (ask/allow/deny, wildcard ruleset) | 없음 | OS 격리 아님. 도구 숨김 + external_directory 룰 |
| oh-my-openagent | **Seatbelt(macOS)/bwrap(Linux)** — 메모리 리플렉션 자식 전용 | OpenCode `permission` 상속 + 에이전트 tool restriction | 없음 | sandbox-exec SBPL / bwrap ro-bind+writable bind |
| pi-mono | 없음 (외부 위임) | 전용 엔진 없음 | **Docker / Docker Sandboxes / OpenShell / Gondolin micro-VM** (문서+확장) | 방식을 사용자가 선택, 자격증명 배치 주의 |
| oh-my-pi | 없음 | **approval 엔진** (read/write/exec tier, always-ask/write/yolo) | 없음 | 없음 (승인은 containment 아님, 문서 명시) |
| codex | **Seatbelt / Landlock+bwrap / PID ns / Windows Sandbox(MXC)** | `SandboxMode` + `approval_policy` + PermissionProfile | 없음 (exec-server는 별도) | 네트워크 프록시, writable_roots, violation 감지 |
| claude-code | **Bash 샌드박스** (enabled/network/excludedCommands) | `PermissionMode` 6종 + permissions allow/ask/deny | 없음 | allowedDomains, unix socket, proxy port, sandbox override |
| openclaw | **Docker/Podman/SSH 백엔드 + 브라우저 컨테이너** | tool-policy + run-authority + 감사 모듈 | Docker(기본)/podman/remote-shell/ssh | readOnlyRoot, network=none, bind-mount 검증, host net 차단 |
| hermes-agent | **Docker/SSH/Modal/Daytona/Singularity/Vercel Sandbox** | approval 큐 + file_safety + secret_scope | 6개 터미널 백엔드 (TERMINAL_ENV) | iron-proxy egress, write/read deny, NT-namespace 가드 |

## opencode
- **OS 샌드박스 없음.** `grep -ri sandbox src/`의 실제 매치는 Modal provider 플러그인(`packages/opencode/src/plugin/modal/`)과 문서뿐. Seccomp/Landlock/bubblewrap/Docker 기반 격리 미구현.
- **권한/승인 = Permission 서비스** (`packages/opencode/src/permission/index.ts`)
  - `ask`: 요청의 각 pattern을 `evaluate(permission, pattern, ruleset, approved)`로 판정 — 마지막 매치 우선(last-match-wins) + `Wildcard.match`. `deny`→`DeniedError`, `allow`→통과, 미매치→`ask`.
  - `ask`가 필요하면 pending `Deferred`를 만들고 `Event.Asked` 발행. `reply`는 `once`/`always`/`reject`; `always`는 `always` 패턴들을 approved에 push. `reject`는 같은 세션의 다른 pending까지 일괄 거부.
  - `fromConfig`가 `~`/`$HOME` 확장, `disabled()`가 `edit/write/apply_patch`→`edit`, `list_*_resource`→`read`로 정규화해 `visibleTools`로 도구를 숨김.
  - ACP 브리지: `packages/opencode/src/acp/permission.ts`; 서브에이전트 필터: `packages/opencode/src/agent/subagent-permissions.ts`; 인자 개수 기반(`permission/arity.ts`).
- **설정 키** (`packages/core/src/v1/config/permission.ts`): `permission` — 문자열 action(`ask|allow|deny`) 또는 오브젝트. 오브젝트 키: `read, edit, glob, grep, list, bash, task, external_directory, todowrite, question, webfetch, websearch, lsp, doom_loop, skill`; 값은 action 또는 `패턴→action` 맵.
- **컨테이너/네트워크 격리 없음** — 프로세스/FS 샌드박스는 범위 밖.

## oh-my-openagent (OmO)
- **OS 샌드박스 = 리플렉션/드림 자식 프로세스 전용 Seatbelt/bwrap** (`packages/omo-senpi/src/components/memory/sandbox-platform.ts`)
  - macOS: `sandbox-exec -p <SBPL> -- <inner>`; `buildDarwinProfile`이 `(allow default)`, `(deny file-write*)`, writableDirs만 `allow file-write*`, `foreignRoots`는 `deny file-read*`로 구성. `/dev/null`, `/dev/tty` 예외. TMPDIR를 `.sandbox-tmp`(0700)로.
  - Linux: `bwrap --ro-bind / / --dev-bind /dev /dev --tmpfs /tmp --bind <writable> <writable> --chdir <cwd> -- <inner>`.
  - 정책 `required|auto|off`(기본 `auto`): `required`인데 sandbox-exec/bwrap 부재면 `SandboxUnavailableError`, `auto`면 unsandboxed + warning. Linux는 아직 없는 lock path를 bwrap으로 grant할 수 없어 required면 실패(`resolveLockPaths`).
  - bwrap 사용가능성 probe: `sandbox-bwrap-probe.ts`; 계약/경로: `sandbox-contracts.ts`, `sandbox-paths.ts`.
- **권한/승인**: OpenCode `permission` 시스템을 그대로 상속. 어댑터 쪽에서는 에이전트별 tool restriction(`packages/omo-opencode/src/agents/tool-restrictions`)과 Prometheus md-only 훅으로 표면 제한.
- **컨테이너 지원 없음.** `packages/omo-opencode/src/help/schema/sandbox.ts`는 CLI help용 JSON 스키마(`SandboxConfig`: enabled/timeout/memory/network/filesystem read·write·tempDir)로, 런타임 격리 구현이 아님.
- **개발용 QA 격리**: XDG 임시 디렉토리 + 별도 `CODEX_HOME`을 쓰는 `script/agent/qa-sandbox.sh`(호스트 `~/.config/opencode`·`~/.codex` 오염 방지).

## pi-mono
- **내장 OS 샌드박스 없음.** 격리 방법을 문서로 제시하고 외부 도구에 위임 (`packages/coding-agent/docs/containerization.md`).
  - **Plain Docker**: `Dockerfile.pi`(node:24-bookworm-slim + `@earendil-works/pi-coding-agent`), `docker run -e KEY -v $PWD:/workspace`.
  - **Docker Sandboxes(sbx)**: 관리형 샌드박스 + 프록시가 실제 provider 자격증명을 호스트에 남기고 placeholder로 대체(`sbx secret set-custom`).
  - **NVIDIA OpenShell**: 로컬/원격 샌드박스, filesystem/process/network/credential/inference 정책(`openshell sandbox create --from pi`).
  - **Gondolin**: 로컬 Linux micro-VM 확장(`examples/extensions/gondolin/index.ts`) — 호스트 working folder를 `/workspace`에 마운트하고 `read/write/edit/bash/grep/find/ls`를 VM으로 위임. QEMU + Node 23.6+ 필요. 호스트 env 상속에 자격증명 노출 주의.
  - 샌드박스 확장 예시: `examples/extensions/sandbox/index.ts`.
- **권한/승인 전용 엔진 없음** — tool_call/tool_result 훅 파이프라인은 있으나 permission 엔진은 미확인. 격리는 컨테이너/VM에 위임.

## oh-my-pi
- **OS 샌드박스 없음. 승인(approval) 엔진이 격리 대체** (`docs/approval-mode.md`).
- **3입력 모델**: ① 도구 선언 `approval` tier(`read|write|exec`, 미선언은 `exec`가 안전 기본), ② 도구 policy(`allow|deny|prompt` + `override`/`reason`/`policyKey`), ③ 사용자 `tools.approval.<toolName>`.
- **모드** `tools.approvalMode`: `always-ask`(read만 자동) / `write`(read·write 자동) / `yolo`(전부 자동, 기본). CLI `--approval-mode`, `--auto-approve`, `--yolo`.
- **안전 override**: bash 파괴 패턴(`rm -rf /`, fork bomb, remote-fetch-then-exec, `/etc/passwd`, shutdown)은 강제 prompt; `bash.patterns`의 `deny`(절대)/`prompt`/`allow`; `bash.allowCompoundCommands`(기본 off, `&&` 플랫 체인만). provider `pendingSafetyChecks`는 yolo여도 prompt.
- **ACP**: `session/request_permission`으로 클라이언트 게이트; 서브에이전트는 headless `yolo`(parent `task` 승인이 경계).
- 구현: `packages/coding-agent/src/tools/approval.ts`, `tools/settings.ts`, `task/executor.ts`, `cursor.ts`, `cli/args.ts`.
- **컨테이너/FS 격리 없음** — 문서가 "승인은 프로세스/FS containment가 아니다"라고 명시(`bash`는 ambient FS/network/subprocess 접근 유지).

## codex
- **OS 샌드박스 = `codex-rs/sandboxing/` 크레이트** (`src/lib.rs`): macOS Seatbelt(`seatbelt.rs`), Linux Landlock(`landlock.rs`) + bubblewrap(`bwrap.rs`, 별도 `codex-rs/bwrap` 크레이트), Linux PID namespace(`linux_pid_namespace.rs`), Windows restricted token + Windows Sandbox MXC(`windows.rs`, `windows_mxc`), 전용 Windows 서비스 `codex-rs/windows-sandbox-service/`. `SandboxManager`, `SandboxType`, policy transforms, `violation.rs`/`denial.rs` 위반 감지, `spawn.rs`. WSL1 bubblewrap은 미지원 경고.
- **정책/승인**: `SandboxMode` = `ReadOnly|WorkspaceWrite|DangerFullAccess`(`codex-rs/protocol/src/config_types.rs:104`); `SandboxPolicy`(`protocol/src/protocol.rs`: `ReadOnly{network_access}`, `WorkspaceWrite{writable_roots,network_access,...}`, `DangerFullAccess`) → `PermissionProfile`(`protocol/src/permissions.rs`). `approval_policy = AskForApproval`. 코어 어댑터: `codex-rs/core/src/sandboxing/mod.rs`, `core/src/safety.rs`.
- **네트워크**: `codex-network-proxy` managed network sandbox, `network_policy_decision`.
- **설정 키**: `sandbox_mode`, `approval_policy`, `sandbox_workspace_write.*`, permission profile. `compatibility_sandbox_policy_for_permission_profile`로 레거시 호환.
- **컨테이너 지원 없음**(원격 exec server 프로토콜 `codex-rs/exec-server-protocol/`은 별도 경로).

## claude-code
- **OS 샌드박스 = Bash 샌드박스 설정** (`examples/settings/settings-bash-sandbox.json`, `settings-strict.json`): `sandbox.enabled`, `autoAllowBashIfSandboxed`, `allowUnsandboxedCommands`, `excludedCommands`, `enableWeakerNestedSandbox`; `sandbox.network.{allowUnixSockets,allowAllUnixSockets,allowLocalBinding,allowedDomains,httpProxyPort,socksProxyPort}`.
- **권한/승인**: `PermissionMode` = `default|acceptEdits|bypassPermissions|plan|dontAsk|auto`(`mods/types/claude-code.d.ts:6076`); `auto`는 모델 classifier가 승인/거부. `permissions.{allow,ask,deny}`, `disableBypassPermissionsMode`, `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`. `--permission-mode`, `acceptEdits`, sandbox override 플래그(`allowDangerouslySkipPermissions`).
- **설정 키**: `permissions`, `sandbox`, `allowManagedPermissionRulesOnly`.
- **컨테이너 지원 없음.**

## openclaw
- **샌드박스 백엔드 시스템** (`src/agents/sandbox/`): backend registry/manager/factory(`backend.ts`, `registry.ts`, `manage.ts`), **Docker 기본**(`docker-backend.ts`, `docker.ts`, `docker-user.ts`, `docker-partial-cleanup.ts`), **Podman**(`podman-runtime.ts`), **remote-shell/SSH**(`ssh-backend.ts`, `remote-shell-backend.ts`, `remote-shell-transport.ts`, `ssh.ts`), **브라우저 샌드박스 컨테이너**(`browser.ts`, `browser-network.ts`, `novnc-auth.ts`), `container-engine.ts`, `container-lifecycle.ts`, `container-inspect.ts`.
- **FS 격리**: `readOnlyRoot` 기본 `true`, `network` 기본 `"none"`(`config.ts`); `validate-sandbox-security.ts`가 bind mount 소스 검증 — 시스템 루트(`/`), 자격증명 경로, Docker socket, reserved container target 거부, `allowedSourceRoots` 밖 차단. `workspace-mounts.ts`(read-only skill mount), `mount-plan.ts`, `fs-bridge*`.
- **네트워크 격리**: `network-mode.ts`가 `host` 및 `container:` namespace join을 위험으로 차단(`allowContainerNamespaceJoin`으로만 우회), `sanitize-env-vars.ts`.
- **권한/승인**: `tool-policy.ts`(`isToolAllowed`), `exec-filesystem-policy.ts`, run authority/approval capability closure(AGENTS.md `Run Authority`), 보안 감사 모듈(`security/audit-sandbox-docker-config*`, `audit-sandbox-browser*`, `audit-exec-sandbox-host*`, `audit-exec-surface*`, `dangerous-config-flags*`), `secret-owner.ts`.
- **설정 키**: `agents.defaults.sandbox.backend`(`docker|ssh`, 기본 docker), `sandbox.docker.{image,binds,tmpfs,readOnlyRoot,network}`, `sandbox.ssh.{command,workspaceRoot}`(기본 `/tmp/openclaw-sandboxes`), `sandbox.browser.{image,network}`. 공개 SDK: `src/plugin-sdk/sandbox.ts`.

## hermes-agent
- **터미널 실행 백엔드 6종** (`tools/environments/`): `BaseEnvironment` ABC + `local.py`, `docker.py`, `ssh.py`, `modal.py`(direct) + `managed_modal.py`(Nous managed, `auto` 선택), `daytona.py`, `singularity.py`, `vercel_sandbox.py`. 선택은 `TERMINAL_ENV`(`terminal_tool._create_environment`). 샌드박스 저장소 루트 `TERMINAL_SANDBOX_DIR`(기본 `{HERMES_HOME}/sandboxes`).
- **네트워크 egress**: Docker 백엔드의 iron-proxy 자격증명 방화벽(`tools/environments/docker_egress.py`) — `proxy.enabled`, `proxy.enforce_on_docker`(기본 true: 반쪽 구성이면 샌드박스 시작 거부), `docker_forward_env`/`docker_env`/`docker_extra_args`가 프록시를 무력화하지 못하도록 가드. 프록시 소스: `agent/proxy_sources/iron_proxy.py`.
- **권한/승인**: `tools/approval.py` + `agent/terminal_approval_batch.py` — 세션별 gateway 승인 큐, `approvals.mode`(`smart`/`yolo`/off), allowlist, approve-once/session/always. 다중 프로필 자격증명 격리는 `agent/secret_scope.py`(fail-closed `UnscopedSecretError`, 프로필 간 `.env` 비혼합).
- **FS 방어(명시적 비보안경계)**: `agent/file_safety.py` — 쓰기/읽기 deny 목록(`~/.ssh`, `.aws`, `.env`, `auth.json` 등), Windows NT/device namespace(NTLM leak) 가드, `HERMES_WRITE_SAFE_ROOT`, sandbox/container mirror soft guard; `agent/runtime_self_protection.py`. 플러그인 격리: `hermes_cli/plugin_isolation.py`.
- **nix/스크립트**: `nix/sandbox.nix`(샌드박스 데스크톱 런처), `scripts/dev-sandbox.sh`, `docker/sandbox-desktop.Dockerfile`.
- **설정 키**: `terminal.*`(backend/image), `docker_image`/`modal_image`/`daytona_image`/`singularity_image`(마이그레이션, `hermes_cli/config_migrations.py`), `proxy.*`, `HERMES_WRITE_SAFE_ROOT`, `TERMINAL_SANDBOX_DIR`.
