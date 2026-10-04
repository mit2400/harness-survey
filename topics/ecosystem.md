# 스킬/MCP 생태계 서베이

## 규모 기준

| Repo | 규모 |
|---|---|
| VoltAgent/awesome-openclaw-skills | ~5,150 스킬 (long-tail, ClawHub 스크래핑) |
| VoltAgent/awesome-agent-skills | ~1,124 엔트리, 70+ 벤더 (curated) |
| hashgraph-online/awesome-ai-plugins | 640 플러그인 (Codex-first) |
| awesome-opencode/awesome-opencode | 223 레지스트리 항목 (136 플러그인/64 프로젝트/9 에이전트/7 테마/7 리소스) |
| qualisero/awesome-pi-agent | **은퇴** — CHANGELOG가 pi-mono 생태계의 최선 기록 |

## 가장 인기 있는 카테고리 (순위)

1. **코드 인텔리전스** — LSP + AST-grep + code-graph + RAG. 모든 하니스가 자체 구현 (oh-my-claudecode LSP/AST Grep 25개 언어, oh-my-opencode-slim AST 25개 언어, senpi는 code-yeongyu/pi-lsp-client + pi-ast-grep). awesome-opencode 최고: **xberg-io/plugins** (300+ 언어, HTML→MD, 크롤링, 143 LLM provider, 97 포맷 문서 추출)
2. **MCP 서버** — awesome-ai-plugins 640개 중 218개가 MCP. 주요 도메인: memory/persistence, observability(langfuse/sentry/grafana/otel), browser(playwright/chrome-devtools), DB(postgres/sqlite/duckdb), devops(kubernetes), niche(n8n, Burp Suite, Asterisk/FreeSWITCH, GSC/GA4)
3. **메모리/컨텍스트 엔지니어링** — awesome-opencode 플러그인 1위 테마 (opencode-agent-memory, harness-memory, magic-context, dynamic-context-pruning, claude-memory, honcho, mnemoria 등 14+). 스킬 쪽: **muratcankoylan/Agent-Skills-for-Context-Engineering** (8개: context-fundamentals/degradation/compression/optimization/memory-systems/tool-design/evaluation/multi-agent-patterns), **NeoLabHQ/context-engineering-kit**
4. **멀티에이전트 오케스트레이션** — awesome-opencode에 ~15개 (CrewBee, opencode-swarm, opencode-ensemble, opencode-workspace, mission-control, FlowDeck, open-dynamic-workflows, ralph-wiggum, subtask2). 하니스 수렴: Team mode+tmux workers, Ralph/ultragoal durable loops, verify→fix evidence gating, Council/multi-model debate
5. **검증/코드리뷰/TDD** — 표준 팩: **obra/superpowers** (16개: test-driven-development, systematic-debugging, requesting/receiving-code-review, verification-before-completion, using-git-worktrees, subagent-driven-development, brainstorming, writing-plans, writing-skills). 가장 많이 cross-reference되는 커뮤니티 스킬 프레임워크
6. **브라우저 자동화/웹 스크래핑** — awesome-openclaw-skills browser-and-automation 311, search-and-research 343. Playwright/Puppeteer/Chrome-MCP + scraping-as-a-business (AnyCrawl, Apify, Firecrawl, Crawlbase, Scrapling)
7. **프론트엔드/UI 디자인 + anti-slop** — web-and-frontend-development 903 (openclaw 리스트). 반대 운동: anti-slop/deslop 스킬 (oh-my-codex ai-slop-cleaner, oh-my-claudecode ai-slop-cleaner + minimal-prose/code-discipline, mblode/agent-skills, ZeroSlop, slop-grader, humanizer)
8. **프로덕트/마케팅 스킬** — awesome-agent-skills의 42%가 비엔지니어링: phuryn/pm-skills(65), deanpeters/Product-Manager-Skills(46), coreyhaines31/marketingskills(31), realkimbarrett/advertising-skills(12). 엔지니어링 프로세스 팩 최고: **garrytan/gstack**(27개)
9. **배포/문서/CI-GitHub** — 벤더별 deploy 스킬 (vercel/netlify/cloudflare/render), GitHub ops (gh-fix-ci, gh-address-comments, yeet, create-pr)
10. **비용/토큰/사용량 텔레메트리** — token-tracker, tokenscope, token-monitor, mystatus, quota, throughput, usage-monitor, otel 등 큰 클러스터

## 주목할 개별 엔트리

- **garrytan/gstack** — 27개. Real Chromium browse, design-review, cso(OWASP+STRIDE), review(staff-eng), canary(post-deploy SRE), ship/land-and-deploy, investigate, careful/freeze
- **obra/superpowers** — TDD/debugging/review 표준 세트, VoltAgent 양쪽 리스트 + oh-my-claudecode 인용
- **openai/*** (42개) — Codex 플러그인 디렉토리: figma(7), playwright, security-best-practices/threat-model/ownership-map, linear, sentry, notion(4), deploy(cloudflare/netlify/render/vercel), doc/pdf/slides/spreadsheet, imagegen/sora/speech/transcribe
- **muratcankoylan/Agent-Skills-for-Context-Engineering** — 가장 엄밀한 개념 스킬 세트
- **xberg-io/plugins** — 코드 인텔 + 크롤링 + 문서 추출 + 143 LLM provider, OpenCode 플러그인/스킬/MCP 3형태
- **czlonkowski/n8n-skills** (7개) — n8n 워크플로 authoring/expression/validation/MCP
- **steipete** — OpenClaw 창작자의 ClawHub 18개 스킬
- 하니스 수렴 어휘: `code-review/review`, `ralph`, `team`, `plan`, `autopilot`, `wiki`, `hud`, `doctor`, `ask`, `verify`가 oh-my-codex와 oh-my-claudecode에 거의 동일 의미로 존재

## 배포 채널 (리스트보다 중요)

- **skills.sh** — 6,579 refs, Claude/Codex/OpenCode 스킬 표준의 주 레지스트리
- **ClawHub / clawskills.sh** — 575 refs, OpenClaw 레지스트리
- **officialskills.sh** — 593 링크 (VoltAgent/awesome-agent-skills 내)
- **agentskills.io** — Agent Skills 스펙 (36 refs)
- **registry.modelcontextprotocol.io**, **mcp.so**, **smithery**, **glama**, **Composio** — MCP 디스커버리
- **modelcontextprotocol/servers** — 사실상 빈 껍데기 (7개만 남음, 나머지는 servers-archived)

## 추가 클론 후보 (이미 클론됨)

Tier 1: obra/superpowers, garrytan/gstack, VoltAgent/awesome-claude-code-subagents, VoltAgent/awesome-codex-subagents, VoltAgent/awesome-clawdbot-skills, muratcankoylan/Agent-Skills-for-Context-Engineering, NeoLabHQ/context-engineering-kit, anthropics/claude-plugins-official, mblode/agent-skills, xberg-io/plugins
Tier 2: phuryn/pm-skills, deanpeters/Product-Manager-Skills, coreyhaines31/marketingskills, LambdaTest/agent-skills, czlonkowski/n8n-skills, cypress-io/ai-toolkit, snyk/agent-scan, hashgraph-online/awesome-codex-plugins, gokapso/agent-skills, resend/resend-skills, trailofbits/skills
Tier 3 (pi-mono): kcosr/pi-extensions, aliou/pi-extensions, tmustier/pi-extensions, cv/pi-ssh-remote, jyaunches/pi-canvas, mrexodia/pi-cost-dashboard, code-yeongyu/pi-lsp-client, pi-ast-grep, pi-agent-system, pi-comment-checker
Tier 4: KyleAMathews/claude-code-ui, juanibiapina/gob, zenobi-us/pi-ds, MoonshotAI/kimi-code, xai-org/plugin-marketplace, Yeachan-Heo/gajae-code, VoltAgent/voltagent, modelcontextprotocol/registry, modelcontextprotocol/servers-archived, marlocarlo/psmux
