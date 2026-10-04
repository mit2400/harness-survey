# Git Workflow & Worktree Isolation 비교 (8개 AI Coding Harness)

> 대상: `/home/minkoo/study/repos/` — opencode, oh-my-openagent, pi-mono, oh-my-pi, codex, claude-code, openclaw, hermes-agent
> 기준: 실제 소스/문서 grep 결과 (2026-10)

## 요약 비교

| 하니스 | worktree 격리 | git 도구 | PR 흐름 | 특이사항 |
|---|---|---|---|---|
| **claude-code** | ✅ 풀 기능 (EnterWorktree/ExitWorktree, baseRef, sparsePaths, bgIsolation, keep/remove 정리) | git status/diff/commit + hook 연동 (WorktreeCreate → hookSpecificOutput.worktreePath) | `plugins/pr-review-toolkit` 번들 | isolation 모드: `worktree` vs `remote` 선택 가능 |
| **codex** | ✅ 전용 Rust 크레이트 `codex-rs/worktree/` + 설정 `worktrees` / `new_worktree` | dirty worktree 인식 프롬프트, linked worktree 메타데이터, trust 검증 | 세션 기반 (PR 전용 스킬 없음) | "You may be in a dirty git worktree" 프롬프트 내장 |
| **openclaw** | ✅ managed worktrees 개념 (docs/concepts/managed-worktrees.md) | 게이트워크/클라우드 워커 연동 | — | 세션·워커 수명주기와 결합 |
| **hermes-agent** | ✅ 가장 성숙 (create/GC/security/sync-base/self-heal, subagent worktree) | git_probe, status_bar_git, web git 라우터 | kanban worktree isolation/teardown | worktree 전용 CLI 서브커맨드 + 테스트 20+ |
| **oh-my-openagent** | ✅ 팀 모드 member worktree (`"worktreePath": "../wt-scout"`) | opencode에 위임 | — | tmux 페인을 member worktree에 연결 |
| **pi-mono** | ❌ worktree 격리 없음 | git 기반 snapshot (`packages/coding-agent/src/snapshot/`) | — | 체크포인트 = git snapshot만 제공 |
| **oh-my-pi** | ✅ 풀 기능 (isolation-runner, ownership, session-worktree, bash-worktree-rewrite) | gh-pr-checkout, repo-lock, autoresearch/git | PR 체크아웃 도구 | 소유권(isolation-ownership) 개념 |
| **opencode** | ❌ 사용자 격리 없음 (내부 snapshot에만 worktree 사용) | snapshot/index.ts가 `--git-dir/--work-tree`로 git 스냅샷 | — | worktree는 체크포인트 구현 세부사항 |

---

## claude-code

- **격리**: `repos/claude-code/mods/types/claude-code.d.ts` — `isolation?: "worktree" | "remote"` (12398행). EnterWorktree/ExitWorktree 도구가 worktree를 생성·진입·종료하며, `worktree.baseRef`, `sparsePaths`, `bgIsolation` 개념 포함. 정리 정책은 `"keep"`(worktree+branch 잔존) / `"remove"`(삭제), 미커밋 파일 있으면 `force` 필요 (12444–12446행).
- **훅 연동**: `WorktreeCreate` 훅이 `hookSpecificOutput.worktreePath`로 생성된 worktree를 반환하고, 미설정 시 세션이 자동 생성 (1046–1076행).
- **git 도구**: 일반 git status/diff/commit + `gh` CLI. PR은 `plugins/pr-review-toolkit/` 번들 플러그인으로 처리.
- **특이사항**: `isolation: "remote"`는 항상 background로 실행되며 가용성이 게이트됨 — worktree 격리와 원격 격리를 같은 스펙으로 통합.

## codex

- **격리**: 전용 Rust 크레이트 `repos/codex/codex-rs/worktree/` (`src/lib.rs`, `git.rs`, `paths.rs`, `settings.rs`, `metadata.rs`, `tests/worktree.rs`). 설정 스키마 `codex-rs/core/config.schema.json`에 `worktrees` 섹션과 `new_worktree` ("Open a new session in a worktree from the project default branch") 옵션 존재.
- **git 도구**: linked worktree 메타데이터 기록/신뢰 검증 (`core/src/worktree_trust_tests.rs` — `worktree add --detach`, `worktree move`, `worktree repair` 테스트). 모델 프롬프트에 "You may be in a dirty git worktree" 명시 (`gpt_5_codex_prompt.md` 등).
- **PR 흐름**: 전용 PR 리뷰 스킬 없음 — 세션 내 git/gh 작업에 의존.
- **특이사항**: worktree를 신뢰 경계(trust boundary)로 취급해 메타데이터 기반 검증을 수행.

## openclaw

- **격리**: `docs/concepts/managed-worktrees.md` — managed worktree 개념 문서화. `docs/gateway/sandboxing/workspace-access.md`, `docs/gateway/cloud-workers/` 세션 수명주기와 결합.
- **git 도구**: 게이트워크 수준에서 워커 배치/체크아웃 관리 (`docs/ci/checkout.md`, `docs/gateway/cloud-workers/dispatching-a-session.md`).
- **특이사항**: worktree가 단일 에이전트 격리보다 클라우드 워커·세션 오케스트레이션의 하위 개념으로 위치.

## hermes-agent

- **격리**: 가장 구현이 성숙. `hermes_cli/worktree_cmd.py`, `hermes_cli/worktree_ops.py`, `hermes_cli/worktree_gc.py`, `hermes_cli/subcommands/worktree.py`, `tools/subagent_worktree.py`. 테스트: `tests/hermes_cli/test_worktree.py`, `test_worktree_gc.py`, `test_worktree_security.py`, `test_worktree_sync_base.py`, `test_worktree_selfheal.py`, `test_kanban_worktree_isolation.py`, `test_kanban_worktree_teardown.py`.
- **git 도구**: `tui_gateway/git_probe.py`, `hermes_cli/status_bar_git.py`, `hermes_cli/web_routers/git.py`, `tools/checkpoint_manager.py`.
- **PR 흐름**: kanban worktree isolation/teardown로 작업 단위 분리.
- **특이사항**: GC(가비지 수집), base 동기화, self-heal, 경로 보안 검증까지 worktree 수명주기 전부를 전용 CLI로 관리.

## oh-my-openagent

- **격리**: `docs/guide/team-mode.md` — 팀 모드 member에 `"worktreePath": "../wt-scout"` 설정 (111행). 파일시스템 상대/절대 경로만 허용, bare branch 이름 거부, `git` 필요. `team_delete`가 runtime state·worktrees·tmux layout 제거 (88행).
- **git 도구**: 실제 git 조작은 각 member가 사용하는 opencode에 위임.
- **특이사항**: tmux 페인을 `opencode attach`로 member worktree에 연결해 스트리밍 출력 관찰 (117행) — 격리 + 가시성 결합.

## pi-mono

- **격리**: **worktree 격리 기능 없음.** `packages/coding-agent/src/snapshot/`은 worktree 매치 0건 — git 기반 snapshot(체크포인트)만 제공. worktree 언급은 `packages/durable/docs/spec.md`, `footer-data-provider.ts`(표시 용도), 스크립트에 한정.
- **git 도구**: snapshot 메커니즘으로 git 활용.
- **특이사항**: 상태 복원 = git snapshot 수준. agent 격리 모델이 없어 동시 작업 격리는 외부에 의존.

## oh-my-pi

- **격리**: `packages/coding-agent/src/commands/worktree.ts`, `task/worktree.ts`, `task/isolation-runner.ts`, `task/isolation-ownership.ts`, `session/session-worktree.ts`, `cli/worktree-cli.ts`. bash 명령의 worktree 경로 재작성(`tools/bash-worktree-rewrite.ts`)으로 격리 투명성 확보.
- **git 도구**: `tools/gh-pr-checkout.ts` (PR 체크아웃), `utils/repo-lock.ts` (동시성 제어), `autoresearch/git.ts`.
- **PR 흐름**: gh-pr-checkout 도구로 PR 기반 작업 지원.
- **특이사항**: isolation-ownership — 격리 작업의 소유권을 추적해 중복 격리/누출 방지.

## opencode

- **격리**: **사용자 대상 worktree 격리 없음.** `packages/opencode/src/snapshot/index.ts`가 내부적으로 git worktree를 사용: gitdir를 `data/snapshot/<project>/<hash>`에 두고 `--git-dir`/`--work-tree` 인자로 git 명령 실행 (70–75행), `read-tree`로 스냅샷 복원 (386행), 대형 worktree 튜닝 주석 (332행).
- **git 도구**: snapshot = 체크포인트 구현 수단.
- **특이사항**: worktree는 agent 격리가 아니라 상태 스냅샷의 구현 세부사항 — 사용자가 진입/종료하는 개념 아님.

---

## 관찰 / 시사점

1. **두 가지 사용 패턴**: (a) agent 격리 수단으로의 worktree (claude-code, codex, hermes-agent, oh-my-pi, oh-my-openagent, openclaw) vs (b) 내부 체크포인트 구현 세부사항 (opencode) vs (c) 아예 없음 (pi-mono).
2. **성숙도 차이**: hermes-agent가 GC·self-heal·security·sync-base까지 전용 CLI로 가장 완전하고, claude-code/codex/oh-my-pi가 그 뒤를 잇는다. codex는 Rust 크레이트 + trust 검증으로 엔지니어링 무게가 높다.
3. **git 연동 공통점**: 대부분 dirty worktree 상태를 프롬프트/신뢰 경계로 명시하고, `gh` CLI를 PR 진입점으로 사용. 단독 PR 리뷰 스킬은 claude-code(pr-review-toolkit)와 oh-my-pi(gh-pr-checkout)에만 뚜렷하다.
4. **체크포인트 vs git 커밋**: opencode/pi-mono의 snapshot은 git 객체를 직접 다루는 반면, hermes-agent의 checkpoint_manager는 worktree 수명주기와 분리된 독립 계층 — rewind 의미가 harness마다 다르므로 비교 시 주의.
5. **팀/동시 격리**: oh-my-openagent(member worktree + tmux)와 oh-my-pi(isolation-ownership)만이 "여러 agent가 같은 repo를 동시에 다루는" 시나리오를 1등 시민으로 다룬다.
