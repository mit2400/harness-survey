# 기본 스킬 비교

## 요약

| 하니스 | 내장 스킬 수 | 발견 경로 | 레지스트리/허브 |
|---|---|---|---|
| opencode | 1 (`customize-opencode`) | `{skill,skills}/**/SKILL.md`, `~/.claude`, `~/.agents`, config.skills.paths/urls | remote Discovery (index.json) |
| oh-my-openagent | 14 TS + 18 shared-skills | 7-source, SCOPE_PRIORITY | upstreams (designpowers 등) |
| pi-mono | **0** (의도적) | `~/.pi/agent/skills`, `.pi/skills`, `~/.agents/skills`, `.agents/skills` | 없음 |
| oh-my-pi | 0 (rules 26개 대신) | 9 provider, native 100 > ... > builtin-defaults 1 | skillshare, omp-plugins |
| codex | 5 | `.agents/skills`, `.codex/skills`, `~/.codex/skills/.system`, `/etc/codex/skills` | plugin roots |
| claude-code | 바이너리 내장 | `~/.claude/skills`, `.claude/skills`, plugin skills, nested | marketplace |
| openclaw | **48** | workspace/.agents/~/.agents/state-dir/bundled/custodian/extraDirs (7-tier) | **ClawHub** |
| hermes-agent | **58** + optional 152 | `~/.hermes/skills`, `.hermes/skills`, `.agents/skills` | **Skills Hub** (official/clawhub/GitHub/SkillSSH) |

## opencode
- 내장 1개: `customize-opencode` (`packages/core/src/plugin/skill/customize-opencode.md`) — 실제 config 스키마를 모델에 주입. 같은 이름의 유저 스킬이 덮어씀
- 발견: `packages/opencode/src/skill/index.ts` — `{skill,skills}/**/SKILL.md` (모든 config dir), `skills/**/SKILL.md` (`~/.claude`, `~/.agents`), `config.skills.paths[]`, `config.skills.urls[]`
- Remote: `discovery.ts` — `<url>/index.json` fetch → `Global.Path.cache/skills/<name>/`, `.opencode-version`, atomic swap
- Frontmatter: `name`(dir명과 일치, kebab-case), `description`(필수), license, compatibility, metadata
- 표시: `<available_skills>` XML을 skill 툴 description에 주입, permission으로 deny 시 섹션 숨김

## oh-my-openagent
- **TS 내장 14개** (`packages/skills-loader-core/src/features/builtin-skills/skills/`): playwright, playwright-cli, playwright-mcp-skill, frontend, git-master, dev-browser, review-work, remove-ai-slops, init-deep, debugging, security-research, security-review, visual-qa, team-mode
- **shared-skills 18개** (`packages/shared-skills/skills/`): ast-grep, browser, coding-agent-sessions, data-scientist, debugging, frontend, git-master, init-deep, lsp-setup, programming, refactor, remove-ai-slops, review-work, ultimate-browsing, ulw-execute, ulw-plan, ulw-research, visual-qa
- upstreams: designpowers, open-design, taste-skill, ui-ux-pro-max
- 발견 7-source, SCOPE_PRIORITY: opencode-project(6) > project(5) > opencode(4) > user(3) > config(2) > builtin=shared(1)
- 커스텀 위치: `.opencode/skills/`, `~/.config/opencode/skills/`, `.claude/skills/`, `.agents/skills/`
- `disabled_skills`로 비활성화

## pi-mono
- **내장 스킬 0개** — README가 명시: "Pi ships with powerful defaults but skips features like sub-agents and plan mode"
- 발견: `src/core/skills.ts` — `~/.pi/agent/skills/`, `<cwd>/.pi/skills/`, `~/.agents/skills/`, `.agents/skills/`(조상 walk, repo root까지), `--skill`/`settings.skills[]`
- Agent Skills spec 준수: `name`(≤64, kebab), `description`(≤1024), `disable-model-invocation`
- SKILL.md 있는 디렉토리가 스킬 루트 (재귀 X), bare `.md`도 로드, `.gitignore` 존중
- 프롬프트에는 name+description+`<location>`만, 본문은 read 툴로 on-demand
- `.agents/skills`는 trust gating (`trust-manager.ts`)

## oh-my-pi
- 내장 스킬 없음. 대신 **26개 내장 rules** (`packages/coding-agent/src/discovery/builtin-rules/`): go-* 9, rs-* 5, ts-* 12. priority 1 (`builtin-defaults`), 같은 이름의 유저/프로젝트 룰이 덮어씀. `ttsr.builtinRules: false`로 해제
- 스킬 발견 9 provider: native(100) > skillshare(95) > omp-plugins(90) > claude(80) > agent-plugins(75) > claude-plugins/agents/codex(70) > opencode(55) > github(30) > omp-managed(5, auto-learn)
- 레이아웃: `<skills-root>/<name>/SKILL.md` (non-recursive)
- Frontmatter: `name`, `description`, `globs`, `alwaysApply`, `hide`, `disableModelInvocation`, `enabled`
- 충돌: 동일 사본은 collapse, 다르면 `<namespace>/<name>`, `~2`, `~3`
- auto-learn: `~/.omp/agent/managed-skills/` (항상 authored보다 뒤)
- 호출: `/skill:<name>`, 내용은 `skill://<name>[/<path>]`

## codex
- **5개 번들** (`codex-rs/skills/src/assets/samples/`, `include_dir!`로 컴파일 타임 임베드): imagegen, openai-docs, review-agent, skill-creator, skill-installer
- 설치 위치: `$CODEX_HOME/skills/.system` (fingerprint marker로 바이너리 변경 시에만 재설치)
- 발견 루트: `<config_folder>/skills`(Repo), `$CODEX_HOME/skills`(deprecated), `~/.agents/skills`(User), `$CODEX_HOME/skills/.system`(System), `/etc/codex/skills`(Admin), plugin roots
- 메커니즘: frontmatter 파싱, `$skill` 명시 멘션, implicit invocation 감지, `@` 툴 멘션, snapshot 캐시
- 설정: `[skills] bundled.enabled`, `config`(이름/경로별 enable/disable/rename), catalog budget = 컨텍스트 윈도우의 2%

## claude-code
- 바이너리 내장 스킬 (`disableBundledSkills`/`CLAUDE_CODE_DISABLE_BUNDLED_SKILLS`로 해제)
- 발견: `~/.claude/skills/`, `.claude/skills/`, `--add-dir`의 `.claude/skills`, nested `.claude/skills`(파일 접근 시 lazy, 이름 충돌은 `<dir>:<name>`), plugin `skills/*/SKILL.md` + plugin root `SKILL.md`
- **Hot reload**: `~/.claude/skills`/`.claude/skills` 변경 시 재시작 없이 반영, `SessionStart` 훅이 `reloadSkills: true` 반환 가능
- `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PLUGIN_ROOT}` 변수
- 번들 스킬 텍스트는 바이너리에 압축 저장 (~2MB 절약)

## openclaw
- **48개 번들** (`/home/minkoo/study/repos/openclaw/skills/<name>/SKILL.md`): 1password, apple-notes, apple-reminders, bear-notes, blogwatcher, blucli, camsnap, clawhub, coding-agent, control-ui, diagram-maker, eightctl, gemini, gh-issues, gifgrep, github, gog, goplaces, healthcheck, himalaya, mcporter, meme-maker, model-usage, nano-pdf, node-connect, node-inspect-debugger, notion, obsidian, openai-whisper, openai-whisper-api, openhue, oracle, ordercli, peekaboo, pyproject.toml, python-debugpy, sag, sherpa-onnx-tts, skill-creator, songsee, sonoscli, spike, spotify-player, summarize, things-mac, tmux, trello, visualize, weather, xurl
- Custodian 4개: add-model-provider, cloud-image-bake, configure-channel, diagnose-gateway
- **7-tier precedence**: workspace/skills > workspace/.agents/skills > ~/.agents/skills > state-dir/skills > state-dir/agents/<id>/workshop-skills > bundled+custodian > extraDirs+plugin
- Frontmatter: `metadata.openclaw.{always, skillKey, primaryEnv, emoji, homepage, os, requires{bins,anyBins,env,config}, install[]}`, install kinds `brew|node|go|uv|download`
- 레지스트리: **ClawHub** (install/upload/archive/provenance), `src/skills/lifecycle/clawhub*.ts`
- Workshop: 에이전트가 제안하는 스킬 (`src/skills/workshop/`)

## hermes-agent
- **58개 번들** (`/home/minkoo/study/repos/hermes-agent/skills/`, 14 카테고리): apple(4), autonomous-ai-agents(5: claude-code/codex/computer-use/hermes-agent/opencode), creative(10), devops(1), email(2), media(3), note-taking(1), productivity(14), research(4), social-media(1), software-development(12), web(1)
- **Optional 152개** (`optional-skills/`, 22 카테고리) — `hermes skills install official/<category>/<skill>`, 기본 비활성
- 런타임 홈: `~/.hermes/skills/` (번들 복사 + hub 설치 + skill_manage 생성 모두 여기)
- Hub 소스: `official`, `clawhub`, GitHub, SkillSSH (+ `skills/index-cache/`에 anthropics/lobehub/openai 캐시)
- 모든 스킬이 슬래시 커맨드, 한 메시지에 최대 5개 스택
- 설정: `skills.external_dirs`, `create_dir`, `project_discovery`(trusted_project_dirs만), `auto_load`, `template_vars`(`${HERMES_SKILL_DIR}`), `inline_shell`
- **Blank-slate opt-out**: `.no-bundled-skills` 마커
- **Curator** (`agent/curator.py`): active→stale(14d)→archived(30d), 삭제 대신 아카이브, content-addressed ledger + single-edit rollback
- 툴: `skills_list`, `skill_view`, `skill_manage`
