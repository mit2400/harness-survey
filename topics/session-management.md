# 세션 생명주기/저장소 비교: 8개 AI 코딩 하니스

## 요약 비교

| 하니스 | 저장 형식 | 트리/fork | resume/share | 특이사항 |
|--------|-----------|-----------|--------------|----------|
| **opencode** | JSONL (append-only) + SQLite (core) | `--fork` 플래그, 트리 구조 | `--continue`, `--session <id>`, revert.ts, snapshot/ | 스냅샷 기반 롤백, compaction, overflow 처리 |
| **pi-mono** | JSONL (append-only 트리) | 트리 구조 (id/parentId), `branch()`, `createBranchedSession()` | `/resume`, `/fork`, `/clone`, `/tree`, `/compact` | 트리 리프 포인터, context_edit, label, branch_summary, UUIDv7 |
| **oh-my-pi** | 다중 백엔드 (JSONL, SQLite, Redis, IndexedDB) | 트리 + worktree 연동 | `/resume`, `/fork`, `/fresh`, `/branch`, `/export`, `/share`, `/handoff`, `/btw` | 94개 세션 파일, 외래 세션 가져오기, 다중 저장소 추상화 |
| **codex** | JSONL (`rollout-*.jsonl`) + SQLite (state_db) | 압축 분기 (compression fork) | `codex resume`, session_index, maintenance | zstd 압축, 날짜 계층 디렉토리, rollout reference index |
| **claude-code** | JSONL (transcript) | fork subagent, `context: fork` 스킬 | `--resume`, `--continue`, `/resume` 피커, SessionResume/SessionSource | SessionStart 훅, 백그라운드 세션, 클라우드 세션 |
| **openclaw** | SQLite (`openclaw-agent.sqlite`) | worktree lifecycle, upstream links | session suspension, session-scopes, handoff | 117개 세션 파일, 상태 이벤트 스트림, diff/revision |
| **hermes-agent** | SQLite (`state.db`) | parent_session_id 계보, compression lineage (재귀 CTE) | `/resume`, `/branch`, `/new`, rewind, auto-archive | 프로필 스코프, delegate/branch 마커, 자동 아카이브 스윕 |
| **oh-my-openagent** | 경로 미확인 (`omo-opencode/src/session/` 없음) | — | — | 저장소 추상화 도구 (session_list/read/search/info)만 확인 |

---

## 1. opencode

**경로**: `packages/opencode/src/session/`, `packages/core/src/session/sql.ts`, `cli/cmd/run.ts`, `src/session/revert.ts`, `src/snapshot/`

- **저장 형식**: JSONL 파일 (append-only) + SQLite (코어 레벨). 세션 스키마는 `schema.ts`에 정의.
- **트리/fork**: `--fork` 플래그로 분기. `revert.ts`로 특정 지점으로 롤백. `snapshot/` 디렉토리에서 스냅샷 관리.
- **resume/share**: `--continue` (이어하기), `--session <id>` (특정 세션 선택). 공유 기능은 별도 확인 불가.
- **세션 이름/프로젝트 스코프**: `cwd` 기반 프로젝트 스코프. 세션 ID는 UUID.
- **서브에이전트**: 별도 서브에이전트 세션 격리 메커니즘은 디렉토리 구조에서 명확히 확인 불가.
- **이벤트 훅**: `run-state.ts`, `status.ts`에서 상태 전이 관리.
- **GC/압축**: `compaction.ts` (컨텍스트 압축), `overflow.ts` (오버플로 처리), `summary.ts` (요약).

## 2. pi-mono

**경로**: `packages/coding-agent/src/core/session-manager.ts`, `session-trees.ts`, `core/slash-commands.ts`

- **저장 형식**: JSONL (append-only 트리). 각 엔트리는 `id`/`parentId`로 트리 구조 형성. 리프 포인터(`leafId`)가 현재 위치 추적. 파일은 `~/.pi/agent/sessions/<cwd인코딩>/`에 저장.
- **트리/fork**: `branch(branchFromId)`로 리프 이동, `branchWithSummary()`로 분기 요약 생성, `createBranchedSession()`로 특정 경로를 새 세션 파일로 추출. `/tree`, `/fork`, `/clone` 명령.
- **resume/share**: `/resume`, `/compact`. 세션 이름은 `session_info` 엔트리로 저장 (`/name` 명령).
- **세션 이름/프로젝트 스코프**: `SessionHeader.cwd`로 프로젝트 스코프. `session_info.name`으로 사용자 정의 이름.
- **서브에이전트**: 트리 구조에서 자식 세션으로 표현 가능하나 명격적 격리 메커니즘은 확인 불가.
- **이벤트 훅**: `fromHook` 필드로 훅 생성 엔트리 구분. 커스텀 엔트리(`CustomEntry`)로 확장 상태 저장.
- **GC/압축**: `CompactionEntry` (요약 + firstKeptEntryId), `BranchSummaryEntry` (분기 요약), `ContextEditEntry` (컨텍스트 편집). 마이그레이션 v1→v2→v3.

## 3. oh-my-pi

**경로**: `packages/coding-agent/src/session/` (94개 파일)

- **저장 형식**: 다중 백엔드 추상화 — `session-storage.ts` (기본), `indexed-session-storage.ts`, `redis-session-storage.ts`, `sql-session-storage.ts`, `agent-storage.ts`.
- **트리/fork**: `session-worktree.ts`로 worktree 연동, `sub-sessions.ts`로 하위 세션, `session-manager.ts`에서 트리 관리. `/tree`, `/fork`, `/fresh`, `/branch` 명령.
- **resume/share**: `/resume`, `/export`, `/share`, `/handoff`, `/btw`. `session-handoff.ts`로 크로스세션 인계.
- **세션 이름/프로젝트 스코프**: `session-metadata.ts`, `session-title-slot.ts`에서 메타데이터 관리. `session-paths.ts`로 경로 계산.
- **서브에이전트**: `sub-sessions.ts`로 서브에이전트 세션 관리. `session-provider-boundary.ts`로 프로바이더 경계 격리.
- **이벤트 훅**: `agent-session-events.ts`, `agent-session-types.ts`에서 이벤트 정의. `session-listing.ts`에서 목록 조회.
- **GC/압축**: `session-maintenance.ts` (유지보수), `session-migrations.ts` (마이그레이션), `compact-modes.ts`, `compaction-methods.ts`. `foreign-session-store.ts`, `claude-session-store.ts`, `codex-session-store.ts`로 외래 세션 가져오기.

## 4. codex

**경로**: `codex-rs/rollout/` (recorder.rs, state_db.rs, compression.rs, session_index.rs, maintenance.rs)

- **저장 형식**: JSONL (`~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`) + SQLite (`state_db.rs`). 날짜 계층 디렉토리 구조.
- **트리/fork**: 압축 분기 (compression fork) — 압축 시 새 세션 파일 생성. `rollout_reference_index.rs`로 참조 관리.
- **resume/share**: `codex resume` 명령. `session_index.rs`로 세션 색인, `list.rs`로 목록 조회, `search.rs`로 검색.
- **세션 이름/프로젝트 스코프**: `metadata.rs`에서 메타데이터 관리. 프로젝트 스코프는 디렉토리 구조에서 추정.
- **서브에이전트**: 별도 서브에이전트 세션 메커니즘은 확인 불가.
- **이벤트 훅**: `recorder.rs`에서 이벤트 기록. `persistence_metrics.rs`로 지표 수집.
- **GC/압축**: `compression.rs` (zstd 압축), `maintenance.rs` (유지보수/정리), `policy.rs` (정책). `seekable_reader.rs`로 효율적 읽기.

## 5. claude-code

**경로**: `CHANGELOG.md`, `mods/types/claude-code.d.ts`

- **저장 형식**: JSONL (transcript 파일). 정확한 내부 구조는 타입 정의에서만 확인 가능.
- **트리/fork**: fork subagent (`context: fork` 스킬), 백그라운드 세션 포크. CHANGELOG에서 fork subagent의 plan mode/permission mode 상속 확인.
- **resume/share**: `--resume <id>`, `--continue`, `/resume` 피커 (TUI). `SessionResume`, `SessionSource` 타입. 백그라운드 세션에 프롬프트 전송 가능.
- **세션 이름/프로젝트 스코프**: 세션 ID 기반. 프로젝트 스코프는 추정.
- **서브에이전트**: fork subagent — 부모의 permission mode, plan mode 상속. MCP tool 목록 재구성 방지.
- **이벤트 훅**: SessionStart 훅 (출력 시 prompt cache miss 관련 이슈 다수). `/clear` 후 resume 시 첫 메시지 손실 이슈.
- **GC/압축**: 컨텍스트 압축 (compaction) 관련 이슈 다수. 대용량 세션 resume 시간 개선. 파일 캐시 복원.

## 6. openclaw

**경로**: `src/sessions/` (117개 파일), `<agentDir>/openclaw-agent.sqlite`

- **저장 형식**: SQLite (`openclaw-agent.sqlite`). 가장 풍부한 세션 관리 파일 집합.
- **트리/fork**: `session-worktree-lifecycle.ts`로 worktree 생명주기, `session-upstream-links.ts`로 업스트림 링크, `session-diff.ts`로 diff/revision 관리.
- **resume/share**: `session-suspension.ts`로 세션 일시중단, `session-lifecycle-admission.ts`로 생명주기 입장 제어, `session-work-admission-handoff.ts`로 인계.
- **세션 이름/프로젝트 스코프**: `src/sessions/classify-session-kind.ts` + `agent-harness-session-key.ts`로 세션 스코프/키 관리, `session-label.ts`로 라벨, `session-id.ts`로 ID 해석.
- **서브에이전트**: `subagent-terminal-state.ts`로 서브에이전트 터미널 상태, `nested-tool-activity.ts`로 중첩 도구 활동.
- **이벤트 훅**: `session-state-events.ts` (상태 이벤트 스트림), `session-lifecycle-events.ts` (생명주기 이벤트), `session-created.ts`.
- **GC/압축**: `session-state-events.prune.ts`로 이벤트 가지치기, `session-upstream-monitor.ts`로 업스트림 모니터링.

## 7. hermes-agent

**경로**: `hermes_state_sessions.py`, `hermes_cli/` session 명령, `hermes_state_rewind.py`(루트), `tests/hermes_state/test_rewind_surfaces_invariant.py`

- **저장 형식**: SQLite (`state.db`). `sessions` 테이블 + `messages` 테이블 + `system_prompts` 테이블 (content-addressed).
- **트리/fork**: `parent_session_id` 기반 계보. 재귀 CTE(`_LINEAGE_CTE_SQL`)로 compression lineage 추적. `_branched_from`, `_delegate_from` 마커로 분기/위임 구분. `/branch`, `/new` 명령.
- **resume/share**: `/resume`, `reopen_session()`으로 재개. `resolve_session_id()`로 접두사 일치. `find_session_by_origin()`으로 게이트웨이 세션 검색.
- **세션 이름/프로젝트 스코프**: `workspace_key()` (git repo root 우선, cwd 차선), `profile_name`으로 프로필 스코프. `display_name`, `session_key`.
- **서브에이전트**: `_delegate_from` 마커로 위임 서브에이전트 추적. `_collect_delegate_child_ids()`로 연쇄 삭제. 프로필 네임스페이스 상속 방지.
- **이벤트 훅**: `touch_session_activity()`로 활동 타임스탬프, `last_activity_description`/`last_activity_provenance`으로 활동 라벨.
- **GC/압축**: `end_session()` + `end_reason='compression'`으로 압축 분기. `_auto_archive_lineage()`로 자동 아카이브 스윕, `set_session_archived()`로 수동 아카이브. `promote_to_session_reset()`으로 리셋 경계.

## 8. oh-my-openagent

**경로**: `packages/omo-opencode/src/session/` — **경로 존재하지 않음**

- 저장소 추상화 도구 (`session_list`, `session_read`, `session_search`, `session_info`)로 추정되나, 실제 소스 경로를 확인할 수 없음.
- 세션 저장 형식, 트리/fork, resume 메커니즘 등을 확인할 수 없음.

---

## 관찰 및 시사점

1. **JSONL vs SQLite 양대산맥**: opencode/pi-mono/codex/claude-code는 JSONL(append-only) 기반, openclaw/hermes-agent는 SQLite 기반. oh-my-pi는 둘 다 지원. JSONL은 단순/이식성, SQLite는 질의/관계에 강점.

2. **트리 구조의 공통 진화**: pi-mono의 id/parentId 트리, hermes의 parent_session_id + 재귀 CTE, codex의 compression fork — 모두 선형 이력을 트리로 확장. opencode의 --fork, claude-code의 fork subagent도 같은 방향.

3. **압축/컨텍스트 관리**: 모든 하니스가 compaction을 구현하지만 방식이 다름. pi-mono는 CompactionEntry + firstKeptEntryId, codex는 zstd + 새 파일, hermes는 end_reason='compression' + 재귀 lineage.

4. **프로젝트/프로필 스코프**: hermes-agent가 가장 정교 (workspace_key + profile_name + 프로필 네임스페이스 상속 방지). pi-mono는 cwd 인코딩 디렉토리. openclaw는 `src/sessions/agent-harness-session-key.ts`.

5. **서브에이전트 격리**: hermes-agent의 delegate/branch 마커, claude-code의 fork subagent permission 상속, openclaw의 subagent-terminal-state가 가장 명확. 나머지는 트리 구조에 의존하거나 미확인.

6. **세션 인계(handoff)**: oh-my-pi의 `/handoff` + session-handoff.ts, openclaw의 session-work-admission-handoff.ts, hermes-agent의 find_session_by_origin() — 크로스세션 연속성이 새로운 관심사.

7. **oh-my-openagent 공백**: 경로가 존재하지 않아 비교에서 제외. 저장소 추상화 도구만 존재하는 것으로 보이며, 실제 세션 관리 구현은 별도 확인 필요.
