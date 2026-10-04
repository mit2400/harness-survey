# 파일 편집 메커니즘 비교

## 요약

| 하니스 | 주요 에디트 방식 | 검증 | undo | 특이사항 |
|---|---|---|---|---|
| opencode | `edit`(exact `oldString` + **9단계 fuzzy replacer**) / `apply_patch`(codiff, **GPT 전용**) / `write` | Levenshtein 0.65 앵커 매칭, 유니크 검사, 비례도 가드, V2는 conditional write | **git 기반 Snapshot** + session `revert`/`unrevert` | 모델별 edit-tool 게이트 (`registry.ts:296`), LSP diagnostics 후속 |
| oh-my-openagent | **hashline edit** (`LINE#ID`, xxHash32 2-char), 3-op (`replace`/`append`/`prepend`) | 라인 해시 재계산 일치 검증, bottom-up 정렬, dedup, `>>>` mismatch 컨텍스트 | opencode snapshot 상속 (plugin 별도 없음) | `hashline_edit: true` opt-in, Read 출력에 해시 태그 주입 hook |
| pi-mono | `edit` (`edits[]` exact string replace, 원본 기준) / `write` | NFKC·smart-quote·NBSP 정규화 fuzzy lookup, 유니크/겹침/no-op 검사 | **없음** | 실행 전 TUI live diff, 모델엔 `Successfully replaced N block(s)`만 전달 |
| oh-my-pi | `edit` **5 mode 교체** (replace/patch/apply_patch/hashline/sloppy) + `ast_edit` + `write` | **파일 해시 4-hex tag**, seen-lines guard, Levenshtein 0.95/0.92/0.8 사다리, no-op 3회 가드 | `checkpoint`/`rewind`는 **대화 전용** (파일/git 미포함) | mode별 lark grammar + `prompts/*.md`, notebook 텍스트 투영 편집 |
| codex | `apply_patch` **freeform 단일** (codiff, lark grammar custom tool) | `seek_sequence` 3-tier (exact→rstrip→trim), `@@` 앵커, EOF 앵커 | **없음** (TUI backtrack은 대화 탐색) | shell heredoc `apply_patch <<EOF`도 동일 파서로 흡수 |
| claude-code | `Edit`(`old_string` exact) / `Write` / `NotebookEdit`(`cell_id`) — **MultiEdit 제거** | 리터럴(내부 regex escape) 매칭 + 유니크, `"String not found in file"`, read-tracking/mtime | **`/rewind`(=`/undo`) + file-history 전체 백업** | `bashEditDiff`: Bash 실행의 파일 변경 diff를 결과에 부착 (모델非표시) |
| openclaw | `edit`(`edits[]` exact+fuzzy lookup) / `write` / `apply_patch`(codiff) | grapheme boundary로 원본 offset 복원, closest-match 3개 제안, **쓰기 후 byte 검증** | **없음** (memory 파일만 sha256 provenance + rollback) | edit planning을 Worker offload, diff receipt 1MiB cap |
| hermes-agent | `patch`(`mode=replace` 9전략 fuzzy / `mode=patch` V4A codiff) / `write_file` | 9전략 사다리, difflib 0.5/0.7, sha256 post-write, staleness blocker, 3회 실패 escalation | **shadow git checkpoint** (`~/.hermes/checkpoints/store/`, 모델 미노출) | 모델 계열별 `_EDIT_FORMAT_GUIDANCE` (gpt/codex→V4A, 그 외→replace) |

## opencode

- **edit-tool 모델 게이트 (edit modes)**: `packages/opencode/src/tool/registry.ts:296-300`
  ```ts
  const usePatch = input.modelID.includes("gpt-") && !includes("oss") && !includes("gpt-4")
  if (tool.id === ApplyPatchTool.id) return usePatch
  if (tool.id === EditTool.id || tool.id === WriteTool.id) return !usePatch
  ```
  GPT 계열엔 `apply_patch`만, 그 외에는 `edit`+`write`만 노출 (상호 배타적). 툴 설명은 `packages/opencode/src/tool/edit.txt`, `apply_patch.txt`.
- **edit 포맷**: `packages/opencode/src/tool/edit.ts:47-56` — `filePath/oldString/newString/replaceAll`. `oldString === newString` 및 빈 `oldString` 거부 (`:75-96`).
- **9단계 fuzzy fallback** (`edit.ts:694-704`), 헤더 주석이 cline/gemini-cli `editCorrector` 출처를 명시 (`:1-4`):
  `Simple → LineTrimmed → BlockAnchor → WhitespaceNormalized → IndentationFlexible → EscapeNormalized → TrimmedBoundary → ContextAware → MultiOccurrence`.
  `BlockAnchorReplacer`는 첫/마지막 줄 앵커 + Levenshtein 유사도 0.65 (`:220-221`), 라인 수 허용 오차 25% (`:303`).
- **가드**: `isDisproportionateMatch` (`:731-737`) — 매칭 span이 `oldString`의 3줄/4배를 넘으면 거부 ("Re-read the file..."). 유니크 실패 시 `"Found multiple matches for oldString"` (`:728`).
- **codiff**: `packages/opencode/src/tool/apply_patch.txt` + `packages/opencode/src/patch/index.ts` — `*** Begin Patch` / `*** Add File:` / `*** Delete File:` / `*** Update File:` / `*** Move to:` / `@@` 앵커. shell heredoc 형태(`apply_patch <<'EOF'`)도 `maybeParseApplyPatch`가 흡수 (`patch/index.ts:250-285`).
- **V2 (`packages/core/src/tool/edit.ts`)**: fuzzy 미포함(주석 L84 TODO). 대신 `FileMutation.writeIfUnchanged`로 permission 승인 사이 파일 변경 시 `StaleContentError` → "File changed after permission approval. Read it again before editing." (`edit.ts:113`).
- **write**: `packages/opencode/src/tool/write.ts` — 덮어쓰기 허용, `writeWithDirs`로 부모 디렉토리 생성, BOM 보존, `write.txt`에 "MUST Read first" 안내.
- **undo**: `packages/opencode/src/snapshot/index.ts` — 별도 `--git-dir`(프로젝트별 데이터 디렉토리)로 git snapshot. `session/revert.ts:70-73`이 `snap.track()`/`restore()`/`revert(patches)`로 메시지 단위 롤백 + `unrevert`.
- **대형 파일**: `packages/opencode/src/tool/truncate.ts:19-20` `MAX_LINES=2000`, `MAX_BYTES=50KB`, 7일 보존. `read.ts:14` 라인 2000자 잘림.
- **NotebookEdit 없음** (`.ipynb` 검색 결과 0건).

## oh-my-openagent

- **hashline edit**: `packages/omo-opencode/src/tools/hashline-edit/` (AGENTS.md가 전체 파이프라인 문서화). 등록은 `packages/omo-opencode/src/plugin/tool-registry-gated-tools.ts:32` — `hashline_edit` config가 켜져 있으면 **opencode 기본 `edit`을 대체**.
- **포맷**: `LINE#ID` = `{line_number}#{2 hex}`. 해시 알파벳 `ZPMQVRWSNKTXJBYH` (`packages/hashline-core/src/constants.ts`), `xxHash32(line, seed) % 256` → 2자 (`hash-computation.ts:5-9`). seed는 알파벳/숫자가 없으면 줄 번호 (`:6`).
- **3-op 모델**: `replace`(pos..end 범위) / `append` / `prepend`, `lines: null` = 삭제, `delete: true` = 파일 삭제. 스칼�/배열 `lines` 모두 허용 (`tools.ts:17-40`).
- **실행 파이프라인**: `normalize → validation(해시 일치) → edit-ordering(역순 정렬) → dedup → apply → autocorrect(들여쓰기 복원) → diff`.
- **검증**: `packages/hashline-core/src/validation.ts:73-80` — 현재 라인 해시를 재계산해 불일치 시 `HashlineMismatchError`. 에러 메시지에 `±2`줄 현재 내용 + 올바른 태그를 `>>>` 마커로 반환 (`MISMATCH_CONTEXT=2`, `:99`). `normalizeLineRef`가 `>>>`/`+`/`-` 접두사와 `|content`를 허용 (`:19-35`).
- **Read 주입**: `packages/omo-opencode/src/hooks/hashline-read-enhancer/hook.ts` — PostToolUse에서 `read` 출력의 `N: text` 행을 `N#XX|text`로 재작성 (`transformLine`). write 성공 시에도 스냅샷 헤더 부여.
- **실패 복구**: `packages/omo-opencode/src/hooks/edit-error-recovery/hook.ts` — `oldString not found` / `found multiple times` 등을 감지해 `[EDIT ERROR - IMMEDIATE ACTION REQUIRED]` system reminder를 tool output에 덧붙임.
- **기타 가드**: `hooks/write-existing-file-guard/` (기존 파일 write 차단), `hooks/atlas/write-edit-tool-policy.ts:1` `["Write","Edit","write","edit","hashline_edit"]` 편집 툴 집합.
- **undo**: 별도 없음. opencode의 git snapshot/revert를 그대로 상속.
- **AST**: `packages/ast-grep-mcp` + `ast-grep-sg-provision` hook — 탐색/검색용 스킬이지 편집 툴이 아님.

## pi-mono

- **툴**: `edit` / `write` 두 개뿐. `packages/coding-agent/src/core/tools/index.ts:95-105` `ToolName` 유니온. 기본 활성 `["read","bash","edit","write"]` (`system-prompt.ts:58`).
- **포맷**: `edit.ts:21-41` — `{ path, edits: [{ oldText, newText }] }`. **배치이지만 한 호출 내 `edits[]`**, 모든 항목이 *원본 파일* 기준으로 매칭 (`test/tools.test.ts:375`). 실행 시 매칭 인덱스 역순 적용으로 offset 유지 (`edit-diff.ts:111-120`).
- **검증** (`edit-diff.ts:300-362`): 빈 `oldText` 거부 → exact `indexOf` → NFKC/트레일링 공백/스마트 따옴표/대시/NBSP 정규화 매칭 (`:34-55`) → 유니크 카운트 → 겹침 검사 → no-op 검사. **전 검사가 단일 `writeFile` 전에 끝나므로 부분 적용 없음** (`test:417`).
- **에러 메시지**: `Could not find the exact text in <path>...`, `Found <n> occurrences... must be unique.`, `edits[i] and edits[j] overlap...` (`edit-diff.ts:253-289`).
- **retry 없음** — 모델이 판단. shape 복구만 존재: `prepareEditArguments` (`edit.ts:103-134`)가 문자열화된 `edits`, 단일 객체, 레거시 top-level `oldText`/`newText`를 복원 (주석이 Opus 4.6/GLM-5.1을 지목).
- **write**: `write.ts:52-53` — "Creates the file if it doesn't exist, overwrites if it does. Automatically creates parent directories." non-atomic (`fs.writeFile` 직접, temp+rename 없음).
- **동기화**: `file-mutation-queue.ts:32-61` realpath 단위 read-modify-write 직렬화.
- **undo 없음.** `checkpoint|rewind|revert` 검색 결과는 compaction/에디터 키바인딩뿐.
- **Notebook 없음** (`ipynb|notebook|nbformat` 0건).
- **모델 노출**: 툴 결과 `details.diff`(unified patch)는 **UI 전용** — 모델은 `Successfully replaced N block(s) in <path>.`만 받음 (`edit.ts:204-209`, `packages/agent/src/types.ts:427`).
- **프롬프트**: 툴 자체의 `promptSnippet`/`promptGuidelines`가 `<rules>`로 조립 (`edit.ts:43-51`): "Keep edits[].oldText as small as possible while still being unique".

## oh-my-pi

- **edit 하나가 5 mode**: `crates/pi-edit/src/engine.rs:15-21` `EditMode { Replace, Patch, ApplyPatch, Hashline, Sloppy }`. 등록은 `packages/coding-agent/src/tools/index.ts:563-594`, 클래스는 `packages/coding-agent/src/edit/index.ts` (`concurrency="exclusive"`, `strict=true`). mode가 `apply_patch`면 wire name도 `apply_patch`로 바뀜 (`:378-380`).
- **mode 선택**: `packages/coding-agent/src/utils/edit-mode.ts` — `edit.modelVariants`(모델 id 부분일치) → `PI_EDIT_VARIANT` → `edit.mode` → 기본 `hashline`. kimi/mimo/minimax/deepseek等은 `replace`로 폴백.
- **포맷** (`packages/coding-agent/src/edit/schemas.ts`):
  - `hashline`: `{ input: string }` — read 출력을 **제자리 편집**. `[PATH#TAG]` 헤더 + `PUT N.=M:` / `PUT N*:`(구문 블록) / `CUT` / `MV` / `REM`. 본문은 `+TEXT`(diff 아님).
  - `replace`: `{ path, old_string, new_string, replace_all? }`
  - `patch`: `{ path, edits: [{op?, rename?, diff?}] }` — `@@` 또는 `@@ $ANCHOR`
  - `apply_patch`: `{ input }` — `*** Begin Patch` codiff
  - `sloppy`: `{ input }` — `*** Find` / `*** Replace` / `*** Insert Before/After`, `…` 갭 캡처
- **파일 해시**: `crates/pi-edit/src/store.rs:80-97` — 줄별 trailing 공백 제거 후 **xxHash32 & 0xffff → 4 uppercase hex** (`[0-9A-F]{4}`). 스냅샷 LRU 256 path × 4버전, **4 MiB 초과 파일은 스냅샷 제외** (`MAX_SNAPSHOT_FILE_BYTES`, `:20`).
- **seen-lines guard**: `crates/pi-edit/src/modes/hashline/patcher.rs:123-186` — 앵커 라인이 read/grep으로 *보여준* 라인이 아니면 거부하되, 해당 구간의 실제 파일 내용을 인라인 노출하고 seen-lines에 합류시켜 **같은 요청 재시도가 바로 성공**하게 함 (`messages.rs:815-855`).
- **fuzzy 사다리**: `crates/pi-edit/src/fuzzy.rs:17-36` `0.95/0.92/0.8/0.8` 상수 + `Exact→TrimTrailing→Trim→CommentPrefix→Unicode→Prefix→Substring→Fuzzy→Character`. 다중 후보는 최고 ≥0.97이면서 2위와 0.08 이상 차이날 때만 채택.
- **stale-tag recovery**: `modes/hashline/recovery.rs:366` — 보존 스냅샷으로 앵커를 재베이스하고 **감싸는 구문을 내용 기준으로 재검증** (위치 검사만으론 형제 블록으로 어긋날 수 있음).
- **no-op 레프 가드**: `store.rs:21` `NOOP_HARD_LIMIT=3` — 3회 연속 바이트 동일 no-op이면 `STOP. Edits to {path} have been a byte-identical no-op...`.
- **post-write 검증**: `edit/index.ts:708-744` — 쓰기 후 재-read, 내용 불변이면 `edit appeared successful but file content did not change on disk` throw.
- **undo**: `docs/tools/checkpoint.md`가 명시 — *"does not call git and does not snapshot filesystem state"* / *"there is no file, artifact, blob, process, or git restore path"*. edit snapshot store는 태그/리베이스용으로만 쓰이고 공개 restore 경로 없음.
- **write**: `tools/write.ts` — 평문 파일 외에 `xd://<tool>`, `conflict://`, archive(`zip:inner/path`), `db.sqlite:table[:key]` 디스패치. **잘린 read 투영으로 덮어쓰기 거부** (`write.ts:278-291`).
- **Notebook**: `crates/pi-edit/src/notebook.rs` — `.ipynb`를 `# %% [code] cell:N` 텍스트로 투영해 edit이 라인 앵커를 걸 수 있게 하고, persist 시 JSON 재직렬화. 단 `write`는 이 codec을 통과하지 않음.
- **프롬프트**: `crates/pi-edit/prompts/{hashline,replace,patch,apply_patch,sloppy}.md`가 **tool description 자체**. 시스템 프롬프트 `prompts/system/system-prompt.md:105-109`는 `sed`/`perl`/`python` 개별 편집을 금지하고 `edit`를 강제.

## codex

- **단일 툴 `apply_patch`, freeform**: `codex-rs/core/src/tools/handlers/apply_patch_spec.rs:14-31`
  ```rust
  ToolSpec::Freeform(FreeformTool { name: "apply_patch",
    description: "The `apply_patch` tool can be used to edit files. This is a FREEFORM tool, so do not wrap the patch in JSON.",
    format: FreeformToolFormat { r#type: "grammar", syntax: "lark", definition } })
  ```
  문법은 `codex-rs/assets/tools/apply_patch.lark`. `protocol/openai_models.rs:323` `ApplyPatchToolType::Freeform`일 때만 `spec_plan.rs:1254`에서 등록.
- **포맷** (`codex-rs/apply-patch/src/parser.rs:7-18` 문법 주석): `*** Begin Patch` / `*** Add File:` / `*** Delete File:` / `*** Update File:` [`*** Move to:`] / `@@ [context]` / `*** End of File`.
- **검증 — `seek_sequence` 3-tier**: `codex-rs/apply-patch/src/seek_sequence.rs:1-60` — ① exact ② `trim_end` 무시 ③ `trim` 무시. `eof=true`면 파일 끝 우선 탐색 (`*** End of File`). 실패 시 `file_update.rs:211` `"Failed to find expected lines in {}:\n{}`.
- **heredoc 흡수**: `codex-rs/apply-patch/src/invocation.rs:120` — `apply_patch <<'EOF'` 형태의 `exec_command`를 파싱해 도구 호출로 승격. `compact_tests.rs:412`에 "exec_command로 apply_patch를 요청하면 apply_patch 툴을 쓰라"는 경고가 있음.
- **line-ending**: `CODEX_APPLY_PATCH_PRESERVE_LINE_ENDINGS_ENV_VAR` (`apply-patch/src/lib.rs:55`)로 CRLF 보존/정규화 모드 전환.
- **모델 프롬프트**: `codex-rs/core/gpt_5_1_prompt.md:287-319`가 `## apply_patch` 절 전체를 담음. `gpt_5_1_prompt.md:145` "NEVER try `applypatch` or `apply-patch`, only `apply_patch`", `:156` "Do not waste tokens by re-reading files after calling `apply_patch`... The tool call will fail if it didn't work.".
  `codex-rs/core/templates/model_instructions/gpt-5.2-codex_instructions_template.md:46`는 *"Try to use apply_patch for single file edits... Do not use apply_patch for auto-generated changes or when scripting is more efficient"* — apply_patch 실패 시 다른 방법도 허용.
- **undo 없음**: `codex-rs/tui/src/app_backtrack/`의 "rewind"는 대화 프롬프트 탐색. `history/src/compaction_checkpoint.rs`는 compaction 히스토리.
- **str_replace / NotebookEdit 없음.**

## claude-code

> 이 저장소는 `anthropics/claude-code` 공개 저장소로 **번들(cli.js)이 없다**. 근거는 `mods/types/claude-code.d.ts`(Claude Code 2.1.277 생성, 13,186줄)와 `CHANGELOG.md`(900KB).

- **툴 정의**: `mods/types/claude-code.d.ts:12427-12435`
  ```ts
  Edit: { file_path: string; old_string: string; new_string: string; replace_all?: boolean }
  Write: { file_path: string; content: string }            // :12590
  NotebookEdit: { notebook_path; cell_id?; new_source; cell_type?; edit_mode? }  // :12455
  ```
  `mods/diff/hooks/tools/editing-tools.ts:5` `EDITING_TOOLS = ['Edit','Write','NotebookEdit']`.
- **MultiEdit 제거**: `.d.ts`·`CHANGELOG` 0건. 플러그인 matcher에만 생존 (`plugins/security-guidance/hooks/hooks.json:33`). 옛 shape는 `plugins/security-guidance/hooks/security_reminder_hook.py:434-439`에서 복원 가능 (`edits: [{old_string,new_string}]`).
- **매칭**: 리터열 exact + 유니크. **fuzzy/공백 정규화/라인 앵커 없음**. 내부적으로 regex escape함이 `CHANGELOG.md:1041`의 `"Invalid regular expression: regular expression too large"` 버그와 `:1040`의 `\uXXXX` escape 버그로 증명됨. 에러 문자열은 `"String not found in file"` (`:1041`, `:1747`). 읽은 뒤 파일이 바뀌어도 *"still matches uniquely"*면 성공 (`:3265`).
- **bash-edit fallback 없음**: `apply_patch|bash-edit|reapply` 전역 0건. 오히려 반대 방향 — `CHANGELOG.md:6583` *"guide the model toward Read, Edit, Glob, Grep instead of bash equivalents (`cat`, `sed`, `grep`, `find`)"*.
- **Bash edit-diff**: `CHANGELOG.md:1506` — `bashEditDiffEnabled` 설정으로 Bash가 파일을 바꿀 때 결과에 diff 부착. `mods/types/claude-code.d.ts:12752-12776`의 `bashEditDiff`는 `@internal`·"not surfaced to the model", PostToolUse Bash hook용 (`changedFiles`, 최대 200개).
- ** freshness**: 내용 hash가 아니라 **read-tracking + mtime**. `d.ts:12727` `staleReadFileStateHint`, `d.ts:12996` Read 결과 `type: "file_unchanged"`.
- **undo / checkpoint**: `mods/diff/hooks/is-checkpointing/is-checkpointing.ts` — `settings.fileCheckpointingEnabled !== false && !CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING`. `/rewind`(=`/undo`, Esc-Esc)이 file-history **전체 내용 백업**으로 롤백 (`CHANGELOG.md:7462`, `:5174`, `:3287` "bounded checkpoint disk usage by pruning superseded file-history backups").
- **Write**: `d.ts:13153-13180` — `type: "create"|"update"`, `structuredPatch`, `originalFile`(신규/과대 파일은 null), `userModified`(퍼미션 다이얼로그에서 사용자가 편집), `staged`. `CHANGELOG.md:2775` *"newer models can overwrite an existing file they haven't read this session... older models still require the read first"*. `"File has not been read yet"` 문자열 실재 (`:3140`).
- **대형 파일**: Edit의 1 GiB 초과 OOM 수정 (`:5529`), diff는 **5초 타임아웃 후 우아한 폴백** (`:5671`), `truncatedByTokenCap` (`d.ts:12931`).
- **NotebookEdit**: 문자열 매칭이 아니라 `cell_id` + `edit_mode: replace|insert|delete`. 출력에 `original_file`/`updated_file`(전체 노트북)과 `old_source`로 셀 단위 diff 렌더링 (`d.ts:12844-12864`).

## openclaw

- **툴 3개**: `edit` (`src/agents/sessions/tools/edit.ts:311`), `write` (`write.ts:397`), `apply_patch` (`src/agents/apply-patch.ts:134`). `str_replace`/`old_string`/`multi_edit`/`notebook_edit` **없음**. 레지스트리 `src/agents/sessions/tools/index.ts:94` `{"read","bash","edit","write","grep","find","ls"}`, apply_patch은 별도 그룹 (`src/agents/core-coding-tools.ts:389-409`).
- **edit 포맷**: `tool-schemas.ts:10-29` `{ path, edits: [{oldText, newText}] }`. 레거시 `prepareEditArguments` (`edit.ts:84-126`)가 top-level `oldText`/`newText`·문자열화된 `edits`·모델 첨부 메타키(`reason`)를 정리.
- **fuzzy**: `edit-diff.ts:43` NFKC → 줄 `trimEnd` → 스마트 따옴표 → 대시 → 특수 공백. **매칭은 fuzzy로 찾되 스플라이스는 항상 원본 바이트** — grapheme 경계 맵(`buildFuzzyBoundaries`)으로 원본 offset 복원, 매핑 불가 시 fail closed (`getUnsafeFuzzyBoundaryError`).
- **에러**: closest-match 최대 3개를 Levenshtein 점수 + `^^^` caret으로 제시 (`edit-diff.ts:231-294`, `EDIT_CANDIDATE_MIN_SCORE=0.45`). 겹침 검사 (`:390-399`). mismatch 시 현재 파일 내용 최대 800자를 덧붙임 (`edit.ts:128-138`).
- **apply_patch 검증**: `@@` 컨텍스트 유니크 필수, 헤더 라인 유니크 필수, **4-tier 양보 매칭** (exact → `trimEnd` → `trim` → 문장부호 fold) `apply-patch-update.ts:296-318`. `*** Add File:`는 exclusive create(`wx`) — 존재하면 `Cannot create <path>: the file already exists.`
- **쓰기 후 byte 검증**: `file-write-verification.ts:12-30` — 길이 + `Buffer.equals` 비교. 실패 시 `Edit verification failed for <path>: the persisted regular file does not match...`.
- **undo 없음**: `undo|rewind|checkpoint|backup` 검색이 전부 무관 히트. `src/snapshot/git-backup.ts`는 **SQLite 에이전트 상태** 백업(수동 CLI)이지 편집 전 파일 백업이 아님. 유일한 롤백은 memory 파일 한정 `memory-artifact-provenance.ts:132-183` (sha256 + reservationId CAS + rollback closure, `MEMORY.md`/`memory/**`만).
- **write**: `write.ts:404-472` — `mkdir(recursive)` → 쓰기 → byte 검증. 내용 동일이면 `No changes made to <path>...` + `changed:false` (터미널 아님).
- **대형 파일**: `truncate.ts` 2000줄/50KB, edit planning은 `file-tool-planning.worker.ts` Worker offload, diff receipt는 1 MiB 초과 시 통째로 생략 (`WRITE_DIFF_MAX_BYTES`, `file-diff.ts:72-74`).
- **Notebook 없음** (`.ipynb`는 불투명 JSON). CLI backend 문서(`docs/plugins/cli-backend-plugins.md:384`)에 `NotebookEdit`은 타 하니스 툴명.
- **프롬프트**: `edit`의 `promptGuidelines` 4개 + `system-prompt-tool-list.ts:19` `apply_patch: "Patch files"`. **apply_patch에는 `promptGuidelines`가 없음** — `@@` 앵커/`Move to` 규칙이 툴 description에만 의존.

## hermes-agent

- **툴 2개**: `write_file` (`tools/file_tools.py:1391`) / `patch` (`:1414`). `str_replace`는 주석에만 (`agent/coding_context.py:63`), `apply_patch`는 V4A *mode*이자 Codex runtime의 네이티브 툴명 (`agent/codex_runtime.py:280`).
- **`patch mode=replace` (기본)**: `file_tools.py:1186-1229` — "Uses fuzzy matching (9 strategies)". `tools/fuzzy_match.py:289`
  ```python
  STRATEGIES = exact, line_trimmed, whitespace_normalized, indentation_flexible,
                escape_normalized, trimmed_boundary, unicode_normalized,
                block_anchor, context_aware
  SIMILARITY_STRATEGIES = {"block_anchor", "context_aware"}
  ```
  `block_anchor`는 difflib 0.50(후보 1)/0.70(다수), `context_aware`는 첫·마지막 줄 0.80 + 전 비공백 줄 0.80.
- **`patch mode=patch` (V4A)**: `tools/patch_parser.py` — `*** Begin Patch` / `*** Add|Update|Delete|Move File:` / `@@ hint @@`. **OpenAI 계열에만 schema 노출** (`_is_openai_family_main`, `file_tools.py:1258`) — *"advertising it to everyone cost every other session ~148 tok/call"* (`:1189`). 핸들러는 어떤 모델이든 두 형태 수용.
- **검증**:
  - 유니크: `fuzzy_match.py:369-382` `Found {N} matches for old_string...` + 최대 5개 위치.
  - V4A는 **dry-run 2단계** — 모든 op를 overlay에 시뮬레이션 후 일괄 적용 (`patch_parser.py:149`, `:291`), 실패 시 `"Patch validation failed (no files were modified):"`.
  - 쓰기 후 sha256 재검증 (`file_operations.py:1475`) + 재-read 정규화 비교 (`:1582` "The patch did not persist.") + lint delta.
  - staleness: 과거 기준선 없음/mtime 변동 시 **디스크에 손대기 전 거부** (`file_tools_write_guards.py:543`, 이슈 #65604).
  - **3회 실패 escalation** (`file_tools.py:1022-1041`): *"Stop retrying with variations of the same old_string... (3) use write_file to replace the entire file"*. 성공하면 카운터 리셋.
  - idempotency: `is_already_applied` (`fuzzy_match.py:307`)는 에러가 아니라 **성공형 no-op** ("do not re-send this patch").
  - escape drift 가드 3종 (`:416-520`) — non-exact 매칭 시 `new_string`의 이스케이프 표기 차이를 거부.
- **undo — shadow git checkpoint**: `tools/checkpoint_manager.py:10` *"This is NOT a tool — the LLM never sees it."* `~/.hermes/checkpoints/store/`에 `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE` 분리 스토어, 프로젝트별 ref로 객체 dedup.
  - 트리거: `agent/tool_executor.py:1023-1035` — 모든 `write_file`/`patch`/파괴적 `terminal` **직전** `_ensure_file_checkpoint`.
  - 복구: `restore()` (`:1116`)가 **agent-write ledger** (`sha256` 기록, `:853`)와 비교해 *사용자가 이후 직접 수정한 파일은 건너뜀* (`skipped_user_edits`).
  - undo-the-undo: `:1275` rollback 전 pre-rollback 스냅샷. 상한 `max_snapshots=20`, `max_total_size_mb=500`.
  - `/rollback` (`gateway/slash_commands.py:713`), `hermes checkpoints` CLI.
- **write_file**: `file_tools.py:1168-1184` — 완전 덮어쓰기, 부모 디렉토리 자동 생성, **기존 파일은 이 테스크의 전체 read/write 기록이 없으면 거부**, `verified:true`는 hash 확정을 뜻함. 원자적 쓰기는 stdin으로 전송해 ARG_MAX 제한 없음 (`file_operations.py:568`).
  - **비용 힌트**: 20k자 초과 파일에서 80% 이상이 미변경이면 `"Re-sending a 48,231-char file costs output tokens for every unchanged line; for edits like this use patch"` (`:827`, 근거로 1,393 에이전트 실행에서 661건·~$155 언급).
- **대형 파일/Notebook**: read 기본 100,000자, 잘리면 `truncated:true` + `next_offset` (`file_tools.py:52-95`). `.ipynb`는 **read-only 추출** (`tools/read_extract.py:30`, `:132`) — 쓰기는 전체 JSON `write_file`로만.
- **모델 프롬프트 (edit_formats)**: `agent/coding_context.py:65-81` `_EDIT_FORMAT_GUIDANCE` — 모델 id 부분일치로 한 줄을 교체:
  - `gpt`/`codex` → *"use `patch` with `mode='patch'` (V4A diff) — including single-file edits. It's the edit format you handle most reliably."*
  - `claude`/`sonnet`/`opus`/`gemini`/`qwen`/`glm`/... → *"prefer `patch` in `mode='replace'`... Reach for `mode='patch'` (V4A) only when an edit genuinely spans several files"*
  - 주석 `:61-64`: *"codex-rs ships apply_patch as its ONLY editor. Anthropic and most open-weight coders were RL'd on str_replace editors."*
  - `CODING_AGENT_GUIDANCE` (`:84-132`): *"If the same region fails twice, rewrite the enclosing function or file with `write_file` instead of attempting a third patch."*

## 관찰 / 시사점

1. **"exact match"는 서로 다른 의미를 가진다.** claude-code는 진짜 리터열(내부 regex escape)이며 fuzzy가 아예 없다. 반면 opencode·hermes·openclaw·pi-mono는 exact 1순위 이후 정규화/유사도 사다리를 두고, oh-my-pi는 fuzzy를 기본으로 켠다. 즉 같은 `old_string` 계열 스키마라도 하니스마다 성공률이 크게 갈린다.
2. **전략 사다리의 계보가 공유된다.** opencode `edit.ts:1-4`는 cline/gemini-cli `editCorrector`를, hermes `fuzzy_match.py`는 `line_trimmed/whitespace_normalized/indentation_flexible/escape_normalized/trimmed_boundary/block_anchor/context_aware`로 거의 같은 이름·순서를 쓴다. 두 코드베이스는 사실상 같은 3세대 (Gemini CLI → Cline) 복구 전략을 각자 포팅한 상태다.
3. **stale-read 문제 세 가지 해법**: (a) *내용 hash* — oh-my-openagent/oh-my-pi는 라인·파일 해시 태그로 읽은 시점과 편집 시점을 결합시켜 원천 차단, (b) *conditional write* — opencode V2 `writeIfUnchanged`는 permission 승인 사이의 변경을 CAS로 거부, (c) *read-state + mtime* — claude-code/hermes는 기록 기반 staleness blocker. hash 방식이 가장 강하지만 Read 출력에 태그를 심어야 하므로 토큰 비용과 툴 결합도가 든다.
4. **undo는 3개 하니스만 실제 파일 롤백을 제공한다.** claude-code(`file-history` 전체 백업 + `/rewind`), hermes(모델 미노출 shadow git), opencode(별도 git-dir snapshot + session revert). 나머지는 "실패 시 부분 적용 없음"과 "diff receipt 보관"이 사실상의 안전망이며, oh-my-pi는 checkpoint/rewind가 *문구상* git인데 실제로는 대화 전용이라 문서로 선을 긋고 있다.
5. **codiff(`*** Begin Patch`)가 사실상의 표준이 되었다.** codex 원본 → opencode(`gpt-*` 전용), openclaw, oh-my-pi(`apply_patch` mode), hermes(V4A)까지 4곳이 같은 봉투 형식을 구현한다. 모두 `@@` 앵커 + `+/-/space` 본문에서 출발하지만, opencode는 `trim`까지, codex는 `seek_sequence` 3-tier, openclaw는 4-tier로 *양보 단계가 서로 다르다* — 즉 동일한 패치라도 하니스를 옮기면 성공 여부가 달라진다.
6. **AST 편집은 전부 "탐색/프리뷰"로 수렴한다.** pi-mono는 AST 툴 자체가 없고, oh-my-openagent는 ast-grep을 스킬로, oh-my-pi만 `ast_edit`를 가지되 **프리뷰 → `xd://resolve` → accept**의 2단계로 실제 쓰기는 별도 승인에 맡긴다. 즉 "AST가 곧바로 쓰기"인 하니스는 없다.
7. **NotebookEdit은 claude-code만 가지고 있다** (cell_id 기반, `edit_mode: replace|insert|delete`). oh-my-pi는 대안으로 `.ipynb`를 `# %% [code] cell:N` 텍스트로 투영해 *일반 edit으로* 편집 가능하게 만들었고, 나머지는 `.ipynb`를 불투명 JSON으로 다룬다.
8. **모델 프롬프트 수준의 edit-format steering이 갈리는 지점**: hermes는 모델 계열별 한 줄을 교체해 V4A/replace를 갈라주고, oh-my-pi는 mode 자체를 모델별로 바꾸며, opencode는 툴 *집합*을 바꾼다(gpt면 apply_patch, 아니면 edit). 반면 claude-code는 도구를 고정하고 *"sed 말고 Edit를 써라"*는 방향 유도만 한다 — 과거 모델에 최적화된 포맷을 강제하지 않는 쪽과, 모델별 최적 포맷을 노출하는 쪽의 설계 갈림이다.
