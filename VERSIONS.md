# 분석 대상 버전 (Pinned Versions)

이 서베이의 **모든 수치·파일 경로·기능 서술**은 아래 커밋 시점을 기준으로 수집했다.
8개 하니스 모두 주간/일간 릴리스이므로, 시간이 지나면 수치가 달라진다. 비교·확인 시 반드시 이 표의 버전을 먼저 대조할 것.

수집일: **2026-10-05** (클론 직후 HEAD)

## 하니스

| 하니스 | 저장소 | 커밋 | 커밋일 | 버전 |
|---|---|---|---|---|
| opencode | [`anomalyco/opencode`](https://github.com/anomalyco/opencode) | `907b3bc` | 2026-10-02 | `1.18.34` |
| oh-my-openagent (OmO) | [`code-yeongyu/oh-my-openagent`](https://github.com/code-yeongyu/oh-my-openagent) | `251cbfe` | 2026-10-04 | `5.1.13` |
| pi-mono | [`badlogic/pi-mono`](https://github.com/badlogic/pi-mono) | `8369268` | 2026-10-03 | `0.0.3` |
| oh-my-pi (omp) | [`can1357/oh-my-pi`](https://github.com/can1357/oh-my-pi) | `69e8c9e` | 2026-10-03 | `18.5.1` |
| codex | [`openai/codex`](https://github.com/openai/codex) | `b741e48` | 2026-10-03 | `0.0.0-dev` (Cargo) |
| claude-code | [`anthropics/claude-code`](https://github.com/anthropics/claude-code) | `1c229fc` | 2026-10-02 | **`v2.1.288`** |
| openclaw | [`openclaw/openclaw`](https://github.com/openclaw/openclaw) | `ee9127b7` | 2026-10-04 | `2026.9.8` |
| hermes-agent | [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) | `620ceb8` | 2026-10-03 | `1.0.0` |

> `pi-mono`의 `0.0.3`은 npm 패키지 버전이고, 커밋 기준으로 이례적으로 느리게 올라간다. 기능 판단은 커밋 SHA 기준으로 한다.

## 버전 민감도가 높은 항목

| 항목 | 기준 | 이유 |
|---|---|---|
| claude-code 전체 기능 | **`v2.1.288` + `CHANGELOG.md`** | 소스 미공개. 내부 동작 확인의 유일한 1차 출처가 CHANGELOG이며, 릴리스마다 기능이 추가·변경된다 |
| opencode 툴 목록 | `1.18.34`의 `tool/registry.ts` | GPT 계열 모델은 `edit` 대신 `apply_patch`를 노출하는 등 **모델별 분기**가 있어 버전이 바뀌면 개수가 달라진다 |
| OmO 툴·에이전트·훅 수 | `5.1.13` | beta를 지나며 빠르게 변경됨. `5.0.0-beta.18` 시점과 내용이 크게 다르다 |
| codex 툴·기능 게이트 | `0.0.0-dev` + feature flag 157개 | Cargo 버전이 `0.0.0` 고정이라 **커밋 SHA만 유효** |
| openclaw 스킬 48개 | `2026.9.8` | 번들 스킬이 매 릴리스 추가·삭제됨 |
| hermes-agent 스킬 58+152 | `1.0.0` | 번들/옵셔널 스킬 수가 릴리스마다 변동 |

## 재현 방법

```bash
# 특정 버전으로 재확인
cd repos/oh-my-openagent
git checkout 251cbfe

# 서베이 전체 재동기화 (버전 표 갱신 필요)
cd repos
for d in opencode oh-my-openagent pi-mono oh-my-pi codex claude-code openclaw hermes-agent; do
  printf "%-16s %s %s\n" "$d" "$(git -C $d rev-parse --short HEAD)" "$(git -C $d log -1 --format=%cs)"
done
```

`repos/`는 `.gitignore` 처리되어 있으므로 이 레포를 클론한 사람은 `repos/`가 없다. 위 명령으로 동일 상태를 만들 수 있다.

## 인용 경로 신뢰도

문서에 적힌 파일 경로는 대개 실재하지만 **전수 자동 검증은 않았다.** 작성 과정에서 다른 하니스의 경로가 잘못 기입된 사례가 확인됐으므로([security.md](topics/security.md)의 opencode 행), 중요한 인용은 `repos/` 원문과 대조하는 것을 권한다.