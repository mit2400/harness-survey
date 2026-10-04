# oh-my-openagent (OmO) — 오케스트레이션 심층

> `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` · 버전 `5.1.13` (2026-10-04)
> 상위 문서: [oh-my-openagent.md](./oh-my-openagent.md) · 교차 비교: [topics/orchestration.md](../topics/orchestration.md)

OmO는 공식 문서로 오케스트레이션을 **가장 잘 문서화한 하니스**다 (`docs/guide/orchestration.md` 325줄, `docs/guide/workflows.md`, `docs/reference/mass-ulw-protocol.md` 292줄). 그래서 이 문서의 가치는 요약이 아니라 **코드 수준 확증과 문서↔코드 불일치**에 있다.

## 먼저: OmO의 오케스트레이션은 "하나"가 아니다

이 점을 모르고 읽으면 모든 것이 뒤섞인다. 대상이 **세 개**로 갈린다.

| 표면 | 소속 패키지 | 사용자가 만나는 것 |
|---|---|---|
| **OpenCode 어댑터 오케스트레이션** | `packages/omo-opencode/` | `/goal`, `/ulw-execute`, boulder 읽기, keyword-detector, `todo-continuation-enforcer` |
| **네이티브 DAG 엔진** | `packages/senpi-task/src/dag/` (6,454줄 impl / 21파일) | `workflow` 툴(= DAG), `omo.dag.*` 와이어 프로토콜, Desktop Workflows 패널 |
| **Team Mode 도메인** | `packages/team-core/` (4,210줄 impl / 58파일) | `team_create` + 12개 `team_*` 툴, tmux, mailbox |

DAG 엔진(`senpi-task`)은 **`omo-senpi` 어댑터만 소비한다**. OpenCode 플러그인 경로에는 DAG가 없다. 즉 "mass ulw가 OpenCode에서 DAG를 돌린다"는 오해가 흔하다 — 공식 문서조차 혼용한다(§9).

## senpi-task DAG 엔진

### 파일 맵 (구현 파일만, 테스트 제외)

| 파일 | impl LoC | 역할 |
|---|---:|---|
| `dag/scheduler.ts` | 1,131 | dependency-frontier 승인 루프(`runFrontier`/`admitFrontier`), 레지던시 거부 큐, skip cascade, 이벤트 replay |
| `dag/store.ts` | 899 | `createDagFileStore`: WAL append/read, torn-tail 복구, 체크포인트/result/key 영속화, run/key/task-owner 락, 보존(pruning) |
| `dag/manager.ts` | 787 | `createDagManager`: `start`/`amend`/`replay`, `DagRunRecordV1` 투영, **fingerprint 키 run 재사용** |
| `dag/recovery.ts` | 602 | `createDagRecovery`: 소유자 대조, 저널 replay, 결과 재사용, 스케줄러 재진입, lost 처리 |
| `dag/graph.ts` | 355 | `compileDag`: 결정적 정렬, invalid-dep/cycle 검출, wave 그룹화, critical path, bottleneck |
| `dag/types.ts` | 399 | 도메인 어휘: 6 run status, 8 node state, route union, 17 이벤트 타입, settings 기본값 |
| `dag/events.ts` | 213 | 17개 경계 이벤트의 순수 빌더. seq/lane 메타데이터 없음 |
| `dag/node-retry.ts` | 252 | 실패/취소/스킵 노드 재시도(연쇄 스킵 해제 포함) |
| `dag/node-send.ts` | 220 | 실행 중 노드의 자식 steer, 종료된 노드의 revive |
| `dag/results.ts` | 187 | `persistDagNodeResult` / `readDagNodeResult` |
| `dag/skills.ts` | 147 | `createDagSkillMaterializer`: 생성 시점 effective-prompt 스냅샷 |
| `dag/handle.ts` | 282 | `createDagWaitSurface`: terminal `DagRunResult` 해석(**reject 하지 않고 resolve**) |
| `dag/fingerprint.ts` | 124 | `dagFingerprint`(canonicalize→sha256), `dagDefinitionFingerprint`, `nodeFingerprintInput` |
| `dag/node-control-context.ts` | 111 | 노드 단위 제어 컨텍스트 |
| `dag/recovery-lease-watch.ts` | 65 | paused run의 이전 holder pid 감시(signal-0, unref'd timer) |
| `dag/owner.ts` | 42 | `DagTaskOwner` = runId + nodeId + definition/node/execAttempt 핑거프린트 |
| `dag/node-activity.ts` | 23 | 마지막 child 트랜스크립트 활동 시각 투영 |
| `dag/execution-mode.ts` | 49 | 노드별 실행 모드 해석 |
| `dag/index.ts` | 143 | 공개 배럴 + `./dag` 서브패스 익스포트 |

> **문서↔코드 불일치 ①**: `senpi-task/src/dag/AGENTS.md:3`과 루트 `AGENTS.md`는 이 서브시스템을 **"35 files, ~13.9k LOC"** 라고 적는다. 실제는 **51파일 / 17,821줄(테스트 포함) / 구현 6,454줄(21파일)** 이다. `scheduler.ts`(1,956줄 테스트), `store.ts`(1,106줄 테스트), `recovery.test.ts`(1,187줄) 같은 회귀 매트릭스가 커진 탓이고, "가장 큰 서브시스템"이라는 서술 자체는 맞다.

### 컴파일 단계 — `compileDag` (`dag/graph.ts:153`)

**순수 함수**다. `at`(타임스탬프)는 호출자가 주입하고 기본값이 `EMPTY_AT = "1970-01-01T00:00:00.000Z"` 이며, 클럭·파일시스템·랜덤을 쓰지 않는다. 그래서 컴파일 결과가 바이트 단위로 재현 가능하다.

**8가지 컴파일 에러 (`DAG_COMPILE_ERROR_CODES`)** — 하나라도 있으면 `ok:false`이고 그래프도 런도 나오지 않는다(부분 실행 없음).

| 코드 | 조건 |
|---|---|
| `empty_graph` | 노드 0개 |
| `node_count_exceeded` | `> max_nodes_per_run`(기본 64) |
| `duplicate_node_id` | 같은 id 두 번 선언 |
| `prompt_bytes_exceeded` | 노드 prompt > `max_prompt_bytes`(기본 262144 = 256KiB) |
| `dependency_fanout_exceeded` | 한 노드의 `dependsOn` 길이 > `max_nodes_per_run` |
| `self_dependency` | 자기 자신 의존 |
| `unknown_dependency` | 선언되지 않은 id 참조 |
| `cycle` | 의존성 순환 |

**사이클 검출(`findCycles`, `graph.ts:107`)** 은 두 단계다. 먼저 Kahn 방식으로 "진입 차수 0이 될 수 없는 노드"를 반복적으로 벗겨내고(`graph.ts:109-119`), 남은 집합이 순환 영역이다. 그다음 **사전순으로 가장 작은 멤버에서 출발해, 그 영역으로 재진입하는 가장 작은 의존으로 항상 이동**한다. 따라서 같은 그래프는 항상 같은 순환 목록을 낸다. `graph.ts:141` 은 경로를 의존성 간선을 **역방향으로** 걸었다는 주석으로, 최종 보고 순서를 실행 순서로 뒤집고 자기 자신으로 닫는다: `[a, c, b, a]`.

**critical path**(`graph.ts:308-332`)는 가중치가 없으므로 "가장 긴 의존성 사슬"이다. `resolveLongest` 는 자식 방향으로 재귀하고, `compareSequences`(`graph.ts:72`)가 **길이 우선, 그다음 사전순**으로 동률을 깨므로 실행 가능한 상한(earliest finish) 내에서 결정적이다.

**bottlenecks**(`graph.ts:334-352`)는 각 노드가 전이적으로 막는 자손 수(`blockedCount`)를 내림차순 정렬한다. 리포트용 메타데이터일 뿐 스케줄링에는 쓰이지 않는다.

**waves**는 깊이(= `max(deps depth) + 1`) 그룹이고, AGENTS.md가 못 박는 강조점이 있다:

> Compiled waves **NEVER** gate execution. `dag.wave.started` 는 한 번의 승인 패스가 스케줄한 노드를 묶는 **정보용 이벤트**이며, 한 wave index가 여러 started 이벤트에 나타날 수 있다. `dag.wave.completed` 는 그 wave의 정원이 전부 terminal이 될 때 index당 한 번만 발생한다(skipped/failed 포함).

즉 **wave는 렌더링·그룹핑 힌트**고, 실제 스케줄링은 wave barrier가 아니라 **frontier**다.

### 의존성 frontier 스케줄링 — `runFrontier` (`dag/scheduler.ts:529`)

노드 하나가 시작되는 조건은 정확히 둘이다: **자신의 `dependsOn` 노드 전부가 `completed`** + **resident 슬롯 하나가 비어 있음**.

루프 한 회전의 순서(`scheduler.ts:535-558`)는:

1. 취소/중단 확인 → `cancelledSnapshot`
2. `attachedTasks.size === 0` 이면 `applyDependentSkipCascade` — **frontier 정지점에서만** 실행
3. `emitCompletedWaves` — 다음 승인 패스보다 **먼저** 완료 wave를 보고
4. `admitFrontier` — 아래
5. 전 노드 terminal이면 종료
6. `attachedTasks.size === 0` 이고 `hasCascadableDependent`면 **루프 재진입**(#8396)
7. 아니면 `settleOne` — 자식 하나 완료 대기

**skip cascade를 정지 시점으로 미루는 이유**가 이 엔진에서 가장 미묘한 설계다. 실패한 노드는 형제 노드가 날던 중에도 `send` → `revive`로 되살릴 수 있다. 그래서 즉시 cascade 하면 "되살릴 수 있었는데 skipped 로 굳어버리는" 의존자가 남는다(`dag_2d12c2f7` 회귀). 실패를 **정지 시점의 사실**로 취급하는 게 맞다.

**residency 거부의 두 갈래**(`scheduler.ts:660-687`)도 같은 정밀도로 나뉜다:

| 거부 `cause` | 처리 |
|---|---|
| `"residents"` + 빈 목록 | 기다려도 소용없음 → 노드를 `residency_denied` 로 **terminal 실패** |
| `"residents"` + 목록 있음 | `scheduled` 로 park, `residency_queued` 사유로 **1회만** 저널 |
| `"lease"` | 즉시 재탐색(lease 획득 자체가 bounded wait) |

park 된 노드의 대기 대상은 `Promise.race([own settlements, taskManager.residencyChanged(parentSessionId), foreign journal commit, cancellation])` 이다. **자기 `attachedTasks` 만 기다리면 안 된다** — 나중에 도착한 런은 자식이 하나도 없어서 영원히 park 된다(#8396, 동일 이슈의 두 번째 증상).

### WAL 저널과 체크포인트 — `dag/journal.ts`

`<stateDir>/dag/{runs,events,results,keys,locks}` 레이아웃, `dagKeyHash = sha256(parentSessionId + "\0" + runKey)`, temp+rename 원자 쓰기, **fsync 기본 on**(`dag/store.ts:153`).

**append 경로**(`journal.ts:95-136`)는 run 락 안에서 전부 일어난다:

1. `recoverCheckpoint` — 체크포인트를 읽고 `checkpointSeq` 이후 이벤트를 **페이지 단위(1000개) 재생**
2. `skipDuplicate?.(recovered, payload)` 가 true면 **아무것도 저널하지 않고 구독자도 통지하지 않고 return** — 멱등 중복 제거(#7412, `dag_923ad20e` completed→completed WAL 클래스)
3. `seq = max(checkpointSeq, tailSeq) + 1` — 체크포인트가 WAL 꼬리보다 뒤처져 있어도 단조 증가 보장
4. `dag.node.transitioned` + `to === "completed"` 면 **WAL append 전에 먼저** reducer를 돌려 result artifact를 스테이징 — 프로세스가 체크포인트 교체 전에 죽어도 replay로 메타데이터를 재구성할 수 있다(`journal.ts:114-118`)
5. `store.appendEvent(event)` → `applyEvent` → `writeCheckpoint`
6. 락 밖에서 구독자 큐에 enqueue, `publishCommit` 으로 프로세스 전역 커밋 알림

**이중 알림 경로**가 있다. `journal.subscribe` 의 `Subscriber` 는 링 버퍼(`subscriber_ring` 기본 1000)를 갖고 있고, `subscribeDagJournal` 의 `CommitSubscriber` 는 링을 갖지 않는 store 级 WeakMap 알림이다. 링이 넘치면 `enqueue`(`journal.ts:207`)가 `OverflowState{ droppedCount, recoverAfterSeq }` 를 쌓고 drain 루프(`journal.ts:233`)가 **WAL 경유로** `dag.stream.overflow` 이벤트 자체를 append한다. 즉 오버플로 통지도 seq를 소비하며, `recoverAfterSeq` 는 "손실 직전 마지막 전달 seq" — **첫 유실 seq가 아니라** exclusive 복구 커서다.

### 복구 시맨틱 — `dag/recovery.ts`

`createDagRecovery` 의 규칙이 이 서브시스템의 하드코딩 난관이다.

| 규칙 | 이유 |
|---|---|
| **살아 있는 자식을 await 하지 않는다** | 재진입된 자식은 `preAttachedTasks` 로 스케줄러에 넘기고 `dag.run.resumed` 를 즉시 방출한다. await 하면 run 이 가장 느린 자식 동안 `paused` 에 묶이고 amend·retry 전부 `run_still_active` 로 거부된다 |
| **자기 pid 를 lease 생존 판정에 쓰지 않는다** | `signal-0` 은 `process.pid` 에서 항상 성공한다. 같은 멀티세션 호스트에서 세션을 재개하면 자기 호스트 종료를 기다린다(#8006). `holderPid === hostPid` 면 주입된 `isRunHeldInProcess(runId)` 만 본다 |
| **`live_lease` 스킵을 자기 세션 런의 최종 판정으로 쓰지 않는다** | 선행 호스트가 셧다운으로 run 을 pause 시키고도 draining 중이라, 한 번만 claim 하면 run 이 영구 pause 된다. `recovery-lease-watch.ts` 가 holder pid 가 죽으면 1회 발화해 복구를 재실행한다 |
| **표시 attempt 로 복구 키를 만들지 않는다** | persisted `execAttempt` 를 읽는다. 재attach 는 표시 attempt 를 올린다 |
| **이전 스케줄러 인스턴스를 resume 하지 않는다** | 취소 deferred와 승인 latch 는 single-shot. 재진입은 `preAttachedTasks` 를 **버린다** — 그 id들은 이전 인스턴스 시점의 자식이고 이미 settle 된 태스크를 재attach 하면 결과가 두 번 접힌다 |
| **자동 readmission 은 stamp 없는 scheduled+lost 만** | `TaskRecord.started_at` 는 러너 호출 **전에** durable. 스탬프된 lost 는 `task_lost` 로 접는다. 스탬프 직후 호출 전 크래시는 보수적으로 `task_lost` |
| **재시도 상한 3회** | `dag.node.retried` + `execAttempt + 1`, 3에서 포화 |

`recovery.ts` 의 처리 파이프라인은 `pauseRunsForShutdown` → `claimOrphanedRun` / `claimPausedRun` → `resumeClaimedRun` → `reconcileNodes` → `foldTaskOutcome` 이다.

**프런티어의 실패 정책은 `continue-independent`** 다. 한 노드가 죽어도 무관 노드는 계속 돈다. 실행 엔진이 워크플로 한 바퀴를 잃지 않기 위한 선택이다. 반대로 **wave barrier는 이 정책과 충돌**하므로 2026-08-25에 `strict-barrier` → `dependency-frontier` 로 바뀌었고, 그 정책 문자열은 **definition fingerprint 안에서 고정**된다 — 스케줄링 정책 변경이 자동 reuse를 깨뜨리지 않는다.

## 와이어 프로토콜 — `docs/reference/mass-ulw-protocol.md`

이 문서는 292줄짜리 **구현 유래** 계약서다. "A viewer can be implemented from this document alone"을 목표로 썼고, 실제로 테스트로 고정된다(`packages/omo-senpi/src/components/task/mass-ulw-protocol-doc.test.ts`). 4 push 채널 + 4 request 메서드, schemaVersion 1.

### Push 채널 4개 (단 1개만 seq 있음)

| 채널 | seq | 영속 | 목적 | 코드 |
|---|---|---|---|---|
| `omo.dag.event` | 있음 | WAL | 저널된 run 원장. **유일한 seq 원장** | `dag-rpc-bridge.ts:31` |
| `omo.dag.heartbeat` | 없음 | 없음 | 비terminal run 생존 비콘, 기본 15초 | `:32`, `DAG_DEFAULT_HEARTBEAT_MS = 15000` |
| `omo.dag.activity` | 없음 | 없음 | 노드별 라이브 텔레메트리, 150ms 윈도우 latest-wins | `:33`, `DAG_ACTIVITY_COALESCE_MS = 150` |
| `omo.dag.updated` | 없음 | 없음 | run 목록 통째 스냅샷, 50ms 디바운스 + 핑거프린트 중복 제거 | `:36`, `DAG_SNAPSHOT_DEBOUNCE_MS = 50` |

**전송 게이트가 실제로 걸린다.** 클래식 RPC 클라이언트는 `SENPI_RPC_CLIENT_CAPABILITIES=extension_events` 를 선언해야 extension 이벤트를 받는다. **미선언 시 4개 push 채널이 조용히 도달 불가능**하고 request 메서드만 동작한다. app-server 전송은 thread-scoped 라 플래그가 필요 없다. UI를 만들 때 이 한 줄 때문에 "이벤트가 안 온다"가 되돌아온다.

`omo.dag.event` 페이로드는 envelope와 payload의 **flat 교집합**이다. `type`/`seq`/payload 필드가 **형제 관계**로 한 오브젝트에 놓인다(중첩 아님). envelope: `schemaVersion`/`runId`/`seq`/`at`/`lane`.

`lane` 은 `"boundary" | "activity"` 둘인데, `dagEventLane()`(`dag/events.ts:17`)은 **17개 저널 타입 전부를 `"boundary"` 로 매핑**하고 미지 타입에 throw 한다. `activity` 는 lane **분류용 이름**일 뿐 실제 `DagActivityEvent` 는 전부 unsequenced로 `omo.dag.activity` 를 탄다. **seq 원장에 `lane:"activity"` 이 나타나는 일은 없다.**

17개 저널 타입은 `dag.run.{created,started,paused,resumed,completed,failed,cancelled}` + `dag.wave.{started,completed}` + `dag.node.{transitioned,task-attached,reused,retried,steered}` + `dag.definition.amended` + `dag.diagnostic.added` + `dag.stream.overflow`.

**배출 규칙**: 저널 이벤트는 **WAL append와 체크포인트 교체가 둘 다 성공한 뒤에만** 채널로 나간다. durability 이전 방출 경로가 없다. 받은 이벤트는 이미 디스크에 있고 `omo.dag.history`로 replay된다.

### Request 메서드 4개

모두 동일 envelope `{ ok: true, value }` / `{ ok: false, error: { code, message } }`. 경계를 넘어가는 throw는 없다 — 알 수 없는 실패는 `history_unavailable` 로 강등된다. 에러 코드: `invalid_arguments`, `run_not_found`, `run_not_owned`, `history_unavailable`.

| 메서드 | 입력 | 출력 |
|---|---|---|
| `omo.dag.list` | `{ statuses?, limit? }` (기본 100, 상한 256) | run 요약 + 전체 node 카운터. **상태 필터는 엔진 최대 윈도우 뒤에 적용**되므로 필터가 윈도우가 잡은 런을 잃지 않는다 |
| `omo.dag.snapshot` | `{ runId }` | 전체 camelCase `DagRunSnapshot`. `lastSeq`, 그래프(`nodes`/`edges`/`waves`/`criticalPath`/`bottlenecks`), `diagnostics`, `counts`, `amendHistory`. 노드 `prompt` 는 제출된 prompt 다(`effectivePrompt` 아님) |
| `omo.dag.history` | `{ runId, sinceSeq?, limit?, lane?, types?, throughSeq? }` | `{ events, nextSinceSeq, headSeq, hasMore }`. `sinceSeq` **exclusive**, 기본 256 상한 1000 |
| `omo.dag.subscribe` | `history`와 동일 형태 | **무상태 catch-up 핸드셰이크.** 서버 구독자를 등록하지 않는다. 스냅샷을 먼저 읽으므로 `highWaterSeq` 는 첫 페이지 이전의 원장 상태이고 페이지가 그 마크로 유계된다 |

**무결 catch-up 6단계**가 문서에 그대로 있다: (1) `omo.dag.event` 리스너를 **가장 먼저** 등록하고 전부 버퍼 → (2) `omo.dag.subscribe` 호출, `highWaterSeq` 기록 → (3) 핸드셰이크 page 적용 → (4) `hasMore` 동안 `sinceSeq = nextSinceSeq`, `throughSeq: highWaterSeq` 로 페이징(마크가 윈도우를 얼려 라이브 이벤트와 경합하지 않는다) → (5) 라이브 버퍼를 `seq <= highWaterSeq` 및 이미 적용한 `(runId, seq)` 를 버리며 seq 순으로 적용 → (6) steady state 는 마지막 적용 seq 추적, 비연속 seq 감지 시 `sinceSeq = lastApplied, throughSeq = received seq` 로 그 구간 재조회.

**오버플로 복구**: 오버플로 이벤트 수신 시 정상 적용을 멈추고 `sinceSeq = recoverAfterSeq, throughSeq = overflow.seq` 로 페이징한다. `recoverAfterSeq` 에서 1을 빼지 말고, `droppedCount` 나 오버플로 자신의 seq에서 커서를 추정하지 않는다.

### UI 노출 — Desktop / 웹

**코드가 확정해 주는 것**:
- `omo-senpi` 슬래시 커맨드 `/dag`(detal 뷰), `/tasks`, `/task-kill` (`components/task/dag-commands.ts`, `commands.ts`)
- TUI 상태 표 렌더러 `dag-status-ui.ts` + `dag-status-row-format.ts` + `quietNodesNotice`(조용해진 노드 경고, `DAG_NODE_QUIET_AFTER_MS = 600_000` = 10분은 **보고 임계값이지 실행 임계값이 아니다**)
- `omo.dag.updated` 는 `omo.task.updated` 와 **똑같은 통째 교체 패턴**으로 만들어졌다: `parent_session_id` 의 run 목록을 `runs` 로 교체, per-event 상태·seq 추적·머지 로직 없음. `truncated_runs` 는 256 상한에서 잘렸을 때만 존재
- **리듀서는 아직 없다.** `mass-ulw-protocol.md:273-292` 의 "omo-desktop-app integration" 절이 명시적으로 *"Deferred follow-up. The wiring described here is NOT shipped"* 라고 쓴다. 앱은 `omo.task.updated` 만 소비하고 `omo.dag.updated` 는 아직 등록하지 않는다

**> 문서↔코드 불일치 ②**: `docs/guide/workflows.md:36-50` 는 OmO Desktop에서 "**Workflows panel** lists every run in the thread", 태스크 상태(Working/Done/Failed/Stopped), 의존 그래프, live feed, 태스크 클릭으로 해당 에이전트 열기를 사용자에게 약속한다. 반면 reference 문서는 이 연동을 **미출시**로 명시한다. 정답은 코드 쪽이다 — reference 문서는 구현을 직접 읽고 쓴 정본이다. workflows.md는 데스크톱 클라이언트가 `workflow` 툴 위에 이미 가진 UI 를 **기술이 아니라 결과로 서술**한 것이다. 데스크톱 UI 는 버전/채널별로 확인해야 한다.

**웰 노드 스냅샷의 snake_case** 는 의도적이다. 앱 데코더가 그 규약을 기대하므로. 단 **애드티브** 규약이다 — 새 필드는 absent(절대 `null` 아님), `schemaVersion` 은 1 로 고정, 알 수 없는 이벤트 타입과 필드를 무시하는 소비자는 계속 동작한다.

## boulder-state — compaction·재시작을 통과하는 작업 상태

### 해결하는 문제

평범한 todo 리스트는 **컨텍스트 안에 산다**. compaction이 일어나면 사라지고, 세션이 죽으면 잃는다. boulder-state는 그 상태를 `<worktree-root>/.omo/boulder.json`(`schema_version: 2`)에 **파일로** 둔다. 세션·워크트리·서브에이전트에 걸친 "지금 무엇을 하고 있었나"를 보존하는 것이 목적이다.

`packages/boulder-state/` — **1,656줄 impl**, **의존성 0개**. 순수 함수형 JSON 상태 머신.

### 핵심: `works` 맵 + 루트 미러

`BoulderState`(`src/types.ts`)는 두 층 구조다:

```
active_work_id ──┐
                 ├─→ works: Record<work_id, BoulderWorkState>   ← 진짜 상태
루트 필드(───────┘   active_plan, status, session_ids, …)         ← 활성 워크의 미러
```

`selectMirrorWork()` 가 활성 워크를 고르고(명시적 id, 없으면 `updated_at` 최신), `projectWorkToMirror()` 가 루트로 복사한다. `writeBoulderState()` 는 직렬화 전에 루트 → work 엔트리로 동기화하므로 **미러가 오래된 상태로 남지 않는다**. 레거시 단일 워크 상태(`works` 없음)는 `getBoulderWorks()` 가 자동 업그레이드한다.

이 구조는 **"active가 바뀌면 root 필드가 통째로 바뀐다"** 는 뜻이다. opencode·senpi·codex 세 어댑터가 같은 파일을 읽어도 각자 `active_work_id` 해석이 일관되고, 다른 세션의 워크는 무시된다.

### 상태 머신

| 열거형 | 값 | 정의 위치 |
|---|---|---|
| `BoulderWorkStatus` | `active` \| `paused` \| `completed` \| `abandoned` | `src/types.ts:22` |
| `BoulderTaskStatus` | `running` \| `completed` \| `cancelled` | `src/types.ts:23` |
| `BoulderSessionOrigin` | `direct` \| `appended` | `src/types.ts:19` |
| `TaskSessionState` | `task_key`/`task_label`/`task_title`/`session_id` + `started_at`/`ended_at`/`elapsed_ms`/`status` | `src/types.ts:58` |

> **문서↔코드 불일치 ③(사소하나 확인 필요)**: `topics/orchestration.md:31` 은 "boulder 상태기 `packages/boulder-state/src/types.ts` (`active|paused|completed|abandoned` + `stale_since`)" 라고 쓰는데, 이 커밋에서 정답은 **`active|completed|paused|abandoned`** 다. 값 집합은 같지만 **선언 순서가 다르다**(`src/types.ts:22`). 스키마나 코드 동작엔 영향 없지만 유니온 순서에 의존한 생성을 하면 어긋난다.

`completeBoulder` 가 **유일한 완료 전이**다. 그래서 abnormal 종료한 워크는 `active` 로 영구 잔류한다(#8413). 이걸 메꾸는 것이 `reconcileStaleWorks`(`src/storage/stale-work.ts`):

- 기준 = 워크의 **마지막 활동** = 자기 세션 트랜스크립트 mtime들, `updated_at`, `started_at` 중 **최신**
- 임계 = `OMO_BOULDER_STALE_WORK_THRESHOLD_MS`(기본 **6시간**) 이상 오래됐으면 `active` → `paused` demote + `stale_since` 스탬프
- **활동 증거가 전혀 없는 워크는 무조건 stale**
- stale 없으면 **쓰기 없음**. `completed`/`abandoned`/status 없는 레코드는 절대 안 건드린다. 모든 실패는 삼킨다
- 트랜스크립트는 **세션 디렉터리 이름이 워크의 cwd/worktree와 알파벳 모양으로 일치하는 경우에만** 읽는다 — agent home에는 수천 개가 쌓이므로
- `selectActiveWork`/`appendSessionIdForWork` 가 `stale_since` 있는 워크를 `active` 로 되돌리며 스탬프를 버린다. 스탬프 없이 pause 된 워크는 status 유지

**plain todo 리스트와의 차이**가 여기서 명확해진다:

| | todo 리스트 | boulder-state |
|---|---|---|
| 저장 위치 | 대화 컨텍스트 | `.omo/boulder.json` (파일) |
| compaction | 사라짐 | 살아남음 |
| 다중 워크 | 없음 | `works` 맵 + `active_work_id` |
| 연속 실행 연료 | 있음 (미완료 = 계속 해라) | 있음 + **demote 규칙으로 무한루프 방지** |
| 세션 추적 | 없음 | `session_ids` + `task_sessions` + 경로 prefix |
| 완료 판정 | 모델이 체크 | `completeBoulder` 단일 전이 + `reconcileStaleWorks` |

`session_ids` 는 **정규화**된다 — `normalizeSessionId` 가 `opencode:` / `codex:` / `senpi:` prefix를 붙인다(senpi 호출자는 `senpi:<id>` 를 직접 전달해야 한다). `readBoulderState` 는 빈 `{}` 를 invalid로 거부한다(null 반환). 프로토타입 오염 방어로 `RESERVED_KEYS = {__proto__, prototype, constructor}` 키는 task upsert에서 거부된다. `writeBoulderState` 는 첫 mkdir 때 `.omo/.gitignore`(`*`, `!/rules/`)를 만든다.

### plan 체크리스트 파싱 — 두 모드

`src/plan-checklist.ts` 의 `parsePlanChecklist` 은 **구조화 모드**(`## TODOs` / `## Final Verification Wave` 헤딩 존재)와 **단순 모드**(최상위 `-`/`*` 체크박스 개수)로 갈린다. 구조화 모드는 그 섹션 안의 라벨된 행만 센다: TODOs 아래 `N` / `T<n>[.<n>[a]]`, final wave 아래 `F<n>` / `H<n>`, 뒤에 `.`/공백/`-`/em-dash. **구조화 파싱이 0개인데 다른 헤딩 아래에 최상위 체크박스가 있으면 단순 모드로 폴백**한다.

`[x]` 완료, `[ ]` 남음, `[~]`(blocked)는 **총계에만 더해지고** 미완료에는 안 들어간다. 즉 blocked 항목이 연속 실행을 붙잡고 있지 않는다 — 이게 의도다. 코드 펜스(`markdown-fence.ts`)와 들여쓴 체크박스는 건너뛴다.

`isStructuredTaskRow` 이 행 문법을 OpenCode의 plan-format validator에, Codex Stop 훅은 `omo-codex/plugin/components/ulw-execute-continuation/src/plan-checklist.ts` 에 **자기 사본**을 갖는다. (일부 의도적 중복이다.)

**소비자**: `omo-opencode`의 `features/boulder-state/*` 재export + 훅 `atlas`/`ulw-execute`/`todo-continuation-enforcer` + CLI `boulder`; `omo-codex`의 `plugin/components/ulw-execute-continuation/boulder-reader.ts`; `omo-senpi`의 `src/components/ulw-execute-continuation/boulder-eligibility.ts`. 두 ulw-execute 읽기 경로 모두 **먼저 `reconcileStaleWorks` 를 호출**한다. 이 패키지는 home 경로를 스스로 해석하지 않는다.

## 루프 패밀리 — 5개 기제가 어떻게 맞물리는가

| 기제 | 레이어 | 정지 조건 | 지속 방식 |
|---|---|---|---|
| `mass ulw` 키워드 | 키워드 → `workflow` 툴 | DAG run terminal | DAG 스케줄러 |
| `/ulw-execute` | boulder + continuation 컴포넌트 | plan 최상위 체크박스 전부 `[x]` | idle 재주입 (최대 8회 연속) |
| `ulw loop` 스킬 | 코드 리뷰 게이트 | 증거 바인딩 성공 기준 | 레지스트리 + ledger |
| `/goal` | `goal` Session 훅 | `update_goal status:complete` (완료 감사 강제) | `session.idle` 마다 continuation 재주입 |
| **boulder** | 위 셋의 **공통 상태 파일** | — | 파일 |

관계는 계층적이다. **boulder는 루프가 아니라 상태기**이고, `/ulw-execute`·`goal`·`todo-continuation-enforcer` 가 그 파일을 읽고 재개 여부를 결정한다. `mass ulw` 는 boulder와 무관하게 DAG 내부에서 돈다. `goal` 은 per-session이고(`~/.omo/goal/{encodeURIComponent(sessionID)}.json`), boulder는 per-worktree다. **같은 "루프"라는 단어가 세 개 서로 다른 수명을 뜻한다.**

`goal` 훅 디스패치는 OpenCode의 위험한 API를 정면으로 통과한다: `buildContinuationPrompt(goal)` → `dispatchInternalPrompt`(async 게이트, `settleMs: 150`, `queueBehavior: "defer"`, source `goal:idle-continuation`). 목적지는 `<untrusted_objective>` 로 XML 이스케이프된다. `inFlightContinuations` Set이 세션당 재진입을 막는다. 목적지를 문장으로 감싸지만 **완료 감사 없이 complete로 못 간다**는 계약을 프롬프트에 못박는다. TUI 미러는 `.omo/ulw-loop/{sessionID}/goals.json`.

### ralph-loop → goal 마이그레이션

`packages/omo-opencode/src/hooks/ralph-loop/` 에 **~52개 파일이 실제로 남아 있다**(31 impl + 21 test). 그리고 `hooks/ralph-loop/AGENTS.md:7` 첫 줄이 이렇게 시작한다:

> **DEPRECATED.** Superseded by `goal/` (PR #6184 "goal-replaces-ralph"). `ralphLoop` was removed from `HookNameSchema` and `create-session-hooks.ts`; the `/ralph-loop`, `/ulw-loop`, `/cancel-ralph` builtin commands and templates were removed; `ralph_loop` config is a deprecated passthrough for migration. The directory, `createRalphLoopHook` factory, and barrel export remain, but no composer imports them.

마이그레이션 표:

| 항목 | 처리 |
|---|---|
| `ralphLoop` 훅 이름 | `HookNameSchema`와 `create-session-hooks.ts`에서 **제거** |
| `/ralph-loop`, `/ulw-loop`, `/cancel-ralph` builtin 커맨드·템플릿 | **제거** |
| `hooks/ralph-loop/` 디렉터리 + `createRalphLoopHook` 팩토리 + barrel export | **남김**(마이그레이션용), 컴포저가 import 안 함 |
| `ralph_loop` config | `z.record(z.string(), z.unknown())` 로 느슨하게 파싱 → `validate.ts:166-173` `migrateRalphLoopConfig()` 가 `goal.{enabled, auto_start, default_max_iterations}` 로 매핑 + deprecation 로그 |
| 명시적 `goal` 설정 | 마이그레이션 값보다 **승리** |
| `default_mode.ralph_loop` | `default_mode.goal` 로 변경 (첫 main 세션 메시지에서 자동 goal 생성) |

**행동 등가성 보존 방식**이 흥미롭다. ralph의 `maxIterations`(기본 100)가 `goal.default_max_iterations`(기본 100)로 그대로 옮겨간다. 판정기인 `<promise>DONE</promise>` 스캔이 "완료 감사" 계약으로 바뀌었다. 즉 **트리거는session.idle 재주입으로 동일하고, 완료 판정만 느슨한 문자열 매치에서 계약 검사로 강화**됐다.

**주의**: `/ulw-loop` builtin 커맨드가 없어졌다는 것은 `packages/omo-senpi/skills/ulw-loop/SKILL.md` 같은 **스킬은 여전히 존재한다**는 뜻이다. 어댑터별 표면이 다르다.

`/stop-continuation`(`plugin/stop-continuation.ts:25-29`)이 이 기제들을 한 번에 해체한다.

## 태스크 위임 — DAG가 자식 태스크를 다루는 방식

DAG 노드는 결국 senpi-task의 자식 태스크다. DAG는 그 위에 얹힌 **스케줄링 계층**이고, 실행은 `startOwned`가 담당한다.

### DAG가 소유하지 않는 것

senpi-task `AGENTS.md` 의 실행 모드 규칙이 그대로 적용된다.

| 모드 | 자식 실행 | 사용 |
|---|---|---|
| **in-process** (기본) | 부모와 **같은 tool closure** 를 통해 같은 Senpi 런타임에서 실행. `filterSharedParentTools` + `mergeChildCustomTools` 로 부모의 라이브 커스텀 툴을 보되 `task_*`/`team_*` 계열은 제거 | 가장 싸고 프로세스 없음 |
| **process** | 자식 Senpi 프로세스 스폰. steer/abort/prompt가 **JSON-RPC 경계**를 넘고 트랜스크립트는 `<stateDir>/children/<taskId>/sessions/<taskId>/` | 팀 멤버는 **항상** 이 모드 |
| **process via daemon** (`task.process_runner: host`) | 프로세스를 띄우는 대신 부모 세션의 엔진 호스트 안에서 **세션으로** 연다 (세션 트리당 호스트 1개, `<shardRoot>/p-<shardKey("p", rootSessionId)>.sock`) | macOS/Linux 기본 |

`dag/execution-mode.ts:25` 의 `resolveDagNodeExecutionMode` 는 **DAG 전용 설정 키가 없다**고 명시한다:

> There is **NO `dag.default_execution_mode` knob**; the strict task schema rejects unknown keys inside `task.dag`.

해상 순서는 기존 체인과 **문자 그대로 동일**: `spec.execution_mode ?? agentDef.executionMode ?? omo.json task.default_execution_mode ?? "in-process"`. curated read-only 에이전트는 `CURATED_READONLY_AGENT_NAMES` 검사로 강제 in-process. `auto` 를 부모 세션이 아직 못 정했으면 **undefined 를 반환한다** — 추측으로 라우팅하지 않는다.

**`auto` 의 판정**: 부모 세션이 자신의 task host에게 **한 번만** 물어본다 — Windows가 아니고, `process_runner: "host"` 이고, host가 `session_context` + `generation_handoff` 를 광고하면 `process`, 아니면 `in-process`. **`OMO_RPC_SOCKET*` 는 자식을 절대 라우팅하지 않는다**(오퍼레이터 명령·thread 툴 전용).

### steering / waiting / stopping

`dag-tool.ts:39` 의 description이 툴 계약을 그대로 선언한다:

- **`wait`는 기본 detach** — 라이브 run에 대해 세션이 노드 완료마다, 그리고 run settle 시점에 웨이크업된다. `detach=false`가 최종 결과를 반환하는 블로킹 대기
- **`send`**는 실행 중 노드의 자식을 steer하고, 종료된 노드는 **revive**한다 (`dag.node.steered` 의 `delivery: "steer" | "revive"`)
- **`retry`**는 실패 노드를 **제자리에서** 다시 돌린다. 연쇄 스킵된 의존자도 함께 해제한다
- **`amend`**는 그래프를 편집하고 **변경된 것만** 다시 돌린다
- description이 못 박는다: *"When a run settles badly, do NOT start a new one"*

> **문서↔코드 불일치 ④**: description 첫 문장이 `mass ulw` 키워드가 **eval 셀 안에서 프로그래밍하게** 그래프를 짜라고 지시한다(*"A graph is a program; write it as one instead of hand-authoring a call"*). 반면 `docs/guide/orchestration.md` 의 `mass-ulw` 절은 "defines a run for the `workflow` tool from an eval cell: nodes with `id`, `prompt`, `category`, and `dependsOn`" 라고 적어 같은 사실을 말하지만 훨씬 수동적이다. 그리고 `docs/guide/workflows.md` 는 코드 셀을 아예 언급하지 않는다. **사용자 가이드가 엔진의 가장 강한 설계(그래프를 프로그램으로 쓰기)를 숨기고 있다.**

노드 오류 코드 11개(`DAG_NODE_ERROR_CODES`): `plan_unresolved`, `depth_denied`, `start_failed`, `residency_denied`, `task_error`, `task_interrupted`, `task_lost`, `task_cancelled`, `resume_task_missing`, `resume_task_orphaned`, `journal_corrupt`.

### idle parking과 메시지 부활

`task.resident_idle_timeout_ms` — 기본 **900000ms (15분)**, 양의 safe integer 밀리초만 허용. `0`·분수·문자열·비활성 센티널은 **모두 거부**된다. 스윕은 `updated_at + timeout_ms` **이후에만** park 한다.

- in-process 자식 → `persisted_only`, process 자식(팀 멤버 포함) → `rpc_detached`. 라이브 핸들은 **해제**될 뿐 파괴되지 않는다
- `task_send`가 `updated_at`을 갱신한다. 실행 중 자식과 pending steering이 있는 자식은 보호된다
- parked 자식에 **직접** `task_send`하면 기록된 트랜스크립트와 launch contract를 복원하고 **새 run epoch 하나를 획득**하며, **전달 acknowledgment 이후에야** 부활을 보고한다.准入 거부·불확실한 전달은 명시적 에러이며 **불확실한 메시지는 자동 재전송되지 않는다**
- killed·cancelled·lost·one-shot 자식은 non-continuable로 남는다
- **팀 이름 전송은 durable mailbox 쓰기**다. parked 프로세스는 inbox poller가 없으므로 팀 이름으로는 부활하지 않는다 — task id로 직접 부활시키거나 소유 세션을 재개시켜야 한다

parking은 `task.ttl_ms`(기본 86400000ms = 24h)와 독립이다. 구 출력은 레코드 만료 전까지 `task_output` 으로 읽을 수 있다.

## Team Mode — lead + 멤버

`packages/team-core/` — **4,210줄 impl / 58파일**(AGENTS.md 는 "82 TypeScript files" 라고 적는데 테스트까지 센 수로 보인다). `omo-opencode` 어댑터와 `senpi-task` 양쪽이 소비하는 **harness-neutral 도메인 프리미티브**다.

### 11필드 설정 스키마 대조

`docs/guide/team-mode.md:34` "Config schema (11 fields)" 를 `packages/team-core/src/config.ts` 의 `TeamModeConfigSchema` 와 1:1 대조했다.

| 필드 | `docs/guide/team-mode.md` | `team-core/src/config.ts` | 일치 |
|---|---|---|:-:|
| `enabled` | boolean, 기본 `false` | `z.boolean().default(false)` | ✅ |
| `tmux_visualization` | boolean, 기본 `false` | `z.boolean().default(false)` | ✅ |
| `max_parallel_members` | int `1..8`, 기본 `4` | `z.number().int().min(1).max(8).default(4)` | ✅ |
| `max_members` | int `1..8`, 기본 `8` | `z.number().int().min(1).max(8).default(8)` | ✅ |
| `max_messages_per_run` | int `>=1`, 기본 `10000` | `z.number().int().min(1).default(10000)` | ✅ |
| `max_wall_clock_minutes` | int `>=1`, 기본 `120` | `z.number().int().min(1).default(120)` | ✅ |
| `max_member_turns` | int `>=1`, 기본 `500` | `z.number().int().min(1).default(500)` | ✅ |
| `base_dir` | optional string, 기본은 `~/.omo` 해석 | `z.string().optional()` | ✅ |
| `message_payload_max_bytes` | int `>=1024`, 기본 `32768` | `z.number().int().min(1024).default(32768)` | ✅ |
| `recipient_unread_max_bytes` | int `>=1024`, 기본 `262144` | `z.number().int().min(1024).default(262144)` | ✅ |
| `mailbox_poll_interval_ms` | int `>=500`, 기본 `3000` | `z.number().int().min(500).default(3000)` | ✅ |

**11개 전부 정확히 일치한다.** 기본 off. `packages/omo-opencode/src/config/schema/team-mode.ts` 는 `export * from "@oh-my-opencode/team-core/config"` 인 재export 한 줄이라 **스키마의 유일한 원본이 team-core**다. (루트 `AGENTS.md` 는 이 스키마를 `packages/omo-opencode/src/config/schema/team-mode.ts` 로 가리키는데, 그건 파생 파일이다.)

한 가지 주의: 루트 `AGENTS.md` 의 인용 JSONC는 `base_dir` 기본값을 "`~/.omo/teams` 또는 `<project>/.omo/teams`" 로 주석 처리하지만 `team-core/AGENTS.md` 는 기본 base directory 아래에 `~/.omo/teams/{name}/config.json`, `~/.omo/runtime/{teamRunId}/`, `~/.omo/worktrees/{teamRunId}/{member}/` 를 둔다. **주석이 실제 레이아웃과 어긋난다.**

### 스펙·런타임·멤버

`TeamSpecSchema`(`src/types.ts:56`): `version: 1`, `name`(`/^[a-z0-9-]+$/`), `description?`, `createdAt?`, `leadAgentId?`, `teamAllowedPaths?`, `sessionPermission?`, `members`(`.min(1).max(8)`).

**lead 부재 규칙**이 Zod로 박혀 있다: `superRefine`이 `leadAgentId === undefined && members.length > 1`이면 `MISSING_TEAM_LEAD_MESSAGE` 이슈를 낸다. `transform`이 없으면 **첫 멤버 이름을 lead로 채운다**. 즉 멤버 1명이면 lead 선언이 불필요하고, 2명 이상이면 필수다.

`RuntimeStateSchema`: `version: 1`, UUID `teamRunId`, `teamName`, `specSource: "project"|"user"`, `createdAt`, `status`(7개: `creating`/`active`/`shutdown_requested`/`deleting`/`deleted`/`failed`/`orphaned`), `leadSessionId?`, `tmuxLayout?`, `members`, `shutdownRequests`, `bounds`.

멤버 런타임 스키마는 `.strict()` 다. `status` 6개(`pending`/`running`/`idle`/`errored`/`completed`/`shutdown_approved`), `agentType: "leader"|"general-purpose"`, `pendingInjectedMessageIds`, `lastInjectedTurnMarker` 등 주입 상태 필드를 갖는다.

`TaskSchema`의 `blocks`/`blockedBy` 배열이 **팀 tasklist의 DAG**다 — DAG 엔진과는 별개지만 형태는 같다. 상태 5개: `pending`/`claimed`/`in_progress`/`completed`/`deleted`. `claim.ts` 가 원자적 claim을 담당한다.

### 멤버 적합성

`AGENT_ELIGIBILITY_REGISTRY`(`src/types.ts`) — **적합성의 진실 원본**:

| 판정 | 에이전트 |
|---|---|
| `eligible` | `sisyphus`, `atlas`, `sisyphus-junior` |
| `conditional` | `hephaestus` — `teammate: "allow"` permission이 없어서 D-36 적용 필요 |
| `hard-reject` | `oracle`, `librarian`, `explore`, `multimodal-looker`, `metis`, `momus`, `prometheus` |

거부 사유가 전부 **같은 논리로 같다**: "read-only여서 mailbox 파일을 쓸 수 없다". 팀 멤버는 durable inbox에 **쓰는** 방식으로 메일을 받으므로, 읽기 전용 에이전트는 구조적으로 불가능하다. `prometheus` 는 다르게 — "plan-mode-only, `prometheusMdOnly` 훅이 `.omo/*.md`만 쓰게 강제하므로 team mailbox에 쓸 수 없다".

> **문서↔코드 불일치 ⑤**: `docs/guide/team-mode.md:78` 의 거부는 **"curated read-only 에이전트(`explore`, `librarian`, `plan-consultant`, `plan-reviewer`)와 ulw-loop reviewer 3종"** 이라고 적는다. 하지만 `team-core/src/types.ts` 의 `AGENT_ELIGIBILITY_REGISTRY` 는 `explore`/`librarian` 와 **`plan-consultant`/`plan-reviewer` 를 아예 나열하지 않는다**. registry 는 `prometheus`·`metis`·`momus`·`oracle`·`multimodal-looker` 를 나열한다. 두 목록은 **성격이 다른 두 개의 거부**: `senpi-task/src/team/member-validator.ts` 의 curated read-only 집합(`CURATED_READONLY_AGENT_NAMES`)이 문서가 말하는 쪽이고, `AGENT_ELIGIBILITY_REGISTRY` 는 **OpenCode 어댑터**의 에이전트 이름 기준이다. 멤버 검증이 **어댑터별로 이중**이라 두 문서가 서로 다른 목록을 진술한다.

> **문서↔코드 불일치 ⑥**: `docs/guide/team-mode.md:68` 은 "**The lead is always the current session, so there is no lead member to declare**" 라고 단정한다. 코드에는 `TeamSpecSchema.leadAgentId` 와 그 superRefine 규칙이 있고, 루트 `AGENTS.md` 도 `leadAgentId required (or write a 'lead: {...}' field, or mark one member with 'isLead: true')` 를 인용한다. 문서가 옛 동작을 기술한 것이다 — lead는 현재 세션이므로 멤버로 선언하지 않는 현재 UI 관습과, 멤버 2명 이상일 때 스키마가 요구하는 `leadAgentId` 가 공존한다. **`team_create` 인라인 경로는 멤버 1명이므로 superRefine이 발동하지 않고, 파일 기반 스펙은 발동한다.**

### tmux 레이아웃·worktree·mailbox

- **worktree**: 멤버별 git worktree. 기본 `~/.omo/worktrees/{teamRunId}/{member}/`, `team-worktree/manager.ts` + `cleanup.ts`(고아 제거)
- **tmux**: `team-layout-tmux/layout.ts`(251줄) — focus + grid pane 배치, `sweep-stale-team-sessions.ts`로 세션 정리, `live-tmux-smoke.test.ts`는 `OMO_LIVE_TMUX=1` opt-in
- **mailbox 전달 모델은 injection-driven이고 예약 기반이다**. 전송은 durable unread JSON 파일을 쓰고 **반환**한다. 전달이 수신자의 실행 중 턴에 **steer**되지만 편집 가능한 후속으로는 큐잉되지 않는다. `listUnreadMessages` → `.delivering-<messageId>.json` → 수신자 세션에서 **확인된 뒤에만** `processed/<messageId>.json` 으로 커밋한다. **processed 파일이 durable exactly-once 원장**이다.
- lead poller는 `leadSessionId` 가 현재 세션과 같은 팀마다 하나씩이고, 어댑터는 `session_start` 와 **매 1초** 틱하지만 **compaction·세션 전환·셧다운 중에는 틱을 멈춘다**. 멤버 inbox는 어댑터가 절대 폴링하지 않는다 — 각 프로세스 멤버가 `member-extension/` 을 로드해 자기 inbox poller를 자식 프로세스 **안에서** 소유한다.
- `createSessionMarkerIndex` 는 세션 JSONL에 대한 경로별 증분 byte-offset 인덱스로, 틱마다 수십 번 일어나는 `messageId` 조회를 O(1) 로 만든다. 파일 잘림/로테이션을 처리하고, 파일이 안 늘었으면 아무것도 읽지 않는다.

## 교차 하니스 비교

[topics/orchestration.md](../topics/orchestration.md) 와 함께 읽을 것. 여기서는 DAG 엔진만 정면 비교.

| | **OmO senpi-task** | pi-mono `TaskGraph` | hermes kanban | claude-code `Workflow` | codex multi-agent | openclaw swarm |
|---|---|---|---|---|---|---|
| 그래프 형태 | 명시 DAG + 컴파일 산출물 | join 정책 그래프 | `task_links` 행 | 스크립트 DSL 결과 | parent/child **트리** | **그래프 DSL 없음**(자체 선언) |
| 그래프 출처 | 사람이 쓴 `dependsOn` | `owner` 필드가 곧 간선 | **LLM이 인덱스로 의존성 작성** | 스크립트 | 런타임 스폰 | `Promise.allSettled` 등 |
| 사이클 검출 | `findCycles` 결정적, 컴파일 실패 | — | `ValueError` raise | — | — | — |
| 스케줄링 | **frontier 승인** (의존 완료 + 슬롯) | `failFast`/`allSettled` | 부모 전부 terminal 게이트 | `parallel()` | 모델에 "N slots" 알림만(세마포어 아님) | FIFO 레인 |
| wave/barrier | 있으나 **정보용** | — | — | `phase()` | — | — |
| **원장** | **파일 WAL** (torn-tail 복구) | durable commit 파생 | SQLite | 결과 캐시(`resumeFromRunId`) | 없음 | 없음 |
| 체크포인트 | 있음 (`checkpointSeq`) | commit | 행 | — | — | — |
| **크래시 복구** | `createDagRecovery` — 소유자 대조 + replay + 결과 재사용 + lease watch | `resume()` 가 스케줄러 재시작 | 크로스 프로세스 체크리스트, 좀비/고아 스윕 | 캐시 재개 | — | — |
| 멱등 | `skipDuplicate`(#7412), `dagKeyHash` | `requestId` | — | 캐시 키 `(prompt, opts)` | — | — |
| 관측 채널 | **4 push + 4 RPC**, 시퀀스 원장 | 없음 | 14개 `kanban_*` 툴 | — | 6 툴 | workboard 카드 링크 |
| 격리 단위 | 세션(내부) → 프로세스 → 호스트 세션 | in-process 태스크 | **OS 프로세스** + env 펜스 | worktree/remote | 스레드 | 세션 |
| 아웃포盘中 | 데스크톱 연결 **미출시**(문서화만) | 예시 extension | 명령행 | CLI | 없음 | 없음 |

**OmO만 하는 세 가지**: ① 파일 시스템 WAL + torn-tail 복구, ② **자기 pid 를 못 믿는** lease 판정(#8006 — 다중 세션 호스트 환경에서 유일하게 실질적으로 필요한 수정), ③ **sequenced 원장 + 무시퀀스 라이브 텔레메트리를 같은 4채널로 분리**한 와이어 계약.

**claude-code `Workflow`** 에는 `resumeFromRunId` 라 부분 재실행이 있지만 결과 **캐시**다. OmO 는 **원장**이라 이벤트 단위 증감이 가능하고, 오버플로 복구 절차가 문서로 존재한다. **hermes** 는 DAG를 **저장소의 행**으로 선언하고 LLM이 인덱스를 써먹는다 — 의존성이 실행 결과가 아니라 프롬프트에서 나온다는 점에서 정확히 OmO의 DAG 계약(`dependsOn`은 **순서만**, 상위 출력을 하위 prompt에 치환하지 않음)과 정반대 철학이다.

## 실전 사용법

### 손대는 진입점

| 진입점 | 언제 | 기대 형태 |
|---|---|---|
| **`mass ulw <작업>`** | 명시적 순서가 있는 다수 작업 | 키워드 → `workflow` 툴. 그래프는 **eval 셀에서 코드로** 짠다 |
| **`/ulw-plan` → `/ulw-execute`** | brownfield·멀티파일, 범위를 먼저 쓰고 싶을 때 | 계획 산출물 → boulder 등록 → 체크박스 실행 |
| **`ulw`** | 좁은 목표, 결정은 에이전트에 맡길 때 | 바로 ultrawork 모드. plan 게이트 **열리지 않는다** |
| **`/goal <objective>`** | 세션에 붙박힌 지속 목표 | `goal.enabled: true` 필요. 완료 감사 없이 완료를 못 marking |
| **`ulw loop`** | 증거 바인딩 품질 게이트가 필요할 때 | reviewer 3종 게이트 |
| **`team_create`** | **같은 모듈을 만지는 겹치는 레인** + 도중 소통 필요 | lead + 멤버, 멤버별 worktree |

### 결정 플로우

```
빠른 수정? → 그냥 프롬프트
아니면:
  문맥 설명이 번거로운가? → ulw
  검토 가능한 계획이 필요한가? → /ulw-plan → /ulw-execute
  여러 작업 간 실제 순서가 있는가? → mass ulw
  겹치는 레인 + 도중 소통? → team mode
  그 외 → ulw
```

### 피해야 할 것

1. **settle 되면 새 run을 시작하지 말 것.** DAG description이 못 박는다: `retry` / `amend` / `send` 를 쓴다. fingerprint 재사용이 있으므로 같은 키로 재시작하면 run을 **재사용할 뿐** 고쳐지지 않는다
2. **멤버로 read-only 에이전트를 넣지 말 것.** `explore`/`librarian`/`oracle`/`metis`/`momus`/`prometheus` 는 하드 거부된다. `task` 툴로 위임할 것
3. **plan-gate를 모델로 열지 말 것.** `plan-consultant`/`plan-reviewer` 는 사용자의 명시적 `/ulw-plan` 요청 + 세션 내 `.omo/plans/*.md` 접촉 + `/ulw-execute` 미실행 3조건이 모두 필요하다. **미연결 dep는 fail CLOSED**다. `/ulw-execute` 가 부트스트랩한 계획은 게이트가 잠긴 채라 gap analysis·review가 없다
4. **`/ulw-execute` 전 bare `ulw` 로 review를 기대하지 말 것.** 그 경로는 notepad 자기검사로 대체된다
5. **pending 목록으로 `.omo/boulder.json` 삭제를 먼저 시도하지 말 것.** 다른 세션의 워크는 세션 id로 걸러진다. `/ulw-execute` 를 다시 돌리는 게 정답이다
6. **`ralph_loop` config를 쓰지 말 것.** deprecation 경고가 난다. `goal` 으로 옮긴다
7. **Desktop Workflows 패널을 신뢰하지 말 것.** `omo.dag.updated` 리듀서는 미출시다(reference 문서가 명시). `/dag` 슬래시를 대신 쓴다
8. **viewer를 만들면 `SENPI_RPC_CLIENT_CAPABILITIES=extension_events` 를 선언할 것.** 미선언 시 push 4채널이 조용히 죽고 request만 된다
9. **멀티 세션 호스트에서 DAG run이 `paused` 에 stuck 되면 기다릴 것.** lease watch가 이전 holder 종료 후 1회만 재실행하므로, 필요하면 세션 재개를 유발한다
10. **`dependsOn` 에 데이터 흐름을 기대하지 말 것.** 순서 전용이다. 상위 노드 출력을 하위 prompt에 넣어야 하면 그래프를 코드로 짜서 문자열을 조립할 것

### 검증 명령

```bash
bun test packages/senpi-task/src/dag          # DAG 서브시스템 전체
bun test packages/senpi-task/src/dag/scheduler.test.ts   # 승인 루프 집중
bun run typecheck                            # tsgo --noEmit
```

DAG 서브시스템 테스트는 `scheduler.test.ts` **1,956줄**, `store.test.ts` 1,106줄, `recovery.test.ts` 1,187줄로, 성격/결정 경계의 회귀 매트릭스가 이 엔진의 실제 품질 지표다. `dag_530ad299`(frontier 승인), `dag_2d12c2f7`(연쇄 스킵 지연), `dag_923ad20e`(멱등 중복), `#8396`(residency 대기), `#8006`(자기 pid lease)이 반복 등장하는 회귀 ID이고, 전부 이 문서에서 다룬 규칙에 대응한다.