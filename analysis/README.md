# AI 코딩 하니스 비교 분석

대상: opencode, oh-my-openagent(OmO, omo-native 포함), pi-mono, oh-my-pi, codex, claude-code, openclaw, hermes-agent
클론 위치: `/home/minkoo/study/repos/`

## 문서 목록

### 토픽별 비교 문서 (`topics/`)

| 파일 | 내용 |
|---|---|
| [topics/memory.md](topics/memory.md) | 메모리 관리 방식 비교 (세션, compaction, 영구 메모리, 컨텍스트 파일) |
| [topics/mcp.md](topics/mcp.md) | MCP 지원 비교 (기본 서버, transport, exposure, OAuth) |
| [topics/skills.md](topics/skills.md) | 기본 스킬 비교 (내장 스킬 수, 발견 경로, 레지스트리) |
| [topics/tools.md](topics/tools.md) | 내장 툴 비교 |
| [topics/agents.md](topics/agents.md) | 에이전트/서브에이전트 비교 |
| [topics/hooks.md](topics/hooks.md) | 훅/플러그인 시스템 비교 |
| [topics/config.md](topics/config.md) | 설정 파일 비교 |
| [topics/unique-features.md](topics/unique-features.md) | 하니스별 독특한 기능 |
| [topics/ecosystem.md](topics/ecosystem.md) | 스킬/MCP 생태계 서베이 (awesome 리스트 분석) |
| [topics/vision-computer-use.md](topics/vision-computer-use.md) | 비전/이미지 툴, 브라우저 자동화, computer use 비교 |
| [topics/sandbox.md](topics/sandbox.md) | 샌드박스/격리/권한 승인 시스템 비교 |

### 하니스별 상세 문서 (`harnesses/`)

| 파일 | 하니스 |
|---|---|
| [harnesses/opencode.md](harnesses/opencode.md) | opencode (anomalyco/opencode) |
| [harnesses/oh-my-openagent.md](harnesses/oh-my-openagent.md) | oh-my-openagent / OmO / omo-native |
| [harnesses/pi-mono.md](harnesses/pi-mono.md) | pi-mono / pi.dev |
| [harnesses/oh-my-pi.md](harnesses/oh-my-pi.md) | oh-my-pi (omp) |
| [harnesses/codex.md](harnesses/codex.md) | codex (openai/codex) |
| [harnesses/claude-code.md](harnesses/claude-code.md) | claude-code (anthropics/claude-code) |
| [harnesses/openclaw.md](harnesses/openclaw.md) | openclaw |
| [harnesses/hermes-agent.md](harnesses/hermes-agent.md) | hermes-agent (NousResearch) |

## 한눈에 보기

| 하니스 | 메모리 | 내장 스킬 | 기본 MCP | 내장 툴 | 서브에이전트 | 훅 이벤트 |
|---|---|---|---|---|---|---|
| opencode | 없음(세션 SQLite만) | 1개 | 없음 | ~16 | 7개 | 20 hook 포인트 |
| oh-my-openagent | git MemFS + Kibitzer | 14 TS + 18 shared | 4개 | 12+26 | 11개 | 54~62개 (5-tier) |
| pi-mono | 없음(세션 JSONL) | 0개 | 없음(codemode exposure) | 8 (기본 4) | 없음(의도적) | 32개 |
| oh-my-pi | 5 백엔드 | 0개(rules 26개) | 없음 | 31 (essential 14) | 5개 | 40+ |
| codex | 2-phase 파이프라인(off) | 5개 | 없음 | ~20 | V1/V2 multi-agent | 12개 |
| claude-code | MEMORY.md 자동 | 바이너리 내장 | 없음(claude.ai 커넥터) | ~25 | 5종 + teams | 32 클래식 + ~90 function |
| openclaw | Dreaming + SQLite | 48개 | 없음 | 66개 | harness 5 + swarm | 3종(~40 typed) |
| hermes-agent | frozen-snapshot + 5 provider | 58개 + 152 optional | 없음(65 recipe) | 41 toolset | delegate + kanban | 4종(~50) |
