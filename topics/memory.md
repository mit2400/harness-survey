# 메모리 관리 비교

## 요약

| 하니스 | 영구 메모리 | 세션 저장 | Compaction | 컨텍스트 파일 |
|---|---|---|---|---|
| opencode | 없음 (서드파티 플러그인만) | SQLite `opencode-<channel>.db` | V1/V2 양쪽, prune 20k/40k | AGENTS.md/CLAUDE.md/CONTEXT.md walk-up |
| oh-my-openagent | **git-backed MemFS + Kibitzer** | OpenCode 세션 상속 | 3중 방어(context injector/todo preserver/preemptive) | AGENTS.md walk-up + rules-engine |
| pi-mono | 없음 | JSONL 트리 `~/.pi/agent/sessions/` | contiguity-aware (tool call/result 사이 안 자름) | AGENTS.md > CLAUDE.md ancestor walk |
| oh-my-pi | **5 백엔드** (off/local/hindsight/mnemopi/sharpshooter) | JSONL `~/.omp/agent/sessions/` | **5 메서드** (remote/snapcompact/handoff/shake/soft) | 18개 discovery provider, `.omp/AGENTS.md` 최우선 |
| codex | **2-phase 파이프라인** (기본 off) | JSONL rollout + SQLite state DB | remote V2 + 로컬, checkpoint | AGENTS.md + AGENTS.override.md |
| claude-code | **MEMORY.md 자동 메모리** (25KB/200줄) | 세션 파일 | /compact, /autocompact, 재귀 복구 | CLAUDE.md 계층 + `.claude/rules` |
| openclaw | **Dreaming** (기본 on) + 계층 메모리 | SQLite per-agent | compaction-notifier 훅 | AGENTS/SOUL/IDENTITY/USER/BOOTSTRAP/MEMORY.md |
| hermes-agent | **frozen-snapshot** MEMORY.md/USER.md + 5 provider | SQLite state.db (13 테이블) | context_compressor (343KB) | SOUL.md + AGENTS.md 체인 + @참조 |

## opencode
- 전용 메모리 서브시스템 **없음**. `grep -r memory src/` 결과 프롬프트 1건뿐.
- 세션: SQLite (Drizzle), 테이블 `session/message/part/todo/session_input/session_context_epoch` (`packages/core/src/session/sql.ts`)
- Compaction: V1 `packages/opencode/src/session/compaction.ts` (PRUNE_MINIMUM=20_000, PRUNE_PROTECT=40_000), V2 `packages/core/src/session/compaction.ts`
- AGENTS.md: `packages/opencode/src/session/instruction.ts` — config-dir → `~/.claude/CLAUDE.md` → walk-up(AGENTS.md→CLAUDE.md→CONTEXT.md). read 대상 기준 상위 디렉토리 AGENTS.md를 메시지당 1회 자동 첨부
- System Context algebra: `packages/core/src/system-context/` (env, date 블록 diff 주입)

## oh-my-openagent (OmO)
가장 독보적인 메모리 시스템.
- **memory-core** (`packages/memory-core/`): git-backed markdown MemFS. YAML frontmatter 필수, 트랜잭션 쓰기(lock→clean repo→validate→commit)
  - 툴: `memory` (create/str_replace/insert/delete/rename), `memory_apply_patch`
  - **Kibitzer**: 작은 모델이 BM25 recall로 메인 에이전트 옆에서 넛지 (`recall/bm25.ts`, `recall/gate.ts` nudge-only 모드)
  - **Reflection**: 상태머신, worktree 실행, orphan sweep, park policy (반복 실패 시 자동 중단)
  - 서브엔진: Soul(identity watermark), Facts(durable queue), People(카드), Dream(idle 통합), Compile(커밋된 메모리→시스템 프롬프트 블록)
- Compaction 방어: `compaction-context-injector`, `compaction-todo-preserver`, `preemptive-compaction` 훅
- 세션 툴: `session_list/read/search/info` (OpenCode가 숨기는 세션 탐색)
- AGENTS.md: `agents-md-core` walk-up + `rules-engine` (`.omo/rules`, `.claude/rules`, `.cursor/rules`, `.github/instructions`)

## pi-mono
- 메모리 툴 없음. 세션 JSONL 트리 (`id`/`parentId`, v3), `/tree` `/fork` `/clone`
- Compaction: `contextTokens > contextWindow - reserveTokens` (reserve 16384, keepRecent 20000). tool call과 그 result 사이는 절대 안 자름. branch summarization, overflow 시 compact+retry 1회
- 컨텍스트 파일: agent dir → cwd 조상까지 각 디렉토리에서 `AGENTS.override.md > AGENTS.md > CLAUDE.md` first-match. git worktree 시 메인 repo 파일 억제

## oh-my-pi
- **5개 메모리 백엔드** (`memory.backend`, 기본 off): local(2-phase 파이프라인→MEMORY.md), hindsight(원격), mnemopi(SQLite), sharpshooter(결정 파일)
- 툴: `retain/recall/reflect` (+mnemopi는 `memory_edit`), `/memory view|stats|diagnose|queue|sync|clear`
- **Compaction 5 메서드** (`compaction.methodOrder` 기본 `["remote","snapcompact","handoff","shake","soft"]`):
  - remote: provider 네이티브 (OpenAI V2 streaming / V1 /responses/compact / Anthropic beta)
  - **snapcompact**: 히스토리를 PNG 프레임으로 아카이브 (모델별 프레임 크기, 로컬 전용)
  - handoff: 별도 요청으로 문서 생성→같은 세션에 CompactionEntry
  - shake: 기계적 elision (artifact:// 참조로 교체, LLM 호출 없음)
  - soft: 클래식 LLM 요약
- 트리거 6종: manual/overflow/incomplete-output/threshold/mid-turn/idle
- 컨텍스트 파일: 18개 discovery provider (native 100 > omp-plugins 90 > claude 80 > ... > builtin-defaults 1), `.omp/RULES.md`는 매 요청 sticky, `@` import 5-hop

## codex
- **Memories 기능** (`codex-rs/memories/`, 기본 off): 2-phase 비동기 파이프라인
  - Phase 1: 스레드별 rollout 추출 (병렬, leased job, secret redaction)
  - Phase 2: 전역 통합 → `~/.codex/memories/raw_memories.md`
  - 설정: `[memories]` 테이블 (max_raw_memories_for_consolidation=256, max_rollout_age_days=10 등)
- 세션: `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` + SQLite state DB (주 인덱스), zstd 압축
- Compaction: `core/src/compact.rs`, remote V2 (`compact_remote_v2.rs`), checkpoint (`history/src/compaction_checkpoint.rs`)
- AGENTS.md: `core/src/agents_md.rs` — project root(`.git`)→cwd walk, `AGENTS.override.md` 우선, `project_doc_max_bytes`, fallback filenames

## claude-code
- **MEMORY.md 자동 메모리**: 인덱스 파일, 25KB/200줄 제한 (초과 시 에러), `/memory` 다이얼로그로 on/off, `autoMemoryDirectory` 설정
- CLAUDE.md: enterprise/user/project/local 스코프, `@` import, HTML 주석 숨김, nested CLAUDE.md는 Read 시 lazy load
- `.claude/rules/*.md`: `paths:` glob frontmatter로 path-scoped 룰
- AGENTS.md: 내장 mod(`mods/agents-md/`)로 지원, 4가지 모드
- Compaction: `/compact`, `/autocompact <tokens>` (모델별 저장), 재귀 복구, Sonnet 5는 1M 윈도우에서 ~967K에 발동

## openclaw
- **Dreaming** (기본 on): light/deep/REM 3단계 백그라운드 통합, `DREAMS.md` + `memory/dreaming/<phase>/` 기록, MEMORY.md로만 승격
- 메모리 계층: Instructions(AGENTS.md) / Curated core(MEMORY.md, USER.md) / Episodic(memory/YYYY-MM-DD.md) / Prospective(standing intents) / Review(DREAMS.md)
- 엔진: per-agent SQLite, FTS5 BM25 + vector + hybrid, MMR, CJK trigram, sqlite-vec는 별도 read-only 프로세스
- **Active Memory**: 과거 질문 + deterministic recall 실패 시에만 blocking recall 모델 실행
- Provenance 기반 쓰기 게이팅 (사후 탐지가 아닌 구조적 차단)
- 세션: DMs 공유/그룹 격리, cron=fresh, webhook=isolated, `session.scope: "global"`
- 컨텍스트 파일: AGENTS.md, SOUL.md, IDENTITY.md, USER.md, BOOTSTRISE.md, MEMORY.md, TOOLS.md

## hermes-agent
- **Frozen-snapshot 메모리**: MEMORY.md(2200자)/USER.md(1375자), 세션 시작 시 시스템 프롬프트에 1회 렌더링 (prefix cache 보존), 쓰기는 즉시 디스크, 절대 auto-compact 안 함
- 툴: `memory` (add/replace/remove/batch), `write_approval` 옵션 (`/memory pending|approve`)
- 외부 provider 5개 내장: openviking, mem0, holographic, retaindb, byterover (+ catalog: honcho, hindsight, supermemory). 파이프라인: 주입→prefetch→sync→extract→mirror→provider 툴
- 세션: SQLite `~/.hermes/state.db`, 13 테이블, FTS5 + trigram (CJK), WAL, self-repair
- Compaction: `agent/context_compressor.py` (343KB), `parent_session_id`로 분할 추적
- 컨텍스트 파일: `.hermes.md > AGENTS.override.md > AGENTS.md > CLAUDE.md > .cursorrules` first-match, SOUL.md는 항상 별도 로드(prompt slot #1), AGENTS.md는 git-root→CWD 머지 체인, 서브디렉토리 AGENTS.md는 해당 파일 터치 시 점진 주입, 프롬프트 인젝션 스캐너 통과 필수
