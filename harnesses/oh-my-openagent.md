# oh-my-openagent (OmO)

> OmO: Just type "mass ulw" keyword with your prompt. (`code-yeongyu/oh-my-openagent`, 69k stars)
> omo-native는 이 레포 안의 어댑터 (`packages/omo-native/`)
> **분석 기준**: `code-yeongyu/oh-my-openagent` · 커밋 `251cbfe` (2026-10-04) · 버전 `5.1.13` — 전체 목록은 [VERSIONS.md](../VERSIONS.md)

## 개요
**50개 디렉터리** 모노레포(플랫폼 바이너리 12개 제외 시 38개). 구성: 공유 core 18개 + MCP 5개 + **어댑터 4개**(`omo-opencode`/`omo-codex`/`omo-senpi`/`omo-native`) + senpi-desktop 5개 + `senpi-task`·`boulder-state`·`rules-engine`·`lsp-daemon`·`shared-skills`·`utils`·`web`·`get-worker` + **플랫폼 바이너리 12개**(darwin/linux/windows × arm64/x64/x64-baseline/musl). Bun 1.4.0 + Rust crates(senpi-desktop computer-use).

## 디렉토리 구조
```
packages/
├── omo-opencode/      # OpenCode Ultimate 어댑터
├── omo-codex/         # Codex Light 어댑터
├── omo-senpi/         # Senpi Native 어댑터
├── omo-native/        # omo-native 런처
├── memory-core/       # 메모리 엔진
├── agents-md-core/    # AGENTS.md 주입
├── rules-engine/      # 룰 엔진
├── skills-loader-core/# 스킬 로더
├── shared-skills/     # 18 shared SKILL.md
├── omo-config-core/   # 통합 설정
├── boulder-state/     # boulder 상태머신
├── senpi-task/        # DAG 엔진
├── lsp-tools-mcp/     # LSP MCP
├── lsp-daemon/        # 공유 LSP 데몬
├── ast-grep-mcp/      # ast-grep MCP
├── git-bash-mcp/      # Windows git_bash
├── model-core/        # 모델 요구사항
├── prompts-core/      # 프롬프트 마크다운
├── team-core/         # 팀 모드
├── openclaw-core/     # OpenClaw 연동
└── utils/             # prompt-async-gate 등
```

## 메모리 관리
- **memory-core**: git-backed markdown MemFS, YAML frontmatter 필수, 트랜잭션 쓰기
  - 툴: `memory`(create/str_replace/insert/delete/rename/update_description), `memory_apply_patch`
  - **Kibitzer**: BM25 recall 사이드카 (`recall/bm25.ts`, `recall/gate.ts` nudge-only)
  - **Reflection**: 상태머신, worktree 실행, orphan sweep, park policy
  - 서브엔진: Soul/Facts/People/Dream/Compile/Search/Sync/Journal/Locks/Seeds
- Compaction 방어: compaction-context-injector, compaction-todo-preserver, preemptive-compaction
- 세션 툴: session_list/read/search/info
- AGENTS.md: agents-md-core walk-up + rules-engine (`.omo/rules`, `.claude/rules`, `.cursor/rules`, `.github/instructions`)

## 스킬
- TS 내장 14개: playwright, playwright-cli, playwright-mcp-skill, frontend, git-master, dev-browser, review-work, remove-ai-slops, init-deep, debugging, security-research, security-review, visual-qa, team-mode
- shared-skills 18개: ast-grep, browser, coding-agent-sessions, data-scientist, debugging, frontend, git-master, init-deep, lsp-setup, programming, refactor, remove-ai-slops, review-work, ultimate-browsing, ulw-execute, ulw-plan, ulw-research, visual-qa
- upstreams: designpowers, open-design, taste-skill, ui-ux-pro-max
- 발견 7-source, SCOPE_PRIORITY: opencode-project(6)>project(5)>opencode(4)>user(3)>config(2)>builtin(1)

## MCP
3-tier:
| Tier | 소스 | 로더 |
|---|---|---|
| 1. 내장 | `omo-opencode/src/mcp/` | createBuiltinMcps() |
| 2. Claude Code | `.mcp.json` | claude-code-mcp-loader |
| 3. 스킬 내장 | SKILL.md frontmatter | skill-mcp-manager (세션별, OAuth PKCE) |
Tier-1: websearch(Exa/Tavily), context7, grep_app, lsp(로컬 stdio)
추가: lsp-daemon(공유 LSP), git-bash-mcp(Windows), ast-grep-mcp(Senpi)
보안: Tier-3 `${sessionID}:${skillName}:${serverName}` 격리, mcp_env_allowlist는 user 레이어만

## 내장 툴
항상 12: grep, glob, session_list/read/search/info, background_output/cancel, call_omo_agent, task, skill, skill_mcp
조건부 +26: look_at, interactive_bash, monitor_*(4), task_*(4), edit(hashline), goal_*(3), team_*(12)
LSP는 내장 `lsp` MCP로: lsp_status/diagnostics/goto_definition/find_references/symbols/prepare_rename/rename/install_decision/format
ast-grep은 OpenCode에선 스킬(sg CLI), Senpi에선 MCP

## 에이전트
11개 (`agents/AGENTS.md`): Sisyphus(기본), Hephaestus, Prometheus, Atlas, Oracle, Librarian, Explore, Metis, Momus, Multimodal-Looker, Sisyphus-Junior
9 delegation 카테고리: visual-engineering, architect, ultrabrain, deep-low, deep-high, artistry, quick, unspecified-low, unspecified-high, writing
Team Mode (실험적, 기본 off): lead + 8 멤버, mailbox, 공유 task list, 멤버별 worktree, tmux

## 훅/플러그인
5-tier 54~62개 (`hooks/AGENTS.md`): Session(24)/Tool Guard(17~18)/Transform(4~7)/Continuation(7)/Skill(2)
주요: comment-checker, write-existing-file-guard, bash-file-read-guard, prometheus-md-only, rules-injector, keyword-detector(ulw), todo-continuation-enforcer(boulder), compaction-context-injector, preemptive-compaction, model-fallback vs runtime-fallback, session-notification
14 OpenCode hook handler 배선
Claude Code 호환 로더, Codex edition 10 컴포넌트, OpenClaw 연동
빌트인 커맨드 7: /goal, /refactor, /ulw-execute, /stop-continuation, /remove-ai-slops, /handoff, /hyperplan

## 설정
`~/.omo/omo.jsonc` (user) + `.omo/omo.jsonc` (project, cwd→$HOME walk)
Zod v4, snake_case, harness별 resolution(shared→[opencode]/[native]/[codex]→profiles)
프로필: OMO_PROFILE > OCX_PROFILE > OPENCODE_CONFIG_DIR tail
머지: deep-merge, scalar/array 교체, prototype pollution 방지
견고성: malformed 값 최소 subtree drop, invalid-value diagnostic, omo doctor
마이그레이션: lock+journal, `config migrate --dry-run`
상태: `~/.omo/agent`(canonical), teams/, rules/, plans/, goal/, lsp-daemon/

## 독특한 기능
- ulw/mass ulw 키워드 → DAG 실행 (`senpi-task/src/dag/`)
- Boulder 상태머신
- Kibitzer 메모리 사이드카
- prompt-async-gate (내부 session.prompt 유일 경로, 정적 audit)
- Hashline edit (LINE#ID, xxHash32)
- Goal (ralph-loop 대체)
- 2개 독립 fallback (model vs runtime)
- btw-side, Herdr DAG 시각화, omo thread SDK
- cmux 감지 (실제 tmux 없이 pane)

## 확장 포인트
- 툴: `packages/omo-opencode/src/tools/` + tool-registry.ts
- 에이전트: `agents/builtin-agents.ts` agentSources
- 훅: `hooks/` + create-*-hooks.ts tier composer
- 스킬: `shared-skills/skills/` 또는 `.opencode/skills/`
- MCP: `omo-opencode/src/mcp/` createBuiltinMcps()
- 설정 키: `omo-config-core/src/schema/`
