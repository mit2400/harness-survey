# oh-my-pi (omp)

> Coding agent with the IDE wired in. Built by Stencil Labs. (`can1357/oh-my-pi`, 34k stars)
> pi-mono의 fork/확장

## 개요
Bun+TypeScript 모노레포 + Rust crates(~80k LoC) + Bazel. 주 초점 `packages/coding-agent/`. docs/ 100+ 파일이 authoritative.

## 디렉토리 구조
```
packages/
├── coding-agent/      # CLI (메인)
├── agent/             # 에이전트 루프 + compaction
├── snapcompact/       # PNG 아카이브 compaction
├── catalog/           # 모델 카탈로그 (KDL 정책)
├── utils/             # dirs 등
└── ...
crates/                # Rust (ripgrep, glob, find, brush bash, coreutils)
docs/                  # 100+ 문서
```

## 메모리 관리
- **5 백엔드** (`memory.backend`, 기본 off): local(2-phase→MEMORY.md), hindsight(원격), mnemopi(SQLite), sharpshooter(결정 파일)
- 툴: retain/recall/reflect (+mnemopi memory_edit), `/memory view|stats|diagnose|queue|sync|clear|enqueue|rebuild|mm`
- 세션: JSONL `~/.omp/agent/sessions/<encoded-cwd>/<ts>_<id>.jsonl`, 256바이트 title 슬롯, tree/leaf, `/tree` `/resume` `/export` `/share` `/fork` `/fresh` `/branch` `/btw` `/handoff`
- **Compaction 6 트리거 + 5 메서드** (`compaction.methodOrder` 기본 `["remote","snapcompact","handoff","shake","soft"]`):
  - remote: provider 네이티브 (OpenAI V2/V1, Anthropic beta)
  - **snapcompact**: PNG 프레임 아카이브, 모델별 프레임 크기, 로컬 전용
  - handoff: 별도 요청 문서 생성→같은 세션 CompactionEntry
  - shake: 기계적 elision (artifact:// 참조)
  - soft: 클래식 LLM 요약
- 컨텍스트 파일: 18 discovery provider (native 100 > omp-plugins 90 > claude 80 > agent-plugins 75 > agents/claude-plugins/codex 70 > gemini 60 > opencode 55 > cursor/windsurf 50 > cline 40 > github 30 > vscode 20 > agents-md/claude-md 10 > mcp-json/ssh-json 5 > builtin-defaults 1). `.omp/AGENTS.md` 최우선, `.omp/RULES.md` sticky, `@` import 5-hop

## 스킬
- 내장 스킬 없음. **26 내장 rules** (`discovery/builtin-rules/`): go-* 9, rs-* 5, ts-* 12, priority 1
- 스킬 발견 9 provider: native(100) > skillshare(95) > omp-plugins(90) > claude(80) > agent-plugins(75) > claude-plugins/agents/codex(70) > opencode(55) > github(30) > omp-managed(5)
- 레이아웃: `<skills-root>/<name>/SKILL.md` (non-recursive)
- Frontmatter: name, description, globs, alwaysApply, hide, disableModelInvocation, enabled
- 충돌: 동일 collapse, 다르면 `<namespace>/<name>`, `~2`, `~3`
- auto-learn: `~/.omp/agent/managed-skills/`
- 호출: `/skill:<name>`, 내용 `skill://<name>[/<path>]`

## MCP
- 설정: `.omp/mcp.json`(project), `~/.omp/agent/mcp.json`(user), profile별
- **10 외부 포맷 네이티브 인식**: Claude Code, Codex, Gemini CLI, OpenCode, Cursor, Windsurf, VS Code, Claude marketplace, OMP extension, Agent Plugins
- Transport: stdio(기본)/http/sse(compat)
- 스키마: `{ $schema?, mcpServers, disabledServers?, enabledServers? }`
- OAuth: managed(`auth.credentialId`), explicit oauth block, auth-broker
- 시크릿: `${VAR:-default}`, `!shell-command`(10초, 캐시)
- 기본값: enableProjectConfig=true, startupTimeoutMs=250, notifications=false
- **브라우저 자동화 MCP 자동 드롭** (네이티브 브라우저 활성 시)
- 네이밍: `mcp__<server>_<tool>`

## 내장 툴
31개 (`tools/builtin-names.ts`): read, bash, edit, ast_grep, ast_edit, ask, debug, ida, eval, github, glob, grep, find, lsp, checkpoint, rewind, context_notes, new_context, security_scan, task, wait, todo, web_search, write, memory_edit, retain, recall, reflect, learn, manage_skill
Hidden 3: yield, goal, think
Essential 14: read, write, bash, edit, glob, find, eval, task, wait, learn, manage_skill, context_notes, new_context
3-tier: essential(top-level)/discoverable(`xd://`)/declared
기본 off: github, security_scan, generate_image, tts, checkpoint, rewind, memory 툴, debug, ida, glob, grep, web_search, todo, ask, AST 툴, lsp, goal
특이: **eval**(Python/JS 커널 툴 역호출), **debug**(DAP), **ida**(바이너리), **ast_edit**(preview→xd://resolve→accept), **github**(PR-as-paths), checkpoint↔rewind 페어링

## 에이전트
5개 내장 (`task/agents.ts` EMBEDDED_AGENT_DEFS): scout, reviewer, security-reviewer, task(`spawns:"*"`, model `@task`), sonic(model `@smol`, 기계적 업데이트)
Frontmatter: name, description, systemPrompt(필수), tools, spawns, model(우선순위), thinkingLevel, output, blocking, autoloadSkills, readSummarize, prewalk, advisor
발견: `~/.omp/agent/agents/*.md` + `.omp/agents/*.md` + extension packages + Claude marketplace, 내장은 마지막 append
**advisor**: 두 번째 모델이 매 턴 리뷰, 인라인 노트
**prewalk**: 자기 모델로 시작→첫 edit/write에서 smol 핸드오프
Agent Hub (Alt+A): 라이브 로스터, transcript, steer, revive
`vibe_spawn`: fast→sonic, good→task

## 훅/플러그인
- **Hooks = Extensions 통합**: `--hook`은 `--extension` alias
- 40+ 이벤트: session(session_start, session_before_compact→cancel/CompactionResult, session.compacting, session_compact, session_before_tree, session_tree, session_shutdown), agent(context, before_agent_start, agent_start/end, turn_start/end, auto_compaction_start/end, auto_retry_start/end, ttsr_triggered, todo_reminder), tool(tool_call→block/reason/input 교체/additionalContext, tool_result→필드별 머지)
- 확장 모듈: TS/JS, default export factory `(pi: ExtensionAPI) => void`
- 로드: native `.omp/extensions/`(cwd만) → `~/.omp/agent/extensions/` → JS/TS hooks `.omp/hooks/pre|post/` → plugin `omp.extensions` → `--extension`
- Marketplace: `.omp-plugin/marketplace.json` (또는 Claude 호환), `omp-plugins.lock.json`
- 제약: 로드 중 action 메서드 호출 시 ExtensionRuntimeNotInitializedError

## 설정
- `~/.omp/agent/config.yml` (YAML canonical), project `<cwd>/.omp/config.yml` + `.omp/settings.json`(legacy)
- **6-layer precedence**: env → runtime → overlays(`PI_CONFIG_FILES`, `--config`) → project → global → default
- Registry 패턴: `register({id, type, default, env?, protocolDefault?, validate?, pathScoped?, credential?, ui?})`, `provenance()`
- Config roots: `.omp` → `.claude` → `.codex` → `.gemini` (user→project)
- 프로필: `omp --profile <name>` / `OMP_PROFILE` / `PI_PROFILE` → `~/.omp/profiles/<name>/agent/`
- Path-scoped arrays: enabledModels, disabledProviders만
- 로드 실패: invalid YAML은 `.broken-<ts>-<pid>-<uuid>`로 이동 후 startup 실패
- XDG: XDG_DATA_HOME/STATE_HOME/CACHE_HOME
- CLI: `omp config set|get|reset|path|list|init-xdg`

## 독특한 기능
- Snapcompact (PNG 아카이브), TTSR (스트림 중간 규칙 주입), Hashline edit, 16 내부 URL 스킴(`conflict://`, `memory://` 등), eval 툴 역호출, 16 외부 config 포맷 네이티브, conflict resolution as URLs, speculative compaction, Advisor, ~80k LoC Rust 내장, 60+ provider/~1000 모델 KDL 정책, 4 엔트리포인트(TUI/print/RPC/ACP), Collab(`/collab` 릴레이+QR+클라이언트 암호화), Vibe mode, 모델별 프롬프트 튜닝, Bazel+Cargo+Bun 트리플 빌드, prompt-cache discipline, Magic keywords(ultrathink/orchestrate/workflowz)

## 확장 포인트
- 확장 작성: `docs/extensions.md` + `docs/skills/authoring-extensions.md`, 예제 `docs/skills/examples/`
- 훅 작성: `docs/skills/authoring-hooks.md`
- 모델/프로바이더 추가: `docs/adding-a-provider.md` + `packages/catalog/src/compat/rules/README.md` (KDL-first, `bun run gen:compat`)
- 런타임 부팅: `packages/coding-agent/src/main.ts`, `cli.ts`, `src/system-prompt.ts`, `src/tools/index.ts:840`
- 설정 해석: `docs/settings.md`, `packages/coding-agent/src/config/settings.ts`, `registry.ts`
