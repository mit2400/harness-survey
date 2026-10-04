# OmO memory-core 딥다이브

> 소스: `packages/memory-core/` (commit `251cbfe`, 버전 `5.1.13`, 2026-10-04)
> 규모: `.ts` **230개**, 전체 **33,607 LoC** — 그중 비테스트 **13,982 LoC**, 테스트 **19,625 LoC** (테스트 비율 58%). 하니스 뉴트럴(`harness-neutral`) 엔진이며 어댑터는 `omo-opencode`/`omo-codex`/`omo-senpi`/`omo-native`가 별개로 붙인다.
> 상위 문서: [../oh-my-openagent.md](../oh-my-openagent.md) · 교차 하니스 비교: [../../topics/memory.md](../../topics/memory.md)

## 문서화 공백 (먼저 읽을 것)

`docs/` 전체를 `memory-core|MemFS|Kibitzer|memfs`로 grep하면 **4개 파일만** 맞고(`docs/guide/overview.md`, `docs/guide/orchestration.md`, `docs/reference/omo-json.md`, `docs/reference/configuration.md`), 그 4개 파일마저 **`omo.json`의 `memory.*` 설정 스키마만** 다룬다. 모듈 이름, 파일 경로, 상태 이름, 락 도메인, BM25 상수 같은 것은 한 줄도 없다.

즉 upstream 문서화 수준은 "설정값 목록"이고, 아래 9개 절의 전부는 소스에서 재구성한 것이다. 이 문서에서 등장하는 모든 상수(`K1=1.5`, `NUDGE_HINT_MAX_CHARS=200`, `REFLECTION_PARK_NON_RETRYABLE_STREAK=3` 등)는 소스에서 직접 읽은 값이다.

---

## 1. 패키지 레이아웃

`src/` 하위 디렉터리 19개. LoC는 **비테스트 파일만** 집계했다(테스트는 별도).

| 디렉터리 | 파일 | 비테스트 LoC | 책임 |
|---|---|---|---|
| `src/facts/` | 32 | 2,098 | 내구성 있는 사실 큐, 추출 라우팅, 실패 backoff/park, 복구 |
| `src/reflection/` | 22 | 1,740 | 반사 상태머신, worktree 실행, orphan sweep, park |
| `src/recall/` | 27 | 1,579 | **Kibitzer** — BM25 랭킹, 쿼리 플래너, nudge 게이트, 리드더 |
| `src/git/` | 23 | 1,395 | `GitMemoryRepo`, porcelain, path-state index, worktree 직렬화 |
| `src/locks/` | 19 | 1,236 | 9개 락 도메인, 후보 sweep, 프로세스 생존 판정 |
| `src/tools/` | 18 | 1,043 | `memory` / `memory_apply_patch` 툴 구현, 패치 파서 |
| `src/memfs/` | 14 | 992 | 경로 구속, YAML frontmatter 파서/렌더러, pre/post-commit 훅 |
| `src/search/` | 7 | 523 | 트랜스크립트 FTS-lite 검색 엔진, Senpi 세션 프로바이더 |
| `src/compile/` | 12 | 377 | 커밋된 메모리 → 시스템 프롬프트 `<memory>` 블록 |
| `src/concurrency/` | 3 | 199 | 두 프로세스 경합 테스트용 자식 프로세스 |
| `src/seeds/` | 5 | 292 | 기본 시드 파일 + memory-discipline 스킬 |
| `src/journal/` | 10 | 912 | 트랜스크립트 저널, 커서 스냅샷, 저널 락 |
| `src/people/` | 5 | 331 | 인물 카드 포맷 파서/직렬화, 슬러그 충돌 해소 |
| `src/fs/` | 6 | 302 | EINTR 재시도, 부분 쓰기 방지, `node:fs` 래퍼 |
| `src/sync/` | 5 | 321 | push 전용 미러, URL·비밀값 redaction |
| `src/identity/` | 8 | 204 | 에이전트 identity 해석, 온디스크 레이아웃 |
| `src/reminders/` | 3 | 227 | dirty/conflict/push 실패 리마인더 생성 |
| `src/soul/` | 4 | 153 | soul 경로 정의 + identity 스코프 notice 워터마크 |
| `src/personas/` | 4 | 40 | 페르소나 애셋 매니페스트(파일명 목록) |
| `src/*.ts` (루트) | 3 | 18 | `index.ts` 배럴 + 2개 테스트 |
| **합계** | **230** | **13,982** | |

`package.json`의 `description`이 정의를 내린다: *"Harness-neutral agent memory engine: git-backed MemFS, memory tools, prompt compiler, reflection state machine, and transcript search"*. subpath export는 5개 — `.`(배럴), `./fs`, `./personas`, `./process-identity`, `./process-start-time`. `typecheck`는 `tsgo --noEmit`, 테스트는 `bun test src/`. 런타임 의존성은 **0개**(devDependency에 `yaml` 하나뿐 — frontmatter round-trip 검증용).

### 온디스크 레이아웃

`src/identity/layout.ts`가 결정한다.

```
$OMO_MEMORY_HOME        (기본 ~/.omo/memory)
└── agents/<safe-id>/
    ├── repo/                     ← MemFS git 저장소 (마크다운 커밋)
    └── runtime/
        ├── locks/                ← 락 파일 9종
        ├── transcripts/          ← journal
        ├── reflection/           ← 상태 + park.json
        ├── reflection-sessions/
        ├── worktrees/            ← <epoch>-<runId> 반사 워크트리
        ├── viewers/
        ├── push-queue/
        ├── facts-queue/          ← 큐 + cursor/ + consumed.json + failures.json
        ├── facts/
        ├── notices/              ← soul-head.json 워터마크
        └── recall/
            ├── ledger/           ← 세션별 surfaced-path 원장
            └── pending/          ← 세션별 대기 nudge JSON
```

`RUNTIME_SUBDIRNAMES`는 11개 리터럴 상수 배열이고, `MemoryIdentityPaths`는 이 각각에 대해 타입 안전한 경로를 노출한다. `resolveMemoryRoot(env, cwd)`는 `OMO_MEMORY_HOME`이 있으면 cwd 기준 resolve, 없으면 `~/.omo/memory`. identity slug는 `MAX_SLUG_LENGTH=40`, 해시는 `SHORT_HASH_LENGTH=8`, 미지정 에이전트는 `AUTO_AGENT_VALUE="auto"` → `FALLBACK_SLUG="agent"`.

---

## 2. MemFS — git 백드 마크다운 메모리 파일시스템

메모리는 "파일 편집"이 아니라 "git 커밋"이다. 모든 쓰기 경로는 커밋을 만들어 세션 간에 이력을 남긴다.

### 저장 형태와 frontmatter 계약

`src/memfs/paths.ts`의 `MEMORY_CONTENT_PATH_RE`이 관리 대상 경로를 정의한다.

```ts
/^(?:memory\/)?(?:(?:system|reference|people)\/.*\.md|skills\/.+\/SKILL\.md)$/
```

즉 `system/`, `reference/`, `people/` 아래의 `.md`, 그리고 `skills/<name>/SKILL.md`만 frontmatter 계약 대상이다(`isMemoryContentPath`).

frontmatter 파서는 `src/memfs/frontmatter.ts`의 `parseMemoryFile` / `renderMemoryFile`이며 구분자는 `FRONTMATTER_RE = /^---\r?\n([\s\S]*?)\r?\n---\r?\n?([\s\S]*)$/`(`src/memfs/frontmatter-scalar.ts:11`). 파일 주석이 규칙을 명시한다:

| 키 | 규칙 |
|---|---|
| `description` | **필수**. 비어 있지 않고 단일 줄. `MAX_DESCRIPTION_LENGTH = 1024` |
| `read_only` | 문자열로 verbatim 보존(불리언으로 강제 변환하지 않음). `"true"`면 모든 수정 명령 거부 |
| `kind` / `aliases` | 인물 레코드 키. `aliases`는 **JSON 배열** |
| `limit` | 허용되되 무시되는 레거시 키 |
| 그 외 | `extra`에 raw scalar 소스로 보존(SKILL.md의 `name`/`version`/`deprecated` 등). 편집해도 절대 안 사라짐 |

렌더러는 두 방어를 겹쳐 둔다. `assertStrictRoundTrip`이 헤더를 `describeHeaderGrammarViolation`으로 재검증한 뒤 `parseMemoryFile`로 **다시 파싱해** 필드별 불일치를 확인한다 → "여기서 쓴 파일은 스킬 로더(`yaml`)에서 절대 파싱 실패하지 않는다". CRLF는 읽을 때 정규화, 쓸 때는 LF만.

`describeDescriptionViolation`에는 실전 휴리스틱이 하나 숨어 있다. `TOOL_CALL_SCAFFOLDING_RE = /<\/?(?:description|parameter)\b[^>]*>?/i`가 걸리면 "설명이 길고 여러 줄"이 아니라 **"이 툴 호출의 인자가 깨져서 잘못 분할됐다. 한 줄 description과 본문을 `file_text`로 다시 보내라"**라고 알려 준다 — 같은 증상을 다른 원인으로 환명하지 않기 위한 의도다.

### 경로 구속(confinement)

`validateMemoryPath`가 5중 방어를 한다: 빈 값·NUL 바이트 거부 → `~/`·`$HOME/` 거부 → 절대경로는 memory root 기준 상대화 → 세그먼트마다 `.`/`..`/`.git` 거부 → `realpathSync`/`lstatSync`로 **실제 심볼릭 링크를 따라 올라가며** root 밖 이탈 감시(`assertRealParentConfined`). 툴 라벨 경로는 확장자를 강제로 `.md`로 정규화하고(`system/contacts` → `system/contacts.md`), 저장소 경로(`delete`의 디렉터리 삭제)는 `validateRepositoryPath`가 쓰인다.

### 트랜잭션 / 원자성

핵심은 `src/tools/commit-write.ts`의 `commitMemoryWrite` 한 함수다. 이게 "트랜잭션 경계"다.

```ts
lock("memory-write", async () => {
  await repo.cleanCheck()                          // ① clean repo 강제
  const paths = await options.apply()              // ② 파일시스템 변경
  if (paths.length === 0) throw noChanges
  result = await repo.commitWrite(paths, msg, author)  // ③ 스테이징 + 커밋
  ...
})
```

4단계가 각각 방어를 담당한다.

| 단계 | 메커니즘 | 실패 시 |
|---|---|---|
| ① dirty repo 거부 | `repo.cleanCheck()` → `DirtyRepoError` | 커밋 없음 |
| ② 락 직렬화 | `src/locks/acquire.ts`의 `withLock("memory-write")` — 후보 파일을 `rename`으로 원자적 획득 | `LockContentionError` |
| ③ 락 소유자 사망 판정 | `isLockOwnerProvenDead` — PID 생존(`getPidLiveness`) **와** 프로세스 시작 시각(`getProcessStartIdentity`)을 모두 대조. PID 재사용으로 인한 오판 방지 | 락 회수 후 진행 |
| ④ 스테일 후보 sweep | `src/locks/candidate-sweep.ts`: `CANDIDATE_STALE_AGE_MS = 3_600_000`, `CANDIDATE_UNLINK_ATTEMPTS = 3` | 조용히 정리 |
| ⑤ 무효 커밋 방지 | `NoEffectiveChangesError` → `"made no effective changes"` 메시지로 변환 | 커밋 없음 |

커밋 메시지에는 프로브런스가 붙는다(`memoryCommitMessage`).

```
<reason>

Omo-Writer: memory-tool
Omo-Session: <sessionId>
Omo-Turn: <userTurns>
```

`Omo-Writer: memory-tool`은 **인밴드(자기 세션) 쓰기 표시**로, Soul 워터마크가 "이 커밋은 이미 사용자에게 통보된 것"으로 인식해 다시 공지하지 않기 위한 표식이다. 반환 메시지는 원격 존재 여부에 따라 갈린다: 로컬 전용이면 `"committed locally (abc1234)"`, 원격이 있으면 `"committed (abc1234); harness will sync after the turn."`. 그리고 경로가 `SOUL_PATHS`(`system/persona.md`, `system/identity.md`, `system/boundaries.md`)에 닿으면 `SOUL_EDIT_RESULT_LINE`("This was a soul edit: announce it to the user in your reply.")이 덧붙는다 — persona를 바꾼 건 숨기지 않고 반드시 알리라는 요구.

부수 기구:
- `src/git/config-lock.ts` — `withSerializedGitConfigMutation`, `withGitLockRetry`, `isGitLockError`. git config 동시 쓰기 직렬화.
- `src/git/worktree-mutation-queue.ts` — `withSerializedGitWorktreeMutation`. 워크트리 변경 직렬화.
- `src/git/path-state-files.ts` — 워크트리 파일의 조건부 쓰기/이동/복구, `reserveMovedWorktreeDeletion`으로 삭제 예약.
- `src/git/path-state-index.ts` — `writeIndexIfIdentity`. 파일 내용이 같으면 스테이징하지 않는다.
- `src/fs/write-all.ts` — `writeHandleAll` / `writePathAll`로 부분 쓰기 금지, `openSyncWithExclusivePolicy`가 `O_EXCL` 플래그 처리.
- `src/fs/retry.ts` — `EINTR_RETRY_CAP = 128` 재시도.

### 툴 표면

**`memory`** — `src/tools/memory.ts`. `runMemoryTool(options)` 한 진입점, `MemoryCommand` 유니온이 7개 명령을 정의한다.

| `command` | 필수 인자 | 동작 |
|---|---|---|
| `create` | `file_path`, `description` | 파일 없음 확인 후 생성. `file_text` 없으면 빈 본문 |
| `str_replace` | `file_path`, `old_string`, `new_string` | 본문에서 `indexOf`로 첫 매치를 정확히 치환. 없으면 `old_string was not found` |
| `insert` | `file_path`, `insert_line`, `insert_text` | 줄 단위 삽입. `insert_line`이 숫자 아니면 거부, 범위 초과 시 끝에 붙임 |
| `delete` | `file_path` | 파일 또는 **디렉터리 전체** 삭제. 디렉터리면 하위 `.md`를 재귀 검사해 `read_only` 위반을 먼저 잡음 |
| `rename` | `old_path`, `new_path` | 목적지 존재 시 거부 |
| `update_description` | `file_path`, `description` | frontmatter만 교체 |
| `apply_patch` | `input` | `applyMemoryPatch`로 위임(`command` 문자열로는 도달 불가, `memory_apply_patch` 툴이 주 경로) |

공통 규칙:
- `reason`은 모든 명령에 필수. 커밋 메시지 첫 줄이 되고 `MemoryToolCommit.subject`가 된다.
- 모든 명령은 `loadEditable`을 통과해야 한다 → frontmatter 파싱 성공 + `read_only != "true"`.
- 인코딩 방어: `readUtf8`가 BOM(UTF-16 LE/BE)을 감지하면, 그리고 `TextDecoder(fatal: true)`가 실패하면 "UTF-16/잘못된 UTF-8" 에러. **조용히 깨진 채로 쓰지 않는다.**
- 인자 누출 복구: `src/tools/leaked-arguments.ts`의 `repairLeakedArguments`가 형제 인자가 description에 섞인 경우(`<description>` 태그 등) 잘라내고, `describeRepairs`로 무엇을 고쳤는지 사용자에게 알린다.

**`memory_apply_patch`** — `src/tools/memory-apply-patch.ts`. `runMemoryApplyPatch`는 자체 `MemoryApplyPatchLock`을 갖고 **동일한 3단계**(cleanCheck → apply → commitWrite)를 밟지만 트랜잭션 정의가 더 엄격하다. `applyOperations`가 변경을 `pendingWrites`/`pendingDeletes`에 모았다가 한 번에 flush한다 — 부분 적용에 의한 중간 상태를 없앤다. 패치 문법은 `src/tools/patch-parser.ts`의 `parseMemoryPatch`(불명확한 컨텍스트는 `MemoryPatchHunkError`), 적용은 `src/tools/patch-apply.ts`의 `applyMemoryPatch`.

에러 타입: `MemoryToolError`, `MemoryApplyPatchError`, `MemoryPatchParseError`, `MemoryPatchHunkError`, `MemoryPathError`, `FrontmatterError`(`src/tools/tool-errors.ts`).

### git 훅

`src/memfs/hooks.ts`의 `installHooks(repoPath)`가 pre-commit / post-commit 훅을 설치하고 `src/memfs/hooks-scripts.ts`가 스크립트를 정의한다. pre-commit 훅은 **frontmatter 규약을 강제**하는 마지막 방어선이다. post-commit 훅은 `getPostCommitHookScript()`(`src/sync/mirror.ts`가 export) — 미러 push를 트리거한다. `resolveCommonGitDir`은 `.git`이 파일(worktree/submodule)인 경우까지 처리한다.

정규화 도구: `src/memfs/normalize.ts`의 `normalizeMemoryFrontmatter`가 `FRONTMATTER_NORMALIZATION_MARKER = "omo-memory-frontmatter-normalized"` 마커와 `FRONTMATTER_NORMALIZATION_SUBJECT = "chore(memory): normalize frontmatter to strict YAML"` 커밋으로 헤더를 강제 YAML에 맞춘다. `frontmatterNormalizationPending(repo)`로 미적용 상태를 감지한다.

---

## 3. Kibitzer — BM25 리콜 사이드카

Kibitzer는 **메모리 파일 목록을 프롬프트에 흘려보내지 않는다.** 검색 후보를 낼 뿐이고, 최종 노출은 별도의 작은 모델이 "judge"로 결정하며, 그 결과조차 **직접 주입이 아니라 nudge(끌지르기)** 다. 이 3단 분리가 이 서브시스템의 핵심 설계다.

### 3-1 무엇을 색인하는가

`src/recall/provider.ts`의 `loadRecallCorpus(repo)`가 만드는 `RecallCorpus = { revision, documents }`, `RecallDocument = { path, description, body }`.

선택 규칙 `isRecallCandidatePath`:
- `.md`만
- 제외는 **루트 `system/` 트리뿐**. 주석이 이유를 명시한다: "`reference/system/deploy.md`는 평범한 사용자 메모리이므로, 세그먼트가 어디에든 등장한다는 이유로 매칭하면 recall 가능한 파일이 조용히 사라진다."
- `parseMemoryFile` 실패 시 **fail-closed**로 조용히 스킵

두 불변식이 코어에 박혀 있다.
- **Compile-from-committed**: 모든 읽기는 HEAD revision의 커밋 트리만 통과한다. 워킹 트리는 절대 참조하지 않으므로 미커밋 편집이나 미추적 파일이 리콜 코퍼스에 새어들 수 없다.
- **프로세스 예산**: 로드는 `ls-tree` **1회 + `cat-file --batch` 최대 1회**. 주석이 문제를 명시한다: "수천 파일짜리 메모리 저장소는 메모리 쓰기마다 HEAD가 움직이고, 살아 있는 모든 세션이 파일마다 `git show`를 하면 머신 전체에 git 폭풍이 일어난다."

증분 캐시: `RecallCorpusCache`가 blob oid별로 이전 결과를 재사용한다. HEAD 이동 비용은 `ls-tree` 1회 + 변경 파일에 대한 `cat-file --batch` 1회. oid는 내용 주소이므로 재사용된 문서는 바이트 단위로 동일하다.

BM25 인덱스도 `WeakMap`로 문서 배열에 캐싱된다(`src/recall/bm25.ts`). `strategy.ts`의 `CJK_SHARES`도 같은 패턴이다. 주석이 규칙을 명시: **"최초 랭킹 이후 배열을 in-place로 바꾸면 낡은 인덱스가 남는다. `RecallCorpusCache`는 revision마다 새 배열을 준다."**

### 3-2 전략 선택

`src/recall/strategy.ts`의 `chooseRecallStrategy(documents, queries)`.

| 전략 | 조건 | 성격 |
|---|---|---|
| `substring` | 그 외 (영문 전용 쿼리 + 작은 코퍼스) | 기존 FTS-lite AND 스코어. **영문 전용 사용자에게는 예전 후보와 완전히 동일한 결과를 유지** |
| `bm25` | 쿼리 중 하나라도 CJK 포함, **또는** 코퍼스 CJK 비중 ≥ `CJK_CORPUS_MIN_SHARE = 0.1` | OR 스코어 BM25 + CJK 바이그램 |
| `hybrid` | 문서 수 ≥ `LARGE_CORPUS_MIN_DOCUMENTS = 200` | substring + bm25를 reciprocal rank fusion. bm25보다 우선(bm25 쪽이 이미 CJK 바이그램을 갖기 때문) |

사용자에게 노출되는 랭커 설정은 없다. 주석이 설계 의도를 말한다: "리트리버는 후보를 랭크하는 방법을 스스로 고르고, 하류의 Kibitzer judge가 정밀도를 지킨다. 그래서 리트리버는 리콜 쪽으로 기운다." 임계값은 `packages/omo-senpi/scripts/qa`의 `recall-ranker-bench.mjs`에서 나왔다.

### 3-3 BM25 스코어링

`src/recall/bm25.ts`. Okapi BM25에 `K1 = 1.5`, `B = 0.75`.

**토크나이저**가 이 구현의 진짜 핵심이다. `CJK_CLASS = \p{Script=Hangul}\p{Script=Han}\p{Script=Hiragana}\p{Script=Katakana}\u30fc`(`U+30FC`는 Script=Common이지만 Kana 런에 속하므로 명시적으로 포함). 처리 규칙:

1. NFKC 정규화 + 소문자. NFD 한글(macOS 파일명에서 붙어 오는 흔함)을 합성하고 전각 라틴을 접는다 — 양쪽이 한 형태에서 만난다.
2. CJK 런이 2자보다 길면 **문자 바이그램**을 추가한다. "한국어 굴절된 쿼리 단어가 저장된 어간과 접두사를 공유하도록" — 형태소 분석기 없이.
3. CJK 런 안의 **한자**(Han) 문자는 독립 term으로도 추가한다. 주석: "중국어 단어나 일본어 한자어는 자주 한 글자이고, 어떤 바이그램도 그것을 고립시키지 못한다."
4. 한글·가나 문자는 독립 term으로 추가하지 않는다. 주석: "그래서 홑 한글이나 홑 가나 글자는 동일 토큰에만 매치하고, 더 긴 단어와 절대 매치되지 않는다."
5. 영문은 `stemEnglishToken`(`src/recall/english-stem.ts`)으로 접미사 접기.

idf는 `Math.log(1 + (N - df + 0.5) / (df + 0.5))`, tf 항은 표준 형태. 문서 길이는 독립 한자 개수를 빼서 계산한다(중복 카운트 방지).

**쿼리 확장** — `RECALL_EXPANSION_WEIGHTS = { synonyms: 0.75, keywords: 0.75, related: 0.4, noteLine: 0.4 }`. `RecallQueryExpansions` 네 티어(synonyms / keywords / related / noteLine). 확장 term은 가중치만큼 BM25 스케일로 곱해져 더해진다. 쿼리에 이미 있는 term은 제외되고, 여러 티어에 나온 term은 최고 가중치를 유지한다. **가중치는 birkin-mnemosyne 검색 벤치마크 dev split에서 측정한 값**이다. 중요한 불변식: 쿼리의 모든 unit을 다 잡은 문서는 확장 점수를 받지 않고 `fullMatch: true`로 맨 앞에 올라간다. 주석: "정확 조회가 오늘 반환하는 것을 그대로 반환한다." — 확장은 순위를 뒤집지 않는다.

BM25 알고리즘 자체는 `birkin-mnemosyne`(한국어 인식 바이그램 BM25 메모리 저장소)에서 차용했다(파일 헤더에 출처 명시).

### 3-4 후보 선택과 excerpt

`src/recall/select.ts`. `selectRecallCandidates(documents, queries, options)`가 `RecallCandidate { path, description, excerpt, score }`를 반환한다. **score 계약은 오름차순(작을수록 좋음)** — FTS-lite와 맞추기 위함.

- `RRF_K = 60`(Cormack et al.의 관례값). hybrid는 `1/(1 + fused)`, bm25는 `1/(1 + bm25)`.
- substring 투표는 **전부 1등과 동일하게** 취급한다. 이유가 주석에 있다: "substring 순서는 첫 매치의 오프셋이지 관련도가 아니다. 패딩된 벤치마크 코퍼스에서 순위 가중 substring 투표는 답보다 채움 노트를 위로 올렸다."
- `phraseLeader`/`pinPhraseLeader`: 비-CJK 다단어 쿼리에서 정확 구절은 lexical 증거 중 최강이다. 그 노트의 1등 자리를 고정해 fusion이 내리지 못하게 한다.
- `EXCERPT_CHARS = 200`. "의도적으로 설정 노트가 아니다"는 주석이 붙는다.
- 제외: `surfaced`(이번 세션에서 이미 노출됨, `options.excludePaths` 추가 가능)

### 3-5 nudge-only 게이트 — 왜 이 설계인가

`src/recall/gate.ts`의 헤더 주석이 이 파일의 설계 논리를 통째로 담고 있다.

> Kibitzer 출력 계약: 상주 in-process 사이드카는 nudge 툴만을 통해서만 말한다. 그 클로저는 수용된 각 nudge를 제공된 경로에against 원장한다. **부모가 권위(authoritative)** — 수집된 모든 nudge는 후보 집합, 세션 원장, 힌트 형태, 설정된 상한에 대해 재검증된다(방어 심화 — 클로저는 호출 시점에 이미 같은 규칙을 적용했다).

세 겹의 방어가 있다.

**1)Admission 규칙** — `describeInvalidHint(hint): InvalidHintReason | undefined`. 거부 사유는 순서대로 5개:

| 사유 | 조건 |
|---|---|
| `empty` | 길이 0 |
| `too-long` | `NUDGE_HINT_MAX_CHARS = 200` 초과 ("사실 한 문장" 예산) |
| `multiline` | 개행 포함 |
| `decision-commentary` | `NUDGE_DECISION_LANGUAGE_PATTERN` — "no stored memory", "clears the bar", "memory does not cover" 류의 판단 코멘터리 |
| `addresses-agent` | **에이전트에게 말하는 형태** |

마지막 규칙이 이 설계의 핵심이다. nudge 블록은 "저장된 노트에 대한 참고 자료"이지 지시문이 절대 아니다. 주석:

> nudge 블록은 저장된 노트에 대한 참고 자료이지, 결코 지시문의 한 줄이 아니다: 주 에이전트의 자신의 작업이 그대로 서고, 노트를 어떻게 쓸지는 그 에이전트가 결정한다. **에이전트를 향해 말하는 힌트는 따라서 입장 시점에 거부된다.** 그 목소리를 띤 세 가지 형태: 2인칭, 문장 시작의 명령(또는 부정 명령), 한국어 의/request 종결. **명사 동사만 스캔한다** — 문장 중간에 규칙을 인용하는 관찰("the note records that publish must follow the guard")은 유효하게 남는다.

`NUDGE_SECOND_PERSON_PATTERN`(`you|your|yours|yourself`), `NUDGE_IMPERATIVE_OPENING_PATTERN`(문두 `do not|don't|never|always|make sure|ensure|verify|check|run|use|read|stop|avoid|remember|keep|prefer|skip|consider`), `NUDGE_KOREAN_REQUEST_ENDING_PATTERN`(`세요|십시오|십시요|하라|해라|합니다|지 마(라)` — 주석에 "그리고 오타 십시요" 명시), `NUDGE_KOREAN_PROHIBITION_PATTERN`(`하지 마`). 평서형(`...한다`, `...이다`)은 건드리지 않는다.

**REPLAY vs ADMISSION** — 이 파일의 가장 미묘한 설계다. `isValidHint`는 `describeInvalidHint`를 **의도적으로 쓰지 않는다.**

> 힌트 예산 판정기: 사실 한 문장, 비어 있지 않고, 최대 `NUDGE_HINT_MAX_CHARS`, 단일 줄, nudge 판단 코멘터리 없음.
> 이것은 계약의 **REPLAY 절반**이며, `describeInvalidHint`가 아닌 것이 의도적이다: 대기 페이로드와 저장된 `omo-kibitzer:nudged` 엔트리는 **수용되던 시점에 성립하던 계약** 아래에서 들어왔으므로, 그것들을 다시 읽을 때는 지시문 형식으로 쓰인 nudge를 소급해 버리면 안 된다. **admission이 nudge-only 규칙이 실제로 물리는 지점**이다.

이 분리 없이는 계약 강화가 과거 기록을 조용히 망가뜨린다.

**2) 부모 재검증** — `validateNudges(nudges, { candidates, surfaced, maxItems })`. 부모는 권위이므로 사이드카가 조작한 출력을 방어한다. 순서: 상한(`memory.recall.max_items`, 1..5) → 중복 path → **후보 집합에 없으면 조작으로 간주해 버림** → 이미 노출됨 → 힌트 형태. `containsSecretLikeMaterial`(§5 Sync)도 게이트에서 재사용된다.

**3) 전달 큐** — `PendingNudges` 클래스, 세션당 JSON 파일 1개.
- 쓰기: 디렉터리 `mode: 0o700`, 임시 파일 `mode: 0o600` → `rename` 원자적 교체.
- 페이로드 자기서술형: `{ version: 1, sessionId, writtenAt, nudges }`. 세션 id 새니타이즈 충돌로 한 세션이 다른 세션의 nudge를 받는 일이 구조적으로 불가능.
- `take()`: 파일 없음/파싱 실패/세션 불일치/만료 → nudge 없음(fail-closed). 만료는 `PENDING_TTL_MS = 24h`.
- `delete()`: 컴팩션이 수용되는 **그 순간** 해당 세션 페이로드를 철회(retract). 파일 안의 `sessionId`를 `take()`와 동일하게 검증한 뒤 삭제 — 새니타이즈 충돌 때문에 무가드 unlink은 다른 세션의 nudge를 철회할 수 있다.
- `prune()`: 24시간 지난 형제 페이로드와 `.tmp-*` 오버런 제거.
- `removeQuietly`는 **fail-open**: 걸린 pending 파일이 턴을 깨면 안 된다.

**렌더링** — `src/recall/render.ts`.

```xml
<recalled-memory source="[[people/kibitzer.md]]">
Kibitzer, a background memory advisor, surfaced this stored note. It may or may not apply: reference only; your current task stands.
<judge의 한 문장 힌트>
</recalled-memory>
```

`RECALL_HINT_HEADER_KO`는 "백그라운드 메모리 조언자 키비처가 짚어준 저장 메모입니다. 맞을 수도 아닐 수도 있으니 참고만 하고, 하던 작업은 그대로 이어가세요." 힌트에 한글이 있으면 한국어 헤더가 선택된다. 헤더는 **보내는 자와 태도(참고만, 현재 작업 유지)**를 명시하므로 블록 자체가 지시를 갖지 않는다. 경로는 힌트가 생략한 상세가 필요할 때를 위해 남겨 둔다.

**왜 nudge-only인가 — 정리**

| 대안 | 문제 |
|---|---|
| 검색 결과를 프롬프트에 직접 인젝트 | 사전순/점수순 후보가 매 턴 컨텍스트를 오염. 관련 없는 메모리가 프롬프트를 계속 지배 |
| 전용 recall 툴 제공 | 모델이 "기억을 찾아야겠다"를 자각해야 함. `compile.ts`의 REMINDER가 명시하듯 **"there is no recall tool to call"** |
| **nudge-only (채택)** | 리트리버는 리콜 지향으로 폭넓게, judge가 정밀도를 유지. 부모는 권위로 재검증. 결과는 참고 블록 하나이고 **자기 작업을 계속** |

`src/locks/recall-wake-domain.ts`가 깨어 있는 사이드카 수를 제한한다: `RECALL_WAKE_DEFAULT_SLOTS = 2`, `RecallWakeBusyError`, 티켓 디렉터리 + `withRecallWakeLease`. 원장(`src/recall/ledger.ts`)은 세션별 surfaced path와 디스크 읽기 횟수(`recallLedgerDiskReads`)를 기록한다.

---

## 4. Reflection — 반사 상태머신

반사는 "완료된 대화에서 durable note를 뽑아 메모리에 써넣는" 배치 작업이다. 구현은 `src/reflection/machine.ts`.

### 상태와 트리거

`ReflectionTrigger = "step-count" | "compaction" | "manual" | "dream"`, `DreamOrigin = "manual" | "idle" | "shutdown" | "pressure"`.

`ReflectionOutcome`(실행 결과)은 7개: `merged`, `no_changes`, `parent_dirty`, `merge_conflict`, `dirty_uncommitted`, `failed`, `timed_out`.

상태 자체는 `MachineState` 하나다: `{ now, journal: { conversationId, state, snapshot }, reservation: { active?, pending? }, config }`. 상태 머신이 별도 열거형으로 존재하지 않는다는 점이 중요하다 — **예약 슬롯 2개(active/pending)가 유일한 상태**이고, 트리거 조건은 저널에서 파생된다.

이벤트는 3개: `settled { success }`, `compaction_accepted`, `manual { focus?, recentN?, conversationIds? }`.

`evaluateTransitions(state, event)` 우선순위 판정 순서:

| 순위 | 트리거 | 조건 |
|---|---|---|
| 즉시 | `manual` | 항상 reserve (park 제외는 §4-3) |
| 1 | `step-count` | `steps_since_last_successful_reflection >= trigger.step_count` (기본 25) |
| 2 | `compaction` | `config.onCompaction === true && journal.state.pending_compaction === true && active/pending에 compaction 트리거 없음` |
| 3 | byte 압력 | `unreflected_bytes >= snapshotMaxBytes` — **자기 자신의 트리거로 reserve** |

byte 압력이 독립 트리거인 이유가 주석에 있다: "단계 몇 개가 매우 크면 단계 임계값이 훨씬 먼저 오지만, 각 캡처는 한 예산만큼만 비우기 때문이다."

`compaction_accepted`는 다른 이벤트와 달리 **상태만 바꾼다**(action 없음) — `pending_compaction: true`를 세팅. 컴팩션 방어 훅(§6)이 이 신호를 memory-core에 넘기는 접합점이다.

### 예약 병합과 백로그 상한

`reserveTransition`은 active 슬롯이 차 있으면 pending으로 밀고, `mergeRequests`가 합칩니다. 우선순위 숫자:

| ReflectionTrigger | priority |
|---|---|
| `step-count` | 1 |
| `compaction` | 2 |
| `manual` | 3 |
| `dream` / `idle` | 1.5 |
| `dream` / `pressure` | 1.5 |
| `dream` / `shutdown` | 2.5 |
| `dream` / `manual` | 3 |

dream은 `origin`이 없으면 `requestPriority`가 `TypeError`를 던진다 — 의도된 방어다.

`capPendingRun`이 공유 pending 슬롯을 **32개 대화 / 4 MiB UTF-8 JSON**으로 제한한다(`REFLECTION_PENDING_MAX_CONVERSATIONS`, `REFLECTION_PENDING_MAX_BYTES`). 주석: "대략 32개의 기본 128 KiB 캡처 윈도우이며, 무한정 자라는 멀티세션 백로그 대신." 제거는 **첫 등장 순으로 통째로 대화 단위**로 이뤄지고, 그 대화의 저널 커서는 재시도 가능 상태로 남는다. 바이트 계산이 세밀하다 — 스냅샷은 pending.json 안에서 6스페이스 들여쓰기라 물리적 JSON 줄 수 × 6을 더하고, 배열 괄호/공백까지 16바이트를 보수적으로 과대계산한다.

`completeTransition`은 성공 판정(`merged || no_changes`)일 때만 스냅샷 커서를 전진시키고 `steps_since_last_successful_reflection`를 갱신하며, compaction 트리거였다면 해당 대화의 `pending_compaction`을 지웁니다. 실패한 실행의 스냅샷은 `finalize = []` — 커서가 전혀 전진하지 않아 다음 트리에 재시도됩니다.

### worktree 기반 실행

`src/reflection/worktree.ts`가 `createReflectionWorktree` / `discardReflectionWorktree` / `finalizeReflectionWorktree`를 제공합니다. 반사는 메인 메모리 저장소를 직접 수정하지 않습니다. `<runtime>/worktrees/<epoch>-<runId>`에 git worktree를 만들고 거기서 작업한 뒤 통합합니다. 브랜치 규약은 `memory/reflection-<epoch>-<runId>`입니다.

`src/reflection/worktree-integration.ts`가 통합 정책을 정합니다: `ReflectionIntegrationMode = "auto" | "integration"`(`memory.reflection.merge` 설정에 대응), `probeReflectionIntegration`, `integrateValidatedReflection`, `cleanupReflectionWorktree`, `probeLegacyAutoRunReceipt`.

예약 메커니즘은 `src/reflection/reservation.ts`의 `ReflectionReservationStore`(메서드 레벨 테스트 가능하도록 분리) + `src/reflection/reservation-files.ts`(`readRun`, `writeJsonAtomic`, `sweepReservationTemporaries`, `currentLauncherIdentity`). 런처 신원(`ReflectionLauncherIdentity`)에 pid·hostname·processStart가 들어가므로, 죽은 런처의 예약을 살아 있는 프로세스가 지우지 않는다.

### orphan sweep

`src/reflection/orphan-sweep.ts`. 반사 잔여물은 그것을 소유한 런보다 오래 삽니다: 죽은 수퍼바이저, 철거된 워크트리, `git worktree add`와 런 원장 사이의 크래시. **finalize는 런 디렉터리에서 도달 가능한 자원만 치우므로, 이 sweep만이 나머지를 회수하는 유일한 경로**다.

`listReflectionLeftovers`가 3류를 수집한다:
1. 등록된 worktree (`git worktree list --porcelain` 중 `worktrees/` 아래 것)
2. `strayDirs` — `^\d+-` 패턴 디렉터리 중 등록되지 않은 것
3. `branches` — `for-each-ref`로 `refs/heads/memory/reflection-*`와 레거시 `refs/heads/reflection/run-*`

소유권 판단(`selectReflectionOrphans` — 순수 함수라 정책만 단독 테스트 가능):
- `parseReflectionOwner`가 `<epoch>-<runId>`에서 runId와 epoch를 추출. 레거시 `reflection/run-<n>`은 epoch 규약 이전이므로 **grace를 받지 않는다**.
- `isLive()`: `liveRunIds.has(runId)` **또는** `liveRunIds.has("reflection-" + runId)`. 주석: "레거시 브랜치는 `run-<n>`과 `reflection-run-<n>` 양쪽으로 기록되었다. 어느 쪽 주장도 보호한다."
- `REFLECTION_ORPHAN_GRACE_MS = 15 * 60_000` — "prelaunch 디렉터리와 그 원장 사이의 윈도우, 그리고 클록 스큐"를 커버.
- 디렉터리가 사라진 등록 항목은 **재생성될 수 없으므로 무조건 prune**, 있는 항목은 grace 경과 후에만.

`sweepReflectionOrphans`는 항목별로 `attempt()`가 개별 `ReflectionOrphanReceipt { kind, target, removed, detail? }`를 반환한다 — **실패 항목 하나가 sweep 전체를 중단시키지 않는다.** 디렉터리 삭제 전에는 `isInside(root, resolve(dir))`로 경로 이탈을 재차 검사하고, 마지막에 `git worktree prune`을 한 번 돌립니다. git 호출은 `GIT_TIMEOUT_MS = 30_000`과 `GIT_TERMINAL_PROMPT=0`을 강제합니다.

### park policy

`src/reflection/park.ts`는 자동 반사의 서킷 브레이커입니다(PR #8304). 헤더 주석이 논리를 완전히 설명합니다.

> 결정적 실패(모델 없음, 부팅 크래시, 샌드박스 거부)는 매 재시도마다 동일하게 반복되므로 **3회 연속이면 identity를 park**한다. 일시적 실패(rate limit, 제공자 장애)는 **2배의 여유**를 받고, 결정적 실패(설정이 바뀌기 전까지 고칠 수 없음이 증명됨)는 **첫 발생에서** park한다. park 중에는 **인터벌당 반-open probe 한 번**으로 self-healing을 유지하되 호스트를 두드리지 않는다.

| 상수 | 값 |
|---|---|
| `REFLECTION_PARK_NON_RETRYABLE_STREAK` | 3 |
| `REFLECTION_PARK_RETRYABLE_STREAK` | 6 |
| `REFLECTION_PARK_PROBE_INTERVAL_MS` | `6 * 60 * 60_000` |
| `REFLECTION_PARK_DETAIL_MAX_CHARS` | 512 |

`ReflectionParkGate` 3종: `{ kind: "open" }` / `{ kind: "probe" }` / `{ kind: "parked", nextProbeAt }`. 상태는 `ReflectionParkState { version: 1, streak, firstFailureAt?, lastFailure?, parkedAt?, lastProbeAt? }`이고 `runtime/reflection/park.json`에 저장(`src/reflection/park-file.ts`의 `PARK_FILENAME`). `isAutomaticReflectionRequest`가 manual/dream-manual을 제외하고 `gateReflectionRequest`를 통과시킨다. **읽을 수 없는 park 상태는 `ReflectionParkStateError`/`isUnreadableReflectionState`로 분리**되어, 손상 파일이 park를 조용히 풀어버리는 일이 없다.

### 완료 검증

`src/reflection/completion-validation.ts` — `validateCompletion`, `validateDreamTokenBudget`(→ `DreamTokenBudgetValidation`). 반사가 산출물을 다 쓴 뒤 예산/형식 검사를 통과해야 큐에 올라갑니다.

---

## 5. 서브엔진 11종

| 이름 | 책임 | 대표 파일 | 핵심 상수/설정 |
|---|---|---|---|
| **Soul** | 에이전트 자아(persona/identity/boundaries) 경로와 변경 통지 | `src/soul/paths.ts`, `src/soul/watermark.ts` | `SOUL_PATHS = ["system/persona.md","system/identity.md","system/boundaries.md"]`, `SOUL_NOTICE_WATERMARK_FILENAME = "soul-head.json"`, `SOUL_SCAN_PAGE = 32`, `MEMORY_SOUL_EDIT_RESULT_TOKEN = "soul edit"` |
| **Facts** | 대화에서 durable fact을 추출해 큐에 넣고 메모리에 라우팅 | `src/facts/queue.ts`, `schema.ts`, `extraction.ts`, `person-routing.ts`, `failures-*.ts`, `payload-cap.ts`, `recovery*.ts`, `mutation-plan.ts` | `FACTS_QUEUE_VERSION = 1`, `MAX_FACTS_PAYLOAD_BYTES = 131_072`, `FACTS_STARVATION_MS = 24h`, `FACTS_FAILURE_REASONS`(7종), `FACTS_FAILURES_VERSION = 1` |
| **People** | 인물 카드 파서/직렬화 | `src/people/format.ts` | `CardPrefix = IDENTITY \| ATTRIBUTE \| RELATIONSHIP \| INSTRUCTION`, `sanitizePersonSlug`, `isReservedSlug`, `resolveSlugCollision` |
| **Dream** | 유휴 시 통합 반사 | `src/reflection/`의 `dream` 트리거 + `src/reflection/assets/assets.ts`의 `loadDreamPersona()` | `DreamOrigin = manual \| idle \| shutdown \| pressure`, `memory.dream.{idle_minutes:30, min_hours_between:24, auto_select_max:5, auto_select_max_chars:150000}` |
| **Compile** | 커밋된 메모리 → 시스템 프롬프트 블록 | `src/compile/compile.ts`, `render.ts`, `cache.ts`, `changes.ts` | `MEMORY_TEMPLATE_STRUCTURE_VERSION = "senpi-memory-v2"`, `MEMORY_SESSION_TRAILER = "Omo-Session"`, 센티넬 `<!-- senpi-memory:<identity>:begin -->` |
| **Search** | 트랜스크립트 FTS-lite 전문 검색 | `src/search/engine.ts`, `query.ts`, `senpi-session-provider.ts` | 기본 limit 100, 점수 오름차순 + 날짜 내림차순 타이브레이크, hidden 대화 제외 |
| **Sync** | push 전용 미러 | `src/sync/mirror.ts`, `redact.ts` | `CONFIG_KEY = "omo.memoryRepository.url"`, `LOG_NAME = "memory-repository-push.log"`, `SYNC_PUSH_ENV = "OMO_MEMORY_PUSH_SYNC"`, `main` 브랜치만 |
| **Journal** | 트랜스크립트 append + 반사 커서 | `src/journal/store.ts`, `lock.ts`, `entries.ts`, `cursor.ts`, `fsync.ts` | `TOOL_ARGS_TRUNCATE_LIMIT = 300`, `TOOL_RESULT_TRUNCATE_LIMIT = 4096`, `REDACTED_REASONING_TEXT`, `REFLECTION_STATE_SCHEMA_VERSION = "v3_assistant_steps"`, `REFLECTION_SNAPSHOT_MAX_BYTES = 131_072` |
| **Locks** | 9개 도메인 크로스프로세스 락 | `src/locks/acquire.ts`, `domains.ts`, `candidate-sweep.ts`, `process-identity.ts`, `recall-wake-domain.ts` | `LOCK_DOMAINS` 9종, `CANDIDATE_STALE_AGE_MS = 1h` |
| **Seeds** | 초기 메모리 저장소 시드 | `src/seeds/seeds.ts`, `default-memory.ts`, `memory-discipline.ts` | `DEFAULT_MEMORY_BLOCK_LABELS = ["persona","human","boundaries","self-aware"]`, `MEMORY_DISCIPLINE_SKILL_PATH = "skills/memory-discipline/SKILL.md"` |

### Soul 워터마크 메커니즘

`soul/watermark.ts` 헤더가 절차 전체를 요약한다:

> identity 스코프 soul-notice 워터마크. identity당 워터마크 하나를 `runtime/notices/soul-head.json`에 두고 **sessionId는 절대 관여하지 않는다**. 워터마크가 없거나 읽히지 않으면 **HEAD에서 조용히 확립한다(히스토리를 재생하지 않는다)**. 이후 notice는 진전당 **정확히 한 번** 발생한다: delta는 `<watermark>..HEAD`에서 soul 경로를 건드린 **가장 최신 커밋** 중 **in-band로 작성되지 않은 것**(trailer `Omo-Writer: memory-tool` 없음)이고, 워터마크는 그 notice가 소비되는 **정확히 그때** HEAD로 전진한다. 읽기, delta 판정, 워터마크 교체를 모두 identity 스코프 `notice` 락 아래에서 직렬화하므로 **같은 identity에 묶인 동시 세션이 둘 다 공지하고 둘 다 전진할 수 없다.**

`SOUL_SCAN_PAGE = 32`가 스캔 페이지 크기. "스캔은 메모리 툴이 쓰지 않은 가장 최신 커밋을 원하므로, 그 페이지가 전부 in-band 쓰기인 경우에만 한 페이지로 부족하다."

### Facts 큐의 내구성

`facts/schema.ts` 헤더: "큐는 내구성 있고 버전이 있으며, **프로듀서를 한 번도 보지 못한 크래스 복구 프로세스**가 읽는다. 그래서 여기 있는 모든 형태는 읽을 때 fail-closed로 검증된다."

워터마크 2개가 의미가 다르다:
- `enqueued_through_message_id` — **단조 증가**, 절대 후퇴하지 않음
- `consumed_through_message_id` — 검증된 적용 이후에만 전진

여기서 가장 흥미로운 설계: message id는 불투명(opaque)이고, 늦게 끝난 구형 배치의 엔트리 리스트에는 **새 배치의 엔드포인트가 들어 있지 않으므로 두 엔드포인트를 위치 기반으로 정렬할 수 없다.** 대신 각 엔드포인트는 그것을 만든 배치의 `end_snapshot_line`(게시 시점의 저널 길이)과 함께 저장된다. 스냅샷 경계는 배치 간에 **비교 가능**하므로 두 워터마크 모두의 정렬 프레임이 된다.

`FactsQueue.enqueue`는 **유효 앵커**를 계산한다 — 영속 enqueue 워터마크, consumed 워터마크, 그리고 이 대화의 **모든 보존된 큐 파일** 중 가장 큰 canonical 위치의 최대값. 보존 파일까지 읽는 이유는 주석에 있다: "큐 파일은 landed했는데 커서 쓰기는 안 된 크래시 윈도우를 닫기 위해서."

`facts/failures-*` 3파일은 실패 관리다. `FACTS_FAILURE_REASONS` 7종, `FactsFailureState = "backoff" | "parked"`, `FactsFailuresCorruptError`(손상 시 조용히 덮어쓰지 않는다). `selectLaunchable`의 `FactsSkipReason = "parked" | "backoff" | "blocked-by-predecessor"`. `payload-cap.ts`의 `selectCappedFactsBatch`는 128 KiB 상한으로 배치를 자르되 `FACTS_STARVATION_MS`(24h) 초과면 굶주림을 경고한다.

`person-routing.ts`가 `planFactsRouting`으로 사실-사람 매칭을 한다. 주석이 렌더 경계에 증거를 인용한다: "규칙은 `DEFAULT_BOUNDARIES_BODY`에서 — 사용자가 말했을 때만 엔트리를 추가하고, 추론이나 자신의 거절에서는 절대 만들지 말 것." (`normalizeObservationText`, `FactsAliasTie`로 동률 처리.)

`recovery-mutation.ts`의 `FactsMutationTransaction` / `FactsOwnershipLostError`는 다른 프로세스가 파일을 집어갔을 때 커밋을 거부한다. `recovery-ownership.ts`의 `captureOwnedFactsState`/`sameFactsOwnedState`/`sameIdentity`가 소유권 스냅샷 비교다.

### Compile이 만드는 프롬프트 블록

`compile.ts`의 `compileMemoryBlockAtRevision(repo, revision, { agentId })`가 만드는 것:

```xml
<reminder 문구>

<self>
<projection>$MEMORY_DIR/system/persona.md</projection>
<persona 본문>
<projection>$MEMORY_DIR/system/identity.md</projection>
<identity 본문>
</self>

<memory>
  <contacts>
    <projection>$MEMORY_DIR/system/contacts.md</projection>
    <description>...</description>
    <본문>
  </contacts>
  <external_projection>
$MEMORY_DIR/: a.md, b.md
people/: alice.md
  </external_projection>
</memory>

<memory_metadata>
- AGENT_ID: <id>
</memory_metadata>
```

persona/identity는 `<self>`로 본문을 통째로 싣고, 나머지 `system/*.md`는 디렉터리 트리로 중첩 렌더(`renderSystemTree`)되어 **설명과 본문만** 싣는다. `system/`·`skills/` 바깥의 모든 파일은 경로 **목록만** 싣는다(`renderExternalProjection`) — 즉 커밋된 메모리의 **인덱스**가 프롬프트에 들어가고 본문은 경로로만 참조된다.

REMINDER 문구는 설계 결정을 그대로 말한다:

> `<projection>`은 메모리 투영의 로컬 경로를 담고 있다. `<memory>`는 대화 간에 지속되는 당신의 기억이다. **그것이 이미 답할 수 있는 것을 사용자에게 묻기 전에 참조하라.** 사실·선호·결정·정정이 떠오르는 **즉각** 메모리 툴로 저장하라. 사람에 관한 사실은 `people/` 아래 그 레코드로 라우팅하라(주 인간의 카드는 `system/human.md`). 관련 저장 메모리는 `<recalled-memory>` 블록으로 스스로 도착한다. **호출할 recall 툴은 없다.**

`render.ts`의 `markMemoryBlock`이 센티넬 `<!-- senpi-memory:<identity>:begin -->` … `:end -->`로 감싸고, `replaceMemoryBlock`은 동일 identity 블록만 치환하고 없으면 뒤에 붙인다, `stripMemoryBlock`은 제거한다. `changes.ts`의 `projectedChangesBetween(repo, from, to)`가 두 revision 차이를 계산해 블록 재생성을 최소화한다.

### Sync의 "단방향성"

`sync/mirror.ts` 헤더:

> 메모리 저장소의 **push 전용 미러**(letta `memory-git.ts` parity). 미러는 사용자 소유 백업 remote이며 **절대 source of truth가 아니다: 이 모듈 어디에도 pull·fetch·import 경로가 의도적으로 없다.** 히스토리는 한 방향으로 흐르고, push를 거부한 미러는 **보고 가능한 상태이지 로컬 메모리를 재작성할 이유가 아니다.**

`redact.ts`의 `containsSecretLikeMaterial` / `redactUrl`은 미러 로그와 nudge 게이트 양쪽에서 쓰인다 — 비밀값이 콜백 경로로 새는 두 지점을 막는다.

### Locks

`src/locks/domains.ts`의 9개 도메인: `memory-write`, `reflection-scheduler`, `reflection-finalize`, `transcript-state`, `skills-usage`, `memory-usage`, `facts-queue`, `facts-runs`, `notice`. 각 도메인은 고유 락 파일 경로 빌더를 가진다(`runFinalizationLockPath`는 run id가 `.`/`..`/빈 값이면 `throw`).

`acquire.ts`의 핵심은 `isLockOwnerProvenDead(owner)`다. 단순히 `kill(pid, 0)`만 하면 **PID 재사용**으로 다른 프로세스를 죽은 소유자로 오판해 락을 빼앗는다. 그래서 `getPidLiveness(pid): "alive" | "dead" | "unknown"`과 `getProcessStartIdentity`를 **둘 다** 대조하고, `startIdentitiesConflict` / `startIdentitiesComparable`로 판정한다. `process-start-time.ts`가 플랫폼별로 `readDarwinProcessStartSeconds`(macOS)와 `readWin32ProcessCreationFiletime`(Windows, bigint)을 읽는다 — Linux는 comparable 밖으로 처리되고 `unknown`이 된다.

`recall-wake-domain.ts`는 `RecallWakeLease`를 티켓 디렉터리 기반으로 발급하고, 슬롯이 다 차면 `RecallWakeBusyError`를 던진다(버리면 조용히 증발하지 않고).

### Journal

`entries.ts`가 프로젝션 계층이다. 도구 인자 300자, 도구 결과 4096자로 자르고(`TOOL_ARGS_TRUNCATE_LIMIT`, `TOOL_RESULT_TRUNCATE_LIMIT`), reasoning은 `REDACTED_REASONING_TEXT`("[REDACTED REASONING]")로 대체한다. `cursor.ts`가 `deriveState` / `captureCursorSnapshot` / `countCompletedSteps` / `finalizeCursor` / `reflectedThroughByteOffset`를 제공하고, 스키마 버전은 `"v3_assistant_steps"`. `store.ts`의 `TranscriptJournal`에 `withLocalJournalLock`(별도 저널 락, `JournalLockTimeoutError`)가 붙고 `fsync.ts`가 파일과 디렉터리 둘 다 fsync한다 — **리눅스에서 rename 후 부모 디렉터리 fsync 없이는 커밋이 사라질 수 있다.**

### Seeds

`buildDefaultSeedFiles()`는 순수 함수(파일시스템 접근 없음)이며 6개 파일을 만든다: `system/persona.md`, `system/human.md`(frontmatter에 `kind: "person"`, `aliases` 포함), `system/boundaries.md`, `system/self-aware.md`, `reference/self/observations.md`, 그리고 `skills/memory-discipline/SKILL.md`. `DEFAULT_PERSONA_BODY`의 첫 줄이 이 메모리 시스템의 정체성이다:

> "You are a coding agent with a persistent self. This file is that self."

`V1_PERSONA_SEED_SHA256` 상수로 v1 시드 본문을 고정해 마이그레이션 시 비교한다. `initMemoryWithSeeds`는 `GitMemoryRepo.init`에 위임하고, no-overwrite 가드(기존 HEAD가 있으면 즉시 반환)는 **repo 안에** 있다 — 이미 초기화된 저장소에 다시 부르는 건 안전한 no-op.

---

## 6. Compaction 방어 — memory-core 밖의 관계

Compaction 방어는 `packages/omo-opencode/src/`에 있고, memory-core의 상태 머신과 **이벤트로 이어진다**. memory-core는 저널과 커서만 소유하고, 훅 실행은 어댑터의 몫이다.

| 방어층 | 위치 | 역할 |
|---|---|---|
| **compaction-context-injector** | `packages/omo-opencode/src/hooks/compaction-context-injector/` (`hook.ts`, `recovery.ts`, `tail-monitor.ts`, `session-prompt-config-resolver.ts`, `validated-model.ts`, `tail.ts`) | 압축 후 컨텍스트를 복원. `recovery.ts`가 크래시 복구, `tail-monitor.ts`가 꼬리 감시 |
| **compaction-todo-preserver** | `packages/omo-opencode/src/hooks/compaction-todo-preserver/` (`hook.ts`, `index.ts`) | 압축 전 todo 목록 보존 |
| **preemptive-compaction** | `packages/omo-opencode/src/hooks/preemptive-compaction.ts` + `-trigger.ts`, `-no-text-tail.ts`, `-degradation-monitor.ts` | 넘치기 **전에** 압축. `-degradation-monitor.ts`는 압축 품질 저하를 감시 |
| 플러그인 접선 | `packages/omo-opencode/src/plugin/session-compacting.ts`, `src/shared/compaction-marker.ts` | 압축 이벤트와 마커 |
| 반사 훅 | `packages/omo-opencode/src/hooks/claude-code-hooks/pre-compact.ts` / `pre-compact-handler.ts` | pre-compact에서 반사 예약 |

**memory-core와의 접합은 하나뿐, 이벤트다.** `ReflectionEvent`의 `{ kind: "compaction_accepted" }`가 `journal.state.pending_compaction = true`를 세팅하고, 그 다음 성공한 `settled` 이벤트에서 `evaluateTransitions`가 `trigger: "compaction"` 반사를 예약한다. 성공 시 `completeTransition`이 해당 대화의 `pending_compaction`을 지운다.

두 번째 접합은 nudge 철회다. `PendingNudges.delete(sessionId)`의 주석: "**compaction은 그 세션의 파일을 수용되는 바로 그 순간에 철회한다.**" — 압축이 일어나면 대기 중인 nudge는 무의미해지므로(압축된 컨텍스트에 그 힌트를 참조할 근거가 없으므로) 주입되지 않는다.

Compaction 방어 전체가 memory-core를 우회하지 않는다: (a) 주입된 컨텍스트에 `<memory>` 블록이 있고, (b) 보존된 todo가 다음 턴을 이어가고, (c) 압축 사건이 Reflection을 깨우고, (d) 새 커밋이 다음 프롬프트의 `<memory>` 블록에 반영된다. 4단 중 하나라도 끊기면 메모리는 압축을 통과하지 못한다.

---

## 7. 세션 툴

`session_list` / `session_read` / `session_search` / `session_info`는 **memory-core가 아니다.** `packages/omo-opencode/src/tools/session-manager/tools.ts`에 있고 (`createSessionManagerTools()`가 4개를 반환), 저장 계층이 3단으로 갈려 있다.

| 툴 | 읽는 것 | 관련 파일 |
|---|---|---|
| `session_list` | 세션 목록 + 메타데이터, cwd 디렉터리 필터 | `session-manager/storage.ts`, `sdk-storage.ts`, `file-storage.ts`, `directory-filter.ts` |
| `session_read` | 세션의 메시지 전체 | `session-formatter.ts` |
| `session_search` | 세션 전체 풀스캔 검색 (FTS-lite) | `search/engine.ts`의 `searchTranscripts`를 프로바이더로 |
| `session_info` | 단일 세션 상세 통계 | `session-manager/types.ts`, `utils.ts` |

**메모리 툴과의 차이 — 이게 핵심이다**

| | `memory` / `memory_apply_patch` | `session_*` |
|---|---|---|
| 대상 | **메모리 저장소**(git, 커밋됨) | **하네스 세션 트랜스크립트**(harness 소유, 영속 아님) |
| 쓰기 | 가능 — 커밋되고 이력에 남음 | 불가 — 읽기 전용 |
| git | 메모리 커밋 + 선택적 미러 push | 무관 |
| 격리 | identity 스코프(`~/.omo/memory/agents/<id>/`) | 하네스 네이티브 스토리지(SDK → 파일 폴백) |
| 신선도 | HEAD 기준, 미커밋은 보이지 않음 | 실시간 |
| 목적 | "다음 세션에도 남을 지식" | "**지금 숨겨진** 세션 탐색" |

`session_*`의 존재 이유는 [`topics/memory.md`](../../topics/memory.md)에 적혀 있는 "OpenCode가 숨기는 세션 탐색"이다. model이 자기 과거를 되돌아볼 수 있게 해주는 것이지, 그 자체가 장기 기억은 아니다. 두 시스템은 완전히 분리되어 있고 `session-tools-store.ts`가 스토어 수명을 관리한다.

`src/search/engine.ts`가 두 시스템의 유일한 공유 코드다 — FTS-lite 스코어러(`search/query.ts`의 `parseQuery` / `matchScore` / `matchScoreNormalized` / `dateInRange`)를 memory-core의 Kibitzer recall이 substring 전략에서 재사용한다. `select.ts`가 이를 명시한다: "`matchScore`는 `description+body` haystack 위에 재사용된다(`SearchDocument` 투영은 recall 파일에 맞지 않으므로 haystack을 직접 조합한다)." 헤더도 letta 정밀도를 명시: `letta-code src/backend/local/transcript-search.ts:393-461` 미러링.

---

## 8. 다른 7개 하니스와 비교

[`topics/memory.md`](../../topics/memory.md) §요약 표를 재구성. OmO가 유난난 이유가 드러나도록 배치했다.

| 하니스 | 영구 메모리 | 리콜 주입 방식 | Compaction 방어 | 메모리 코퍼스 포맷 |
|---|---|---|---|---|
| **oh-my-openagent** | **git-backed MemFS** (identity별 별도 repo) | **Kibitzer nudge-only** — BM25 후보 → 작은 모델 judge → `<recalled-memory>` 참조 블록 1~5개 | 3중 (context injector / todo preserver / preemptive) + **compaction이 반사를 깨움** | YAML frontmatter 강제(.md), git 커밋 이력, `<memory>` 프롬프트 블록 |
| opencode | 없음 (서드파티 플러그인) | — | V1/V2 양쪽, prune 20k/40k | AGENTS.md/CLAUDE.md/CONTEXT.md walk-up |
| pi-mono | 없음 | — | contiguity-aware (tool call/result 사이 안 자름) | AGENTS.md > CLAUDE.md 조상 walk |
| oh-my-pi | 5 백엔드 (off/local/hindsight/mnemopi/sharpshooter) | 명시적 `recall` 툴 | 5 메서드 (remote/snapcompact/handoff/shake/soft) | `.omp/AGENTS.md` 최우선, 18개 discovery provider |
| codex | 2-phase 파이프라인 (기본 off) | 파이프라인 출력 | remote V2 + 로컬, checkpoint | AGENTS.md + AGENTS.override.md |
| claude-code | MEMORY.md 자동 메모리 (25KB/200줄) | 자동 주입 | /compact, /autocompact, 재귀 복구 | CLAUDE.md 계층 + `.claude/rules` |
| openclaw | **Dreaming**(기본 on) + 계층 메모리 | Dream 결과 주입 | compaction-notifier 훅 | AGENTS/SOUL/IDENTITY/USER/BOOTSTRAP/MEMORY.md |
| hermes-agent | frozen-snapshot MEMORY.md/USER.md + 5 provider | 프로바이더 스코어 | context_compressor (343KB) | SOUL.md + AGENTS.md 체인 + @참조 |

**OmO만 다른 세 축**

1. **메모리가 쓰기 가능한 산출물이다.** 다른 하니스의 메모리는 읽기 전용 컨텍스트 주입 파일이다. OmO의 `memory`는 git 커밋을 만든다 — 이력, 되돌리기, diff, 감사 로그가 전부 공짜로 따라온다. 그래서 `DirtyRepoError`(clean repo 강제)와 `NoEffectiveChangesError`(무효 커밋 금지)가 존재한다.
2. **리콜이 능동적이지 않다.** 6개 하니스는 "메모리가 컨텍스트에 들어온다"는 모델이다. OmO는 "메모리가 **당신에게** 먼저 다가온다"는 모델이다. 검색은 리콜 지향으로 넓게, 노출은 작은 모델 judge가 좁히고, 최종 산출물은 **참고 블록이며 지시가 아니다**(2인칭·명사·한국어 의/request 종결을 admission에서 거부). 모델이 "기억을 찾아야 한다"고 자각할 필요가 없다.
3. **자기 반사가 트랜잭션이다.** 다른 하니스의 통합은 프롬프트 요약이다. OmO의 반사는 worktree + 분기 + 통합 + orphan sweep + circuit breaker(3/6회, 6시간 probe)를 갖춘 배치 작업이고, 실패하면 커서를 전진시키지 않아 재시도된다.

**OmO의 대가**: 13,982 LoC(테스트 19,625 LoC 추가)에 9개 락 도메인, 크로스프로세스 소켓 자살 회수, worktree 잔여물 회수기가 붙는다. 다른 하니스들은 파일 몇 개로 같은surface를 낸다. 그 복잡도는 전부 **여러 세션이 하나의 identity를 공유할 때** 필요한 것이고, 단일 세션 단독 실행에서는 순수 비용이다.

---

## 9. 실전 가이드

### 설정해야 할 것

메모리는 `omo.json`의 `memory` 블록에서 설정한다(`packages/omo-config-core/src/schema/memory.ts`, per-agent 오버라이드는 `memory.agents.<name>`). 전부 strict 스키마이고 전부 기본값이 있다.

| 키 | 기본값 | 의미 |
|---|---|---|
| `reflection.enabled` | `true` | 반사 전체 on/off |
| `reflection.trigger.step_count` | `25` | N스텝마다 반사 |
| `reflection.trigger.on_compaction` | `true` | 압축을 반사 트리거로 |
| `reflection.merge` | `"auto"` | `auto` \| `integration` — worktree 결과 통합 방식 |
| `reflection.category` | `"quick"` | 반사 실행 카테고리(모델 등급) |
| `reflection.timeout_minutes` | `15` | 반사 실행 타임아웃 |
| `reflection.sandbox` | `"auto"` | `auto` \| `required` \| `off` |
| `recall.enabled` | `true` | Kibitzer on/off |
| `recall.max_items` | `2` | **1..5 hard 범위**. 한 턴이 실을 수 있는 nudge 개수 |
| `recall.category` | `"quick"` | 사이드카 모델 카테고리 |
| `recall.event_caps` | `tool_args:400, result_head:600, assistant:1500, prompt:4000` | 이벤트를 훑을 때 잘라읽는 상한 |
| `recall.sidecar_max_tokens` | `48000` | 사이드카 컨텍스트 예산 |
| `recall.max_concurrent_wakes` | `2` | 동시 기기 슬롯 |
| `recall.tool_budget` | `8` | 사이드카 툴 호출 예산 |
| `recall.query_expansion` | `false` | BM25 확장 term 활성화 |
| `nudge.enabled` / `nudge.every_user_turns` | `true` / `10` | 주기적 nudge |
| `facts.enabled` / `facts.debounce_settles` | `true` / `4` | 사실 추출 큐 |
| `dream.enabled` | `true` | 유휴 Dream |
| `dream.idle_minutes` | `30` | 유휴 판정 |
| `dream.min_hours_between` | `24` | Dream 최소 간격 |
| `dream.shutdown_launch` | `true` | 종료 시 Dream |
| `dream.auto_select_max` / `auto_select_max_chars` | `5` / `150000` | 자동 선택 상한 |
| `people.enabled` / `max_entries` / `max_entry_chars` | `true` / `40` / `200` | 인물 카드 한도 |
| `search.enabled` | `true` | 트랜스크립트 검색 |
| `sync.enabled` / `sync.remote` | `true` / 없음 | 미러 미러 |
| `soul.edit_notice` | `true` | soul 편집 통지 |
| `write_notice.enabled` | `true` | 툴 결과 행에 쓰기 통지 |

여기에 환경변수 3개가 얹힌다.
- `OMO_MEMORY_HOME` — 메모리 루트 오버라이드(기본 `~/.omo/memory`)
- `OMO_MEMORY_PUSH_SYNC=1` — post-commit push를 포그라운드로
- `memory.recall.max_items`는 **1 또는 5를 벗어나면 파싱 실패**한다 (`strict` 스키마 + 범위 검증)

### 실패 모드

| 증상 | 원인 | 조치 |
|---|---|---|
| `memory path: ... escapes memory directory` | 심볼릭 링크가 root 밖으로 나감 | 심링크를 실제 경로로 교체. 구버전 경로는 자동 변환 안 됨 |
| `memory path: 'file_path' must target a lowercase .md markdown file` | 확장자 지정 | 확장자 없이 `system/contacts` 형태로 |
| `... is read_only and cannot be modified` | `read_only: "true"` | frontmatter에서 제거 |
| `... is UTF-16 encoded` / `not valid UTF-8` | 인코딩 불일치 | UTF-8로 변환. **조용히 고쳐지지 않는다** |
| `'description' contains tool-call scaffolding` | 인자가 잘못 분할됨 | 한 줄 description + `file_text`로 재전송 |
| `frontmatter: target file is missing required frontmatter` | 수동 편집으로 헤더 파괴 | `normalizeMemoryFrontmatter`로 정규화 커밋 |
| `DirtyRepoError` | 워킹 트리에 미커밋 변경 | `git status`로 확인 후 커밋/stash |
| `made no effective changes` | 내용이 이미 디스크와 동일 | 실제 변경이 있는지 확인 |
| `LockContentionError` | 동시 쓰기 | 재시도. 지속되면 `runtime/locks/`의 후보 파일 확인 |
| `JournalLockTimeoutError` | 저널 락 정체 | `runtime/locks/` 확인 |
| `GitLockError` / config lock | git config 동시 쓰기 | `withGitLockRetry`가 자동 처리, 반복되면 git 프로세스 확인 |
| `ReflectionParkStateError` / park 발동 | 모델 없음·샌드박스 거부 등 | `runtime/reflection/park.json` 확인. 6시간 후 probe 1회 대기. **수동 해제 가능** |
| `FactsFailuresCorruptError` | 실패 파일 손상 | 자동 덮어쓰지 않는다. 수리 후 재생성 |
| nudge가 하나도 안 옴 | (a) judge가 전부 기각 (b) 후보 없음 (c) 이미 `surfaced` (d) `recall.max_items=1..5` | 원장과 pending 디렉터리 확인 |
| `<recalled-memory>`가 계속 옴 | pending 파일이 삭제 안 됨 | 24시간 TTL 후 자동 정리. 그 이전엔 `PendingNudges.take` 소비 여부 확인 |
| reflection이 계속 `parent_dirty` | 다른 프로세스가 worktree에 미커밋 변경 | worktree 잔여물 → orphan sweep 대상 |

### 상태 확인

**메모리 커밋 이력** — 그 자체가 상태 덤프다.
```bash
cd ~/.omo/memory/agents/<id>/repo
git log --oneline --stat -20        # 왜 이 메모리가 생겼는지 (reason 첫 줄)
git log --format='%H%n%B' -1        # 프로브런스 Omo-Writer / Omo-Session / Omo-Turn
git status                          # dirty면 툴이 전부 막힌다
```

**프롬프트에 실제로 들어가는 블록** — 커밋된 HEAD 기준으로 컴파일된다.
```bash
git ls-tree -r HEAD --name-only     # ← 이 목록이 <memory> 인덱스가 된다
```
미커밋 파일은 절대 이 목록에 없다(compile-from-committed 불변식).

**Kibitzer 상태**
```bash
ls ~/.omo/memory/agents/<id>/runtime/recall/ledger/   # 세션별 surfaced 원장
ls ~/.omo/memory/agents/<id>/runtime/recall/pending/  # 대기 nudge (24h TTL)
cat ~/.omo/memory/agents/<id>/runtime/recall/pending/*.json
# { version, sessionId, writtenAt, nudges: [{ path, hint }] }
```
nudge가 0이면 원인을 4단계로 좁힌다: `recall.enabled` → `max_items` → 세션 원장에 이미 있는지 → judge가 기각했는지(`pending`이 비었다면 기각).

**반사 상태**
```bash
cat ~/.omo/memory/agents/<id>/runtime/reflection/park.json   # streak / parkedAt / lastProbeAt
ls  ~/.omo/memory/agents/<id>/runtime/reflection/            # reservation 상태
ls  ~/.omo/memory/agents/<id>/runtime/worktrees/             # <epoch>-<runId> 잔여물
cd  ~/.omo/memory/agents/<id>/repo && git branch | grep reflection-   # memory/reflection-* 잔여 브랜치
```
`worktrees/`에 15분 넘은 디렉터리나 미병합 `memory/reflection-*` 브랜치가 있으면 orphan sweep이 아직 안 돌았거나 실패한 것이다.

**Facts 큐**
```bash
ls ~/.omo/memory/agents/<id>/runtime/facts-queue/       # 대기 배치
cat ~/.omo/memory/agents/<id>/runtime/facts-queue/consumed.json   # 소비 워터마크
cat ~/.omo/memory/agents/<id>/runtime/facts-queue/failures.json   # backoff/park 상태
ls ~/.omo/memory/agents/<id>/runtime/facts-queue/cursor/          # 대화별 enqueue 워터마크
```
`failures.json`에 `parked` 항목이 쌓이고, 큐 파일이 남아 있는데 `consumed_through`이 안 움직이면 실행 프로세스가 죽은 것이다.

**락**
```bash
ls -la ~/.omo/memory/agents/<id>/runtime/locks/
```
`*.lock` 외에 `*.tmp-*` 후보 파일이 있으면 획득이 끝나지 않은 채 죽은 프로세스가 남긴 것이다. 1시간(`CANDIDATE_STALE_AGE_MS`) 넘으면 sweep 대상.

**Soul 통지**
```bash
cat ~/.omo/memory/agents/<id>/runtime/notices/soul-head.json   # { version, lastNotifiedHead }
git -C ~/.omo/memory/agents/<id>/repo log --oneline <lastNotifiedHead>..HEAD -- system/persona.md system/identity.md system/boundaries.md
```
결과가 비어 있으면 통지할 변경이 없다. `Omo-Writer: memory-tool` 트레일러가 있는 커밋은 이미 통지된 것이라 delta에서 제외된다.

---

## 문서화 공백

위 9개 절의 모든 내용은 upstream 문서가 아니라 **소스에서 재구성한 것**이다.

`docs/` 전체를 grep한 결과 `memory-core`, `MemFS`, `Kibitzer`, `memfs`는 4개 파일에서만 나오고(`docs/guide/overview.md`, `docs/guide/orchestration.md`, `docs/reference/omo-json.md`, `docs/reference/configuration.md`), 거기에도 `memory.*` **설정 스키마**만 있다. `packages/memory-core`의 230개 파일 중 **단 하나도** 공식 문서에 등장하지 않는다. 모듈 경로, 상태 이름(`ReflectionTrigger` 4종, `DreamOrigin` 4종, `ReflectionOutcome` 7종), 락 도메인 9종, BM25 상수(`K1`/`B`), nudge 거절 사유 5종, park 임계값(3/6/6h), frontmatter 그라머, orphan sweep 소유권 정책 — 어느 것도 문서화된 적이 없다.

이 문서에서 인용한 값은 모두 해당 파일을 직접 읽고 확인했다. 다만 두 가지 한계를 명시한다. 첫째, **Harness 계층 배선은 소스 스캔으로만 확인했다** — Kibitzer 사이드카의 실행 주체, nudge 전달 타이밍, compaction 훅이 memory-core 이벤트를 올리는 지점은 `omo-opencode`/`omo-senpi` 쪽 파일을 끝까지 추적하지 않았다. 둘째, **설정 기본값은 `packages/omo-config-core/src/schema/memory.ts` 기준**이며 어댑터가 추가로 주입하는 기본값이 있으면 여기 반영되지 않았을 수 있다.