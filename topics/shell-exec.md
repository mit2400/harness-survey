# 셸/커맨드 실행 비교

## 요약

| 하니스 | 셸 구현 | 출력 제한 | 백그라운드/커널 | 특이사항 |
|---|---|---|---|---|
| opencode | `bash` 툴 + platform shell 해석 (`core/src/shell.ts`) | 2000줄 / 50KB, 캡처 최대 1MiB, spill 7일 | 배경은 `task` 툴 전용, cron 없음 | codemode와 모델 출력 예산 분리 |
| oh-my-openagent | tmux 기반 `interactive_bash` 한 툴 | 동적 truncator 최대 50K 토큰 (남은 컨텍스트 50%) | background task 4종 + 동시성 5 + 폴링 2~3초 | Windows는 `git-bash-mcp`, 컨텍스트 소진 시 `[Output suppressed…]` |
| pi-mono | `bash`/`powershell` 툴, `BashOperations` 셀 | 2000줄 / 50KB tail, rolling 102KB, structured 1MiB | **없음** (detached PID 추적만), cron 없음 | **승인 게이트 전무**, extension에 전면 위임 |
| oh-my-pi | brush 기반 Rust 셸 (`crates/pi-shell`) + PTY | 3000줄 / 50KB / 열 512, artifact 16MiB | auto-background + `:async:` + `wait` (최대 30분) | eval 툴 = 영구 IPython/Bun 커널, `CRITICAL_BASH_PATTERNS` |
| codex | 통합 `exec` (`unified_exec`), zsh fork 별도 | 기본 10K 토큰, 1MiB 바이트 캡 | `yield_time_ms` 10초, `sleep` 12h, heartbeat | seatbelt 샌드박스 + `is_dangerous_command` 트리 |
| claude-code | 닫힌 바이너리, 스키마만 공개 (`Bash`/`PowerShell`) | 30K 인라인 → 디스크 영속 (조정 최대 128K) | `run_in_background` 30분/2h + CronCreate + ScheduleWakeup | sandbox는 Bash 전용, 5가지 배경화 원인 |
| openclaw | `exec` + `process` 툴, 4 host | 모델 표시 200K자, 폴링 30K자, 프로세스 16MB | yieldMs 10초, TTL 30분(최대 3h), process 10액션 | 3축 승인 정책 + code-mode 커널 64MB |
| hermes-agent | `terminal` 툴 + **8 백엔드** | 50KB/2000줄, 40:60 head-tail, spill 5MB | `process_manage` 64개 + 크론 32모듈 + **영구 Python 커널** | Tirith + `DANGEROUS_PATTERNS` 결합 승인 |

## opencode
- 툴명 **`bash`** (`packages/core/src/tool/bash.ts`): `DEFAULT_TIMEOUT_MS=120_000`, `MAX_TIMEOUT_MS=600_000`, `MAX_CAPTURE_BYTES=1MiB`.
- 별도 `shell` 툴 (`packages/opencode/src/tool/shell.ts`, 645줄): `MAX_METADATA_LENGTH=30_000`, cwd/파일/커맨드 파일 클래스 집합. 종류는 `bash|pwsh|powershell|cmd` (`shell/id.ts`)인데 ToolID는 항상 `"bash"`.
- 절단 (`packages/opencode/src/tool/truncate.ts`): `MAX_LINES=2000`, `MAX_BYTES=50*1024`, head/tail 방향 지정, 초과분은 `TRUNCATION_DIR`로 스파일(7일 보존), `tool_output.max_lines/max_bytes`로 설정 가능. 한도를 모델에게 고지하는 문구는 `shell/prompt.ts` (PowerShell/cmd 변형 포함).
- 셸 해석 (`packages/core/src/shell.ts` META 표): fish/nu는 `deny`, bash/dash/ksh/sh/zsh는 login+posix, powershell/pwsh는 `ps`, `acceptable` 플래그. win32 종료는 `taskkill /f /t`, darwin은 git-bash 폴백.
- 백그라운드: **`task` 툴만** (`packages/opencode/src/background/job.ts`, `BackgroundJob`). cron 없음.
- codemode 예산 (`packages/codemode/src/codemode.ts`): `ExecutionLimits{timeoutMs, maxToolCalls, maxOutputBytes}` (없으면 무제한), `DiscoveryOptions.catalogBudget` 기본 2000 토큰.

## oh-my-openagent (OmO)
- **tmux 전용 `interactive_bash`** 한 툴 (`packages/omo-opencode/src/tools/interactive-bash/`): `DEFAULT_TIMEOUT_MS=60_000`, `BLOCKED_TMUX_SUBCOMMANDS` (capture-pane/save-buffer/pipe-pane), `PROHIBITED_TMUX_SUBCOMMANDS` (kill-server), cmux 호환.
- 출력 (`hooks/tool-output-truncator.ts`): `DEFAULT_MAX_TOKENS=50_000` (~200K자), webfetch는 10K, `TRUNCATABLE_TOOLS` 목록, `truncate_all_tool_outputs` 플래그. `shared/dynamic-truncator.ts`: `maxOutputTokens = min(remainingTokens*0.5, 50_000)`, 컨텍스트 소진 시 `[Output suppressed - context window exhausted]`.
- 백그라운드: `create-background-{task,output,cancel}.ts` (`bg_` ID, `POLL_INTERVAL_BACKGROUND_MS=2000`, `SESSION_TIMEOUT_MS=60분`) + `features/background-agent/` (모델/프로바이더당 동시성 5, `task-poller` 3초, `concurrency.ts`, `loop-detector.ts`).
- Windows: `packages/git-bash-mcp/` — Windows 전용 `git_bash` MCP 툴군(Codex 에디션용), `git-bash-resolver.ts`. OpenCode 에디션은 tmux `interactive_bash`를 쓰므로 POSIX 기반.

## pi-mono
- `bash` 툴 (`packages/coding-agent/src/core/tools/bash.ts`): `spawn(shellConfig.shell, …, { detached: platform!=="win32", windowsHide: true })` (:112-118). **기본 타임아웃 없음**, `MAX_TIMEOUT_MS=2^31-1`.
- 유일한 확장 지점 **`BashOperations`** (:75-94)와 **`BashSpawnHook`** (:186-225) — 명령/cwd/env를 spawn 직전 재작성 가능. `packages/durable/src/env/node.ts:459-691`에 완전히 복제된 두 번째 구현.
- 출력 (`truncate.ts`): `DEFAULT_MAX_LINES=2000`, `DEFAULT_MAX_BYTES=50*1024`, `GREP_MAX_LINE_LENGTH=500`. bash는 **tail 우선**(에러 생존). `output-accumulator.ts`: rolling `maxBytes*2`=102,400, 초과 시 `tmpdir()/pi-bash-<hex>.log` 스파일. `STRUCTURED_OUTPUT_MAX_BYTES=1MiB` (호출자용 별도 상한). 바이너리 제어문자 정규식 제거 (`utils/shell.ts:158-161`).
- 백그라운드: **1차원 잡 없음.** `trackedDetachedChildPids` 집합 + 종료 시 `killProcessTree` (win32: `taskkill /F /T /PID`, POSIX: `process.kill(-pid)`). 크론 없음 (`durable/src/tasks.ts`는 8줄 레지스트리).
- 승인 게이트 **전무**: allow/deny 목록·sudo 탐지·승인 다이얼로그 없음. 안전은 extension(`examples/extensions/sandbox/` — `denyRead ~/.ssh ~/.aws ~/.gnupg`, `denyWrite .env *.pem`)과 디렉터리 신뢰 프롬프트에 위임. `user_bash` 이벤트로 `operations` 교체 가능.
- codemode (`packages/codemode/`): QuickJS-WASM, **매 `execute()`마다 fresh worker+VM** (`runtime/host.ts:81-86`), 기본 300초/256MiB/10K 출력 토큰/인-VM 16MiB·100K item. 상태는 `store`/`load`뿐, Python·Jupyter·`eval` 없음. PowerShell 툴은 win32 전용.

## oh-my-pi
- **brush 기반 Rust 셸**: `crates/pi-shell/src/{shell.rs,process.rs,cancel.rs,windows.rs,output_decode.rs,minimizer}` + `crates/vendor/brush-{core,parser}`, 빌트인 대량 (`crates/pi-builtins/src/` — `eval.rs`, `jobs.rs`, `bg.rs`, `fg.rs`, `nohup.rs`).
- `bash` 툴 (`packages/coding-agent/src/tools/bash.ts`, 1559줄): **`CRITICAL_BASH_PATTERNS`** — `rm -rf /`, `--no-preserve-root`, `sudo rm`, `chmod -R … /`, 포크밤, `mkfs`, `dd of=/dev/`, `shred`, `cryptsetup`, `>/etc/passwd|shadow|sudoers`, curl/wget 파이프→셸. 세그먼트별 deny/prompt 정책. `approval.ts` (모드 `always-ask|write|yolo`, 정책 `allow|deny|prompt`).
- 출력 (`packages/tui/src/tools/streaming-output.ts`): `DEFAULT_MAX_LINES=3000`, `DEFAULT_MAX_BYTES=50*1024`, `DEFAULT_MAX_COLUMN=512`, `ARTIFACT_DEFAULT_MAX_BYTES=16MiB` 스파일 (`output-meta.ts`/`settings.ts`의 `cfgToolsArtifact*`), `TailBuffer`, `truncateForPrompt`.
- 백그라운드: `cfgBashAutoBackgroundEnabled` 자동 배경 전환, `:async:` 셸, `wait.ts` (`WAIT_MAX_MS=30*60_000`). 크론 없음. TUI 앱용 `bash-interactive.ts` (PTY) + `bash-worktree-rewrite.ts`.
- **eval 툴** (`tools/eval.ts` 1164줄 + `src/eval/`): 영구 IPython(py)/Bun(js) 커널, `%load`/`%pip`/`%bun`/`%environment`, 커널에서 `agent()`/`workpool()`로 서브에이전트 스폰, `judge()`/`judge_batch()` 롤 체인(`@smol`, `@default`), `completion()`, `budget.*`. 프롬프트는 `prompts/tools/eval{,-agents,-judge,-helpers,-code-mode}.md`.

## codex
- 통합 exec: `codex-rs/core/src/tools/handlers/unified_exec.rs` + `unified_exec/{exec_command,write_stdin}.rs`, 모듈 `core/src/unified_exec/mod.rs`. 인자 `cmd/shell/login/tty/yield_time_ms(기본 10_000)/timeout_ms/max_output_tokens/sandbox_permissions/justification/prefix_rule`.
- `DEFAULT_MAX_OUTPUT_TOKENS=10_000`, `format_output_omission_marker` ("... {n} bytes omitted ..."), 바이트 캡 `EXEC_OUTPUT_MAX_BYTES=DEFAULT_OUTPUT_BYTES_CAP=1MiB` (`utils/pty/src/lib.rs:25`), `READ_CHUNK_SIZE=8192`, `MAX_EXEC_OUTPUT_DELTAS_PER_CALL=10_000`.
- **zsh fork**: `UnifiedExecShellMode::ZshFork` (`unified_exec.rs:122` — 로컬에서 `shell` 파라미터 미지원), `Feature::ShellZshFork` (`session/session.rs:1384`), `config.zsh_path`, `bundled_zsh_path()`는 **Windows에서 `None`** (`install-context/src/lib.rs:232`), 플래그 `unified_exec_zsh_fork`.
- **예산 분리** (`core/src/tools/context.rs`): `model_output_policy()`는 요청 토큰과 절단 정책의 최소, `code_mode_result()`는 원본 또는 `max_output_tokens` → 모델 vs codemode가 다른 상한. `code-mode-protocol/src/runtime.rs:17` 기본 10K 토큰/호출.
- 안전: `shell-command/src/command_safety/is_dangerous_command.rs` (323줄, sudo는 내부 명령 재검사, `MAX_DANGEROUS_COMMAND_WRAPPER_DEPTH=8`), `powershell_parser.rs`/`powershell_tree_sitter.rs`/`windows_dangerous_commands.rs` (771줄), `core/src/exec_policy.rs`. 샌드박스 `SandboxType::MacosSeatbelt` (`/usr/bin/sandbox-exec`) + `execpolicy` 크레이트.
- 스케줄링: `handlers/sleep.rs` (`MAX_SLEEP_DURATION_MS=12h`), heartbeat 스케줄러 (`history/src/heartbeat.rs`). 셸 타입 `core/src/shell.rs` (bash/zsh/powershell/cmd), PowerShell은 `shell_snapshot.rs`로 `.ps1` 스냅샷.

## claude-code
- 저장소는 **docs/plugin 미러** — 실제 구현은 닫힌 네이티브 바이너리 (`CHANGELOG.md:5048`). 신뢰 가능한 원본은 `mods/types/claude-code.d.ts` (툴 input/output 스키마 원문)과 `CHANGELOG.md`.
- 스키마 (:12400-12411): `Bash { command, timeout (max 600000 ms), description, run_in_background, dangerouslyDisableSandbox }`, `PowerShell`은 동일 형태 (:12467-12478). 결과 (:12690-12771): `stdout/stderr/rawOutputPath/interrupted/isImage/backgroundTaskId/timedOutAfterMs/persistedOutputPath/{bashEditDiff,gitOperation}`.
- 출력 이력: 30K 인라인 한도(`CHANGELOG.md:6926`) → **디스크 영속으로 전환** (:6872, 디스크 1GB 캡 :1766), `bashOutputMaxChars`로 최대 128K 조정 (:1822), `TaskOutput` 제거 후 `Read`로 읽음 (:1095). diff 뷰는 `MAX_LINES_PER_FILE=400`, 호스트 프로세스 `$.process.run`은 `HOST_OUTPUT_CAP_BYTES=4MiB` (`mods/diff/hooks/limits/sizes/…`).
- 백그라운드: `run_in_background`, **timeout 기본 30분/최대 2시간** (:381). 결과 스키마에 5가지 배경화 원인 (Ctrl+B, turn-abort, 메시지 전달, 타임아웃 자동, 서브에이전트 종료). `!` 셸 모드는 샌드박스 **밖** (:1947).
- 스케줄링: `CronCreate/CronDelete/CronList` 완전 스키마 (5필드 크론, `durable`은 `.claude/scheduled_tasks.json`, **7일 자동 만료** :12418), `ScheduleWakeup`(60~3600초 클램프), `Monitor`(최대 30분, `-p`는 10분).
- 영구 코드 실행 **없음** (23개 내장 툴에 `execute_code` 없음; `Workflow`가 유일한 스크립트형).
- 안전: `PermissionMode = 'default'|'acceptEdits'|'bypassPermissions'|'plan'|'dontAsk'|'auto'` (:6074), `ToolCheckDecision = 'allow'|'ask'|'deny'` (:9915). **sandbox는 Bash 전용** (`examples/settings/README.md:27`). dangerous-rm 하드닝 지속 (변수 치환 :758, `BASHPID` 산술대입 :32, `excludedCommands`는 합성명령 전부 매치 :1073, Read/Edit deny가 Bash argv로 확장 :1333 등). Windows: PowerShell은 독립 툴이지만 Bash 거부 시 함께 꺼진다는 경고 (:102), `CLAUDE_CODE_GIT_BASH_PATH`.

## openclaw
- 툴명 **`exec`** (`src/agents/bash-tools.exec-run.ts:187`) + **`process`** (`bash-tools.process.ts:309`). `bash-tools.*` 파일 prefix는 유산 명칭. 세 번째 `terminal` 툴은 Control UI PTY 관리용.
- 셸 자동자원 (`src/agents/shell-utils.ts`): POSIX는 `bash --noprofile --norc -c` / `zsh -f -c` / `fish --no-config -c`, Windows는 **pwsh 우선**(PS 5.1은 `&&` 미지원), Git Bash 캐시(16), `NON_INTERACTIVE_SHELLS`. 호스트는 `auto|sandbox|gateway|node` 4종.
- 출력: **모델 표시 `DEFAULT_MAX_OUTPUT=200,000자`** (`bash-tools.exec-runtime.ts:104-109`), 폴링 `DEFAULT_PENDING_MAX_OUTPUT=30,000자` (:111-116), 프로세스 primitive 16MB (`src/process/exec-output.ts:31`), `maxBuffer` 1MB, 로그 tail 200줄, notify 180/400자. head/tail/discard 모드, UTF-8 경계 복원(win32는 기본 미적용), 버려진 구간에는 `[earlier output was discarded at the retention cap …]` 표기. 환경변수 `OPENCLAW_BASH_MAX_OUTPUT_CHARS` 등.
- 백그라운드: `yieldMs` 기본 10초, 세션 TTL **30분(최소 1분/최대 3시간)** (`bash-process-registry.ts:18-20`), 완료 세션 50개·2M자 보존. `process` 10액션 (`list|poll|log|write|send-keys|submit|paste|kill|clear|remove`), poll 최대 30초, `notifyOnExit`, `timeoutSeconds` 기본 1800초. **셸 `&` 사용을 문서로 금지** (:433). 크론은 `src/cron/` — 스크립트 페이로드 900초/200 툴콜, 동시 트리거 3, 정각 5분 스태거.
- 안전: **3축 정책** — `ExecSecurity(deny|allowlist|full)` / `ExecAsk(off|on-miss|always)` / `ExecMode(deny|allowlist|ask|auto|full)` (`src/infra/exec-approvals-core.ts:10-12`), ask 폴백은 **deny**, 승인 타임아웃 30분. `isSafeExecutableValue`(메타문자·제어문자 차단), `WINDOWS_UNSAFE_CMD_CHARS_RE=/[&|<>%\r\n]/`, safe-bin 프로필, 모델 기반 auto-reviewer 30초. Codex 하니스는 `extensions/codex/src/app-server/approval-bridge.ts`가 `requestApproval`를 변환 (미리보기 4096/설명 700/명령 80/값 48자).
- code-mode 커널: 툴명도 `exec`이지만 `code_mode_exec`로 구분 + `wait`. Node worker와 QuickJS 이중, 기본 10초/64MB/64KB 출력/10MB 스냅샷/900초 TTL, headless 30초~900초·5~200 툴콜.
- Windows: `taskkill.exe` 종료 + **Windows Job Objects** (`KILL_ON_JOB_CLOSE`), 콘솔 코드페이지 디코딩, PTY node/bun 이중 백엔드.

## hermes-agent
- **`terminal` 툴** (`tools/terminal_tool.py`, 1639줄) + companion 모듈군 (config/backends/lifecycle/sudo/guards/background/result). 패치는 facade에서 (re-export seam).
- **8 백엔드** (`agent/terminal_env_registry.py:26-29`): `local, docker, singularity, modal, managed_modal, daytona, vercel_sandbox, ssh`. 사용자 선택(`_BUILTIN_BACKENDS`)은 7개 — `managed_modal`은 `modal_mode` 경유. 디스패치는 `_ENV_BUILDERS` + `_BACKEND_SPECS` + 통합 `_create_environment`, 플러그인은 `TerminalEnvironmentProvider`, 빌트인 이름 충돌은 거부.
- 출력 (`tools/tool_output_limits.py`): `max_bytes=50_000`(실제는 문자 수), `max_lines=2000`, `max_line_length=2000`. `truncate_head_tail` **40:60** head-tail + 마커 `… [label TRUNCATED - …] …`. 스트리밍 `_BoundedOutputCollector`도 40:60, 5MB 스파일(`~/.hermes/cache/terminal-output/`, 7일 보존)은 재심사 후 재작성. 바이너리는 `errors="replace"`로 U+FFFD 치환. 후처리 순서: 절단 → ANSI 제거 → 재심사 → 종료코드 노트.
- 백그라운드: **`process_manage`** (`poll/log/wait/kill/write/submit/close/list/handoff`) — `MAX_OUTPUT_CHARS=200_000`, `MAX_PROCESSES=64`, 완료 TTL 1800초, `read_log` 기본 200줄, systemd cgroup 격리(64MiB~4GiB). yield-to-background(local 전용) + `notify_on_complete`. 크론은 `cron/` 32모듈·17,694줄 (`scheduler.py` 4429줄), 스크립트 타임아웃 900초.
- **영구 커널**: `execute_code` (`tools/code_execution_tool.py`) + `code_kernel.py` — 커널당 세션 영속, **시프레임 JSON RPC**(Unix 소켓, Windows는 loopback TCP), 기본 4개 커널·1800초 idle, stdout 50KB/stderr 10KB/스파일 5MB, 툴콜당 300초. 원격은 실패 시 per-call로 폴백.
- 안전: `check_all_command_guards` (`tools/approval.py:1168`)가 **Tirith 스캐너 + `DANGEROUS_PATTERNS`(1548줄)를 하나의 승인 요청으로 결합**. `rm` 플래그 순서 변형(`rm build/ -rf`), IMDS 에nd포인트, `base64 -d | sh`, `xxd -r | sh`, Windows 전용 파괴 패턴(`Remove-Item`, `diskpart`, `Format-Volume` 등). sudo는 세션 키 캐시 + `sudo -S -p ''` 재작성, 프롬프트 45초. bind-mount Docker은 guard 유지, `vercel_sandbox`는 무조건 guard skip.
- Windows: **bash 필수**(Git for Windows), `MSYS2_ARG_CONV_EXCL`, 드레인 select vs `PeekNamedPipe` 이중, `npm.cmd` 호환, WSL 런처.

## 관찰/시사점
- **출력 단위가 제각각**: codex/codemode는 토큰(10K), opencode/pi-mono/oh-my-pi는 줄+바이트(2000~3000줄/50KB), openclaw는 문자(200K), hermes는 문자(50K) + 5MB 스파일, claude-code는 인라인→디스크 영속. 하니스를 옮길 때 "줄수/토큰/바이트" 단위가 다르므로 숫자만 보면 오히려 오해한다.
- **스파일 파일이 사실상 표준**: opencode 7일, hermes 5MB·7일, oh-my-pi 16MiB artifact, pi-mono temp log, openclaw 16MB→2M자 세션, claude-code 1GB. 공통 패턴은 "모델에는 절단본 + 전체 경로".
- **백그라운드는 3파종**: (a) 전용 잡 레지스트리 (opencode `task`, openclaw `process`, hermes `process_manage`), (b) 툴 파라미터로만 (claude-code `run_in_background`, codex `yield_time_ms`, openclaw `yieldMs`), (c) 아예 없음/셸 문법 의존 (pi-mono, oh-my-openagent는 tmux 세션). shell `&`를 못 믿는 것은 openclaw가 명시적으로 문서화.
- **크론 보유**: claude-code(`CronCreate`), openclaw(`src/cron`), hermes(`cron/` 32모듈)만. opencode/pi-mono/oh-my-pi는 없고, codex는 `sleep`+heartbeat로 대체.
- **영구 커널 스펙트럼**: hermes(Python 세션 커널, 진짜 영속) → oh-my-pi(IPython/Bun + 서브에이전트/롤체인, 가장 진보) → openclaw(code-mode 스냅샷, 900초 TTL) → pi-mono/codemode·claude-code(매 실행 새로 생성 또는 부재). "영구"의 정의가 런타임마다 다르다.
- **안전 정책 3단**: 없음(pi-mono) → 패턴 정규식 + 승인(oh-my-pi, hermes, codex) → 정책 모드 + OS 샌드박스(openclaw, claude-code). 공통 진화 방향은 "일회성 패턴 매치 → 위조 회피 방지(변수 치환·래퍼 깊이·합성명령 전부 매치) → 샌드박스 자동 승인".
- **Windows는 전원 POSIX 우선**: hermes는 bash 필수(Git for Windows), pi-mono/opencode는 Git Bash 탐색 + taskkill 트리 킬, OmO는 git-bash MCP. codex의 zsh fork와 oh-my-pi의 PTY는 Windows 폴백이 없거나 제한적.
- **툴명이 하니스마다 다르다** — `bash`(opencode, pi-mono, oh-my-openagent의 interactive_bash) / `exec`+`process`(openclaw, codex) / `terminal`(hermes) / `Bash`+`PowerShell`(claude-code). 탐색 시 grep 대상부터 틀린다.
