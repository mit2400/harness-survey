# 비전 에이전틱 기능 & 컴퓨터 유즈 비교

> 8개 하니스의 비전(이미지/영상 이해), 컴퓨터 유즈(데스크톱 제어), 브라우저 자동화, 이미지 생성, 비전 관련 에이전트/스킬을 소스 증거 기준으로 정리.
> 레포지토리는 `/home/minkoo/study/repos/` 기준. `claude-code` 레포는 배포판이라 구현이 없어 CHANGELOG 참조만 존재.

## 요약

| 하니스 | 비전/이미지 분석 | 컴퓨터 유즈(데스크톱) | 브라우저 자동화 | 이미지 생성 | 비전 에이전트/스킬 |
|---|---|---|---|---|---|
| opencode | 이미지 첨부만 (전용 툴 없음) | **없음** | **없음** (E2E 테스트 인프라만) | **없음** (프로토콜 passthrough만) | 없음 |
| oh-my-openagent | **`look_at` 툴** + 전용 비전 모델 카탈로그 | **`computer`/`computer_actions` + Rust 엔진(`senpi-desktop-*`)** | browser 스킬(omowright) + playwright/dev-browser MCP + ultimate-browsing | 없음 (외부 imagegen 스킬 참조만) | **`multimodal-looker` 에이전트**, `visual-qa` |
| pi-mono | `read` 툴 이미지 지원 (전용 툴 없음) | **없음** | **없음** | 라이브러리 API만 (`generateImages`) | 없음 |
| oh-my-pi | **`read <image>?q=`** + text-only 모델 비전 폴백 + `modelRoles.vision` | **`computer` Eval 프렐류드** (네이티브 Rust 데스크톱) | **`browser` Eval 프렐류드** (Chromium/CDP/relay/Tern/cmux) | **`generate_image` 툴** (5 transports) | `vision` 모델 롤 + image-question (전용 에이전트 없음) |
| codex | **`view_image` 툴** | 설정/게이트만 (실행은 번들 플러그인) | 설정/CDP 게이트만 (실행은 `chrome@openai-bundled`) | **`image_gen.imagegen` 툴 + `imagegen` 스킬** | `imagegen` 스킬, guardian multimodal |
| claude-code | **없음** (SDK 타입 선언만) | 없음 (CHANGELOG 참조만) | 없음 (Claude in Chrome은 별도 배포) | **없음** | 없음 |
| openclaw | **`view_image` 툴** + media-understanding 엔진 | **`computer` 툴** + `cua-computer` 확장 + Peekaboo | **`browser` 툴** (CDP + Playwright + chrome-devtools-mcp) | **`image_generate` 툴** (11 providers) | browser-automation/peekaboo/camsnap/visualize 등 |
| hermes-agent | **`vision_analyze`/`video_analyze` 툴** | **`computer_use` 툴** + `cua-driver` + `bot_desktop` | **18개 `browser_*` 툴** + CDP 슈퍼바이저 + Camofox | **`image_generate` 툴** (8개 인트리 플러그인) | `computer-use` SKILL, 정밀 비전 라우팅 |

**범례**: `look_at`(omo) / `view_image`(codex, openclaw) / `read ?q=`(oh-my-pi) / `vision_analyze`(hermes) 처럼 같은 "이미지를 본다" 기능도 하니스마다 이름과 구현이 전혀 다름.

---

## opencode
**비전/이미지 분석 (툴 없음, 첨부만)**
- 이미지를 모델에 붙이는 유일한 경로는 `read` 툴: `packages/core/src/tool/read.ts` — `toModelOutput`이 `image/jpeg|png|gif|webp`를 `{type:"file", data, mime}`로 반환.
- 이미지 정규화/리사이즈: `packages/core/src/image.ts` (`Image.Service.normalize`), `packages/core/src/image/photon.ts` (photon WASM, Lanczos3), `packages/core/src/config/attachments.ts` (max_width/height 2000, 5MB).
- 멀티모달 wire 인코딩은 프로토콜별: `packages/llm/src/protocols/{anthropic-messages,openai-chat,openai-responses,gemini,bedrock-converse}.ts`.
- 비전 모델 능력 감지: `packages/opencode/src/plugin/github-copilot/models.ts` (`supports.vision` → `capabilities.input.image`), `packages/core/src/models-dev.ts` (modality 리터럴).
- `look_at`, `view_image`, `vision_analyze`, `multimodal-looker` **없음**.

**컴퓨터 유즈**: 없음. `packages/opencode/src/mcp/browser.ts`는 이름과 달리 OS `openUrl()`만 호출. `packages/desktop/src/`는 Electron 셸일 뿐 입력 합성 없음. `openai-responses.ts`에 `computer_use_call` 타입 passthrough만 존재.

**브라우저 자동화**: 에이전트 능력 없음. `packages/app/playwright.config.ts`, `packages/app/e2e/**`는 자체 UI 테스트용. `packages/core/src/plugin/skill/customize-opencode.md` L111-113의 playwright MCP는 사용자 MCP 설정 **문서 예시**일 뿐.

**이미지 생성**: 에이전트 툴 없음. GitHub Copilot 프로토콜 계층에만 존재: `packages/core/src/github-copilot/responses/tool/image-generation.ts` (`openai.image_generation`), `openai-responses-prepare-tools.ts` L111-133. 등록 사이트가 없어 caller가 제공해야만 도달 가능.

**비전 에이전트/스킬**: 없음. 플러그인은 전부 provider 인증용.

---

## oh-my-openagent (OmO)
가장 깊은 컴퓨터 유즈 + 고유한 비전 서브에이전트를 가진 하니스.

**비전/이미지 분석 — `look_at` 툴**
- `packages/omo-opencode/src/tools/look-at/` — `look_at` 툴. `tools.ts` 스키마(`file_path`/`file_paths`/`image_data`/`image_data_list`/`goal`)가 자식 `multimodal-looker` 세션을 스폰해 요약만 회수.
- `look-at-session-runner.ts`(자식 세션 생성·첨부·재귀 방지), `session-poller.ts`(idle까지 1s 폴링, 120s 타임아웃), `image-converter.ts`(HEIC/WebP/RAW/PSD → JPEG, sips/ImageMagick), `mime-type-inference.ts`, `multimodal-fallback-chain.ts`(GPT-5.6 Sol→Kimi K3→GLM-4.6V→GPT-5 Nano).
- 게이트: `multimodal-looker`가 `disabled_agents`에 없을 때만 등록 (`plugin/tool-registry-core-tools.ts` L139).

**컴퓨터 유즈 — 네이티브 Rust 엔진 + CUA**
- TS 툴: `packages/senpi-desktop-tool/src/tool-definition.ts` (`COMPUTER_TOOL_NAME="computer"`), `cua-definition.ts` (`computer_actions`, OpenAI CUA 스키마), `cua-actions.ts` L54(9개 액션), `permission.ts`(`computer:read`/`computer:exec`).
- 프렐류드: `packages/senpi-desktop-prelude/src/{prelude.js,prelude.py}` — Eval 커널 `computer` 글로벌.
- 프로토콜/서비스/엔진: `packages/senpi-desktop-{protocol,service,engine}/src/`.
- 4개 OS 백엔드 Rust 크레이트: `crates/senpi-desktop-backend-macos`(ScreenCaptureKit+Skylight+AX), `-win32`(GDI+SendInput+UIA+UIPI), `-x11`(RandR+XTEST), `-wayland`(PipeWire+libei), `-atspi`(AT-SPI), `-fake`(테스트).
- 세션/안전: `crates/senpi-desktop-{core,session,safety,engine}`; 엔진은 MCP stdio 서버(`--mcp`)도 제공.
- 의존성 증거: `Cargo.toml` — `xcap`, `core-graphics`, `objc2-*`, `x11rb`, `atspi`, `ashpd`, `reis`, `enigo`, `uiautomation`, `windows-sys`.
- 문서: `docs/guide/computer-use.md`, `docs/reference/computer.md`.

**브라우저 자동화**
- `packages/shared-skills/skills/browser/SKILL.md` — 2엔진: attached(`connectBrowserSkill()`, 사용자 로그인 브라우저) vs owned(`connectPipe()`/`connectCloakProfile()`, stealth/네트워크 캡처).
- `browser/scripts/omowright.mjs`(JS Eval 커널용 omowright 라이브러리), `browser-engine-guard.mjs`(`OMO_BROWSER_ENGINE` 게이트).
- `packages/shared-skills/skills/ultimate-browsing/SKILL.md` + `engine/`(Python: curl_cffi TLS impersonation, yt-dlp, Jina Reader, Playwright 실브라우저 폴백).
- builtin MCP 스킬: `packages/skills-loader-core/src/features/builtin-skills/skills/playwright-mcp-skill.ts` (`@playwright/mcp`), `playwright-cli.ts`, `dev-browser/SKILL.md`.
- 설정 스키마: `packages/omo-opencode/src/config/schema/browser-automation.ts` (`playwright`/`playwright-cli`/`dev-browser`, `playwright_mcp_args`).
- `skill_mcp` 툴(`packages/omo-opencode/src/tools/skill-mcp/tools.ts`)로 chrome-devtools 등 임의 MCP 연결.

**이미지 생성**: 인트리 구현 **없음**. `imagegen`은 세션이 보유 시 로드하는 **외부 스킬** 참조뿐: `shared-skills/skills/ulw-research/SKILL.md` L333·338, `frontend/SKILL.md`. `scripts/generate-opengateway-models.ts` L230은 이미지 생성 모델을 카탈로그에서 명시 제외.

**비전 에이전트/스킬**
- **`multimodal-looker` 서브에이전트**: `packages/omo-opencode/src/agents/multimodal-looker.ts` (temp 0.1, 툴 allowlist `["read"]`, `READ_ENABLED=false`).
- 비전 모델 캐시: `plugin-state.ts`(`VisionCapableModel`), `shared/vision-capable-models-cache.ts`, `generated/model-capabilities.generated.json`(2758 모델, 31+ `input:["image"]`).
- `visual-qa` 스킬: `shared-skills/skills/visual-qa/SKILL.md` + `scripts/`(실제 PNG 디코드/diff, `visual-qa.mjs`).
- `computer-use` 스킬도 컴퓨터 툴과 함께 배포.

---

## pi-mono
**비전/이미지 분석 (read 툴에 내장)**
- `packages/coding-agent/src/core/tools/read.ts` — 이미지 MIME 감지 후 `{type:"image", data, mimeType}` 반환, 비전 미지원 모델엔 안내문. jpg/png/gif/webp/bmp.
- 파이프라인: `utils/image-process.ts`, `utils/mime.ts`(매직바이트), `utils/image-resize{,-core,-worker}.ts`, `utils/image-convert.ts`, `utils/exif-orientation.ts`, `utils/photon.ts`.
- 툴 결과 이미지 정규화: `utils/tool-result-images.ts`; 클립보드 붙여넣기: `utils/clipboard-image.ts`.
- 터미널 렌더: `packages/tui/src/terminal-image.ts`(kitty/iTerm2), `packages/tui/src/components/image.ts`.
- code-mode: `packages/codemode/src/{declarations.ts,runtime/prelude-source.ts}` — `image()` 출력 헬퍼.
- `view_image`/`look_at`/`vision_analyze`/`multimodal-looker` **없음** — 비전은 전적으로 `read`에 통합.

**컴퓨터 유즈**: 없음. 화면 캡처/입력 합성 코드 없음.

**브라우저 자동화**: 없음. `packages/coding-agent/src/utils/open-browser.ts`는 OS 기본 핸들러(`open`/`xdg-open`/`rundll32`)로 URL만 열 뿐. Playwright/Puppeteer/CDP 없음.

**이미지 생성 (라이브러리 API만, 에이전트 툴 아님)**
- `packages/ai/src/images.ts` (`generateImages`), `images-api-registry.ts`, `providers/images/register-builtins.ts`, `api/openrouter-images.ts`, `image-models.ts`, `models.ts`(`Models.generateImages()`, L948).
- coding-agent 툴셋은 `bash/edit/find/grep/ls/powershell/read/write`뿐 — 이미지 생성 툴 없음. code-mode 샌드박스 `models.generateImages`로만 도달.

**비전 에이전트/스킬**: 없음. `.pi/skills/`는 개발용(add-llm-provider 등), 예시 서브에이전트(planner/worker/scout/reviewer)는 비전 무관.

---

## oh-my-pi
비전·컴퓨터 유즈·브라우저·이미지 생성 4개 모두 자체 구현. 단, 컴퓨터/브라우저는 AgentTool이 아니라 **Eval 프렐류드**.

**비전/이미지 분석 — `read ?q=` + 비전 폴백**
- `packages/coding-agent/src/tools/read.ts` + `docs/tools/read.md` — `read <image>?q=<question>`이 비전 모델에 질의해 텍스트 반환; `?q=` 없으면 image-capable 모델엔 인라인 이미지 블록, text-only 모델엔 메타데이터+힌트.
- `packages/coding-agent/src/utils/image-question.ts` — `@vision`→`@default`→active 순으로 비전 모델 해석, `images.questionTimeoutMs`.
- `packages/coding-agent/src/utils/image-vision-fallback.ts` — text-only 모델에 이미지가 첨부되면 세션 `local://`에 저장 후 비전 모델로 `<image path><description>` 텍스트 블록 생성.
- `packages/ai/src/providers/vision-guard.ts` (`sendsImageInputOnWire`), `utils/image-loading.ts`, `utils/image-resize.ts`, `utils/video.ts`.
- 문서 변환/PDF: `tools/read-pdf.ts`, `utils/markit.ts`(pdf/docx/pptx/xlsx/epub), PDF 페이지는 headless Chromium 스크린샷.

**컴퓨터 유즈 — `computer` Eval 프렐류드 (네이티브 Rust 데스크톱)**
- `packages/coding-agent/src/tools/computer.ts` + `tools/computer/{call,protocol,worker}.ts`, 프렐류드 `computer/{prelude.js,prelude.py,declarations.d.ts}`.
- 네이티브 구현: `crates/pi-natives/src/desktop/` (macOS Linux X11/Wayland/Windows, capture+AX+input+clipboard).
- 문서: `docs/computer-use.md`, `docs/tools/computer.md`, `prompts/system/computer-use.md`.
- 설정 `computer.enabled` 기본 `false`; `/computer` 토글. 창/디스플레이 열거, 스크린샷, AX 트리, 픽셀 입력, `computer.run` 지속 세션. **주의: 샌드박스 아님**(Bun/Node 풀 호스트 접근).

**브라우저 자동화 — `browser` Eval 프렐류드**
- `packages/coding-agent/src/tools/browser.ts` + `tools/browser/`(40+ 파일: `tab-supervisor`, `tab-worker`, `registry`, `launch`, `attach`, `screenshot`, `aria/`, `relay/`, `cmux/`, `tern/`, `react/`).
- 백엔드: managed Chromium(스텔스 패치), spawned, connected CDP, relay(사용자 실 Chrome 탭), Tern(native WKWebView), cmux.
- 문서: `docs/tools/browser.md`. `browser.enabled` 기본 `true`.

**이미지 생성 — `generate_image` 툴**
- `packages/coding-agent/src/tools/image-gen.ts` + `prompts/tools/image-gen.md` + `sdk.ts`(`imageGenTool`); `generate_image.enabled` 기본 `false`.
- provider transports: `packages/ai/src/images/` (openai-images, openrouter-images, google-generative-ai, google-antigravity, openai-hosted, openai-images 등).
- text-to-image / image edit / text rendering / model 선택. `modelRoles.image` + `retry.fallbackChains.image`. 문서: `docs/tools/generate_image.md`.

**비전 에이전트/스킬**
- 전용 비전 에이전트는 없고 `modelRoles.vision` 롤 + `read ?q=` + image-vision-fallback으로 처리.
- 프롬프트 스킬: `prompts/skills/` (system-prompts, semantic-compression, tool-prompt-optimization) — 비전 전용 아님.

---

## codex
**비전/이미지 분석 — `view_image` 툴 (일급)**
- `codex-rs/core/src/tools/handlers/view_image.rs` — `ViewImageHandler`: `InputModality::Image` 확인, 샌드박스 FS로 읽고 `image::load_from_memory` 검증, `InputImage`(data URL) 반환.
- `view_image_spec.rs` — 스키마(`path`, `detail: high|original`, `environment_id`), `spec_plan.rs` L1270 등록.
- 지원 파이프라인: `tools/src/image_detail.rs`, `core/src/image_preparation.rs`, `utils/image/src/lib.rs`(리사이즈/인코드, MAX_DIMENSION 2048), `core/src/original_image_detail.rs`, `model-provider/src/capabilities.rs`.
- TUI: `tui/src/bottom_pane/chat_composer.rs`(로컬 이미지 붙여넣기/원격 URL), `tui/src/goal_files.rs`.
- 테스트: `core/tests/suite/view_image.rs`, `image_rollout.rs`.

**컴퓨터 유즈 — 설정/게이트만, 실행은 번들 플러그인**
- `codex-rs/config/src/computer_use.rs` (`ComputerUseConfigToml`: macOS bundle_ids, Windows aumids/exes), `config/src/browser_computer_use_requirements.rs`.
- `features/src/lib.rs` — `Feature::ComputerUse`는 "Requirements-only gate for desktop apps".
- `core-plugins/src/discoverable.rs` L47 — `computer-use@openai-bundled` 플러그인 id.
- OSS 레포에 화면 캡처/입력 크레이트 **없음**.

**브라우저 자동화 — 설정/CDP 게이트 + 외부 플러그인**
- `codex-rs/config/src/browser_use.rs` (per-origin `access/downloads/uploads/full_cdp_access`), `features/src/lib.rs`(`BrowserUse`, `BrowserUseFullCdpAccess`, `InAppBrowser`, `BrowserAnnotationApi`), `app-server-protocol/src/protocol/v2/browser_use_config.rs`.
- MCP 브리지: `core/src/mcp_tool_call.rs` (`access_browser_origin` elicitation, `browser-use` connector), `sandboxing/src/seatbelt_tests.rs`(`/tmp/codex-browser-use` 소켓).
- `core-plugins/src/discoverable.rs` L46 — `chrome@openai-bundled` 플러그인 id. Playwright/Puppeteer/CDP 클라이언트 **없음**.

**이미지 생성 — `image_gen.imagegen` 툴 + 스킬 (일급)**
- `codex-rs/ext/image-generation/src/tool.rs` — `image_gen.imagegen` 툴: `prompt`, `transparent_background`, `referenced_image_paths`(최대 5), `num_last_images_to_include`; 모델 `gpt-image-2`. `lib.rs`(`IMAGE_GEN_NAMESPACE="image_gen"`, `IMAGEGEN_TOOL_NAME="imagegen"`), `backend.rs`, `artifact.rs`, `extension.rs`.
- API 클라이언트: `codex-api/src/endpoint/images.rs`, `codex-api/src/images.rs`(`ImageGenerationRequest`, `ImageEditRequest`).
- `features/src/lib.rs` L1623 `Feature::ImageGeneration`.
- code-mode: `code-mode-protocol/src/description.rs` `generatedImage()`.

**비전 관련 스킬/에이전트**
- `imagegen` 스킬: `codex-rs/skills/src/assets/samples/imagegen/SKILL.md` + `scripts/image_gen.py`(generate/edit/generate-batch CLI 폴백) + `references/` + `agents/openai.yaml`.
- guardian 멀티모달: `guardian-context/src/images.rs`, `node_repl.rs`(`Multimodal` REPL 스크린샷).
- 번들 플러그인 id: `chrome@openai-bundled`, `computer-use@openai-bundled`.

---

## claude-code
**레포 성격**: 애플리케이션 소스가 아니라 **플러그인/스킬/모드 배포판 + 문서 + CHANGELOG**. Rust/`crates/`/`src/` 앱 없음. 5개 카테고리 모두 이 레포에 구현 없음.

**비전/이미지 분석**: 없음(구현). `mods/types/claude-code.d.ts`에 SDK 타입 선언(`type: 'image'`, `Image` 블록)만. `plugins/frontend-design/skills/frontend-design/SKILL.md` L59는 "환경이 지원하면 스크린샷" 조언뿐. `look_at`/`view_image`/`imagegen`/`multimodal`/`vision` 전 레포 0건.

**컴퓨터 유즈**: 없음(구현). CHANGELOG 산문만 — `CHANGELOG.md:2405`(macOS 컴퓨터 유즈 권한), `:5595`(`switch_display` 멀티모니터 수정), `:1156`(Computer Use 문서 URL). 크레이트 없음.

**브라우저 자동화**: 없음(구현). `playwright`/`puppeteer`/`chrome-devtools` 0건. "Claude in Chrome"(별도 배포 확장)이 CHANGELOG에 36회 등장(`:3207` `save_to_disk`, `:1170` `browser_batch`, `:1848` `file_upload`). `mods/diff/.../gutter-chrome.ts`는 터미널 거터 장식 폭이라 무관.

**이미지 생성**: 없음. `image_generation`/`imagegen` 전 레포 0건.

**비전 관련 에이전트/플러그인/스킬**: 없음. 14개 플러그인 모두 개발 워크플로우용(code-review, feature-dev, frontend-design, plugin-dev 등). `mods/`는 agents-md/diff/sec-default/telemetry.

> 참고: Claude Code의 실제 비전/컴퓨터 유즈 동작은 사유 컴파일 CLI에 있고, 이 레포에는 없음.

---

## openclaw
5개 카테고리 전부 깊게 구현.

**비전/이미지 분석 — `view_image` 툴 + media-understanding 엔진**
- `src/agents/tools/image-tool.ts` — `view_image` 툴("Image understanding"). 구 명칭 `image` → `view_image` 마이그레이션: `src/commands/doctor/shared/legacy-tool-name-migration.ts:17`.
- 실행/헬퍼/결과: `image-tool.model-execution.ts`, `image-tool.helpers.ts`, `image-tool.result.ts`; 등록 `src/agents/core-tool-factory-descriptors.ts:43`, 카탈로그 `src/agents/tool-catalog.ts:499`.
- 엔진: `src/media-understanding/` — `image-runtime.ts`(`describeImageWithModel`), `image-model-runtime.ts`(비전 모델+자격 해석), `runner.ts`(`modelSupportsVision` 게이팅), `attachments*.ts`, `provider-registry.ts`.
- SDK 계약: `src/plugin-sdk/media-understanding-runtime.ts`, `media-understanding.ts`.
- 이미지 입력 직렬화: `src/media/anthropic-inline-images.ts`(jpeg/png/gif/webp, 10MB 캡), `packages/media-core/src/inline-image-data-url.ts`, `src/media/prompt-image-input.ts`("Stored images require vision").
- 브라우저 스크린샷→텍스트: `extensions/browser/src/browser/vision.ts` (`describeBrowserScreenshot`), `browser/screenshot-annotate.ts`(ref 바운딩박스 라벨).
- 멀티모달 메모리: `src/memory-host-sdk/multimodal.ts`.
- `look_at`/`multimodal-looker` 없음.

**컴퓨터 유즈 — `computer` 툴 + `cua-computer` + Peekaboo**
- `src/agents/tools/computer-tool.ts` + 모듈 패밀리(`computer-tool-{schema,request,result,guidance,node,control,gateway,bindings,outcome,shared}.ts`).
- 액션 계약: `src/plugins/computer-use-contract.ts:9` — 25개 v2 액션(screenshot, left_click, ..., get_accessibility_tree, launch_app, set_value, zoom).
- 노드 실행: `src/node-host/computer-command.ts`(`computer.act`), `src/node-host/desktop-stream-command.ts`(RFB/VNC 5900), `src/worker/node-desktop-protocol.ts`; `src/agents/computer-use-node-capabilities.ts`.
- 확장: `extensions/cua-computer/`(category `computer-use`, Windows/Linux — `src/actions.ts`, `window-actions.ts`, `browser-actions.ts`, `mcp-driver-client.ts`, `frame.ts`).
- macOS 기본 provider: `skills/peekaboo/SKILL.md`(외부 CLI). 카메라: `skills/camsnap/SKILL.md`.
- Rust 크레이트는 화면 캡처/입력 아님(`openclaw-node-host`=노드 런타임, `openclaw-gateway-client`=WebSocket 전송).
- 문서: `docs/nodes/computer-use.md`, `docs/plugins/codex-computer-use.md`.

**브라우저 자동화 — `browser` 툴 (3백엔드)**
- `extensions/browser/plugin-registration.ts:316-321` — `browser` 툴 등록; `src/browser-tool.schema.ts` — 24개 action(`open/navigate/snapshot/screenshot/...`) + 14개 act kind.
- raw CDP: `extensions/browser/src/browser/cdp*.ts` (`cdp.ts`, `cdp-page-session.ts`, `cdp-ax.ts`, `cdp-role-snapshot*.ts` 등).
- Playwright: `browser/pw-session*.ts`, `pw-ai.ts`, `pw-tools-core.*`, `playwright-core.runtime.ts`.
- chrome-devtools MCP: 번들 의존성 `chrome-devtools-mcp@1.10.1`(`package.json:2241,2267`), 패치 `patches/chrome-devtools-mcp@1.10.1.patch`, 배선 `browser/chrome-mcp*.ts`.
- Chrome 확장 relay: `extensions/browser/chrome-extension/`; SDK: `src/plugin-sdk/browser-{cdp,config,types}.ts`; Lightpanda: `deploy/lightpanda/README.md`.
- 스킬: `extensions/browser/skills/browser-automation/SKILL.md`. Puppeteer 없음.

**이미지 생성 — `image_generate` 툴 + 11 providers**
- `src/agents/tools/image-generate-tool.ts`(비동기 task + detached 완료 wake), `.execution.ts`, `.actions.ts`; 카탈로그 `src/agents/tool-catalog.ts:506`.
- 런타임: `src/image-generation/`, `src/media-generation/`, `extensions/image-generation-core/runtime-api.ts`.
- providers: openai, google, fal, comfy, deepinfra, litellm, microsoft-foundry, minimax(+portal), openrouter, vydra, xai.
- 형제: `src/video-generation/` + `video_generate`(18 backends). 문서: `docs/tools/image-generation.md`.

**비전 관련 스킬**: `browser-automation`, `peekaboo`(macOS UI 캡처/자동화), `camsnap`(RTSP/ONVIF), `meme-maker`(Playwright), `visualize`, `diagram-maker`, `songsee`(스펙트로그램), `canvas`.

---

## hermes-agent
비전·컴퓨터 유즈·브라우저·이미지 생성 전부 인트리 구현. 특히 비전 라우팅 비용 제어가 정교함.

**비전/이미지 분석 — `vision_analyze`/`video_analyze`**
- `tools/vision_tools.py`(1189줄) — `vision_analyze`(스키마 L956) + `video_analyze`(L1147). `image_url`(http/local/data), `question`, `region` [x1,y1,x2,y2] 크롭. 네이티브 모델엔 픽셀을 멀티모달 툴 결과 봉투로 붙이고, 아니면 보조 비전 LLM으로 라우팅.
- 보조: `tools/vision_tools_image_prep.py`(포맷 감지/크롭/검증), `tools/vision_tools_history_budget.py`(`vision.embed_target_bytes` 256KB, `vision.max_calls_per_image`), `tools/image_source.py`(SSRF/path allowlist).
- 라우팅/비용: `agent/image_routing.py`(562줄, `auto|native|text`), `agent/vision_message_prep.py`(359줄), `agent/image_token_cost.py`, `agent/image_eviction_policy.py`.
- `read_file`(`tools/file_tools.py:1156`)은 이미지를 거부하고 `vision_analyze`로 안내.
- 게이트웨이: `gateway/run_inbound.py`(인바운드 이미지 자동 분석), `acp_adapter/content.py`(ImageContentBlock), `mcp_serve.py`.
- 문서: `website/docs/user-guide/features/vision.md`.

**컴퓨터 유즈 — `computer_use` + `cua-driver` + `bot_desktop`**
- `tools/computer_use/tool.py`(964줄), `schema.py`(단일 action 디스크리미네이터 14 액션: capture/click/drag/scroll/type/key/set_value/wait/list_apps/list_windows/focus_app; 캡처 모드 `som`/`vision`/`ax`).
- 백엔드: `cua_backend.py`(기본 MCP-over-stdio → `cua-driver`; 포커스 안 뺏는 background primitive), `cua_backend_{capture,input,parse,session,daemon,driver}.py`, `backend.py`, `permissions.py`(macOS TCC), `doctor.py`(`hermes computer-use doctor`), `vision_routing.py`.
- Bot Desktop(프로필별 headless Xfce + RFB + human takeover): `tools/bot_desktop/{runtime.py,launcher.sh,rfb_filter.py,lease.py,...}`.
- CLI: `hermes_cli/subcommands/computer_use.py`, `computer_use_screen.py`.
- 테스트: `tests/computer_use/`(11 파일). 스킬: `skills/autonomous-ai-agents/computer-use/SKILL.md`.
- 문서: `website/docs/user-guide/features/{computer-use,bot-screen}.md`.
- 외부 카탈로그: `plugin-catalog/hermes-windows-computer-use.yaml`(THEIA).

**브라우저 자동화 — 18개 `browser_*` 툴**
- `tools/browser_tool.py`(1367줄) — `browser_navigate/snapshot/click/type/scroll/back/press/get_images/vision/console`; 백엔드 local Chromium, Browser Use/Browserbase/Firecrawl, 사용자 CDP, Camofox.
- CDP: `tools/browser_cdp_tool.py`(`browser_cdp`), `browser_supervisor.py`(지속 CDP 슈퍼바이저, OOPIF/worker 추적), `browser_supervisor_{dialogs,frames}.py`, `browser_dialog_tool.py`.
- 비전 브리지: `tools/browser_tool_vision.py`(`browser_vision`, `_native_vision_result`, aux LLM 폴백). Lightpanda: `browser_lightpanda.py` + `browser_tool_lightpanda_fallback.py`. Camofox: `browser_camofox.py`.
- 자격증명: `tools/browser_vault_tool.py`. 클라우드 providers: `plugins/browser/{browser_use,browserbase,firecrawl}/provider.py`.
- 툴셋: `toolsets.py::_HERMES_CORE_TOOLS`(browser_* 전부). **Playwright는 `tests-js/package.json`/`apps/desktop/playwright.config.ts` 테스트 하니스에만**, Puppeteer 없음, chrome-devtools-mcp는 문서상 외부 MCP 예시만.

**이미지 생성 — `image_generate` + 8개 인트리 플러그인**
- `tools/image_generation_tool.py`(935줄) — `image_generate`(스키마 L554), text-to-image + edit, `get_tool_definitions()` 시점에 활성 backend 능력으로 스키마 재구성.
- 카탈로그/관리: `image_generation_catalog.py`, `image_generation_managed.py`(`image_gen.provider: nous`), `fal_common.py`; provider ABC `agent/image_gen_provider.py` + `agent/image_gen_registry.py`.
- 플러그인: `plugins/image_gen/{fal,openai,openai-codex,xai,krea,openrouter,deepinfra,meta-ai}/`.
- 형제: `tools/video_generation_tool.py`(`video_generate`), `tools/xai_video_tools.py`.
- 문서: `website/docs/user-guide/features/image-generation.md`.

**비전 관련 스킬/에이전트**
- `skills/autonomous-ai-agents/computer-use/SKILL.md`(유일한 전용 퍼셉션/액션 스킬, v2.1.0).
- `skills/media/songsee/`(오디오 스펙트로그램), 각종 creative 스킬이 `vision_analyze`/`image_generate` 소비.
- MCP 노출: `agent/transports/hermes_tools_mcp_server.py:87`.
- `multimodal-looker` 에이전트/비전 서브에이전트 **없음** — 비전은 `vision_analyze`/`browser_vision`/`computer_use(capture)` 툴로만 노출.

---

## 하니스별 특징 요약

- **네이티브 데스크톱 제어(자체 Rust/네이티브 엔진)**: oh-my-openagent(`senpi-desktop-*`), oh-my-pi(`pi-natives/src/desktop/`). 둘 다 4개 OS 백엔드 + AX + 백그라운드 입력을 갖춘 유이한 하니스.
- **전용 비전 서브에이전트**: oh-my-openagent의 `multimodal-looker`가 유일. 나머지는 툴/모델 롤로 처리.
- **`view_image` 네이티브 툴**: codex, openclaw. **`look_at` 위임 툴**: oh-my-openagent. **`read ?q=`**: oh-my-pi. **`vision_analyze`**: hermes-agent. **전용 툴 없는 이미지 첨부**: opencode, pi-mono.
- **이미지 생성 실구현**: codex(`imagegen`), openclaw(11 providers), hermes-agent(8 plugins), oh-my-pi(`generate_image`). **라이브러리만**: pi-mono. **외부 스킬 참조만**: oh-my-openagent. **없음**: opencode, claude-code.
- **브라우저 자동화 실구현**: oh-my-openagent, oh-my-pi, openclaw, hermes-agent. **설정/CDP 게이트만**: codex. **없음**: opencode, pi-mono, claude-code.
- **claude-code 레포는 구현 증거 없음** — 컴파일된 사유 CLI만 실제 기능을 가지며, CHANGELOG가 유일한 단서.
