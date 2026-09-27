# CCX 프로젝트 분석 정리 (한국어)

> 작성일: 2026-09-27
> 작성: Claude Code 세션 기반 전수조사 결과 정리

## 저장소 정보

| 항목 | 값 |
|---|---|
| 내 저장소 | https://github.com/bmshin94/ccx |
| 업스트림(원본) | https://github.com/BenedictKing/ccx |
| Go 모듈명 | `github.com/BenedictKing/ccx` |
| 릴리스 | https://github.com/BenedictKing/ccx/releases/latest |
| 문서 사이트 소스 | `docs/` (VitePress) |
| 버전 | `v3.0.0` (루트 `VERSION` 이 유일 버전 소스) |
| 라이선스 | MIT |
| 커밋 기여 | BenedictKing 49, bmshin94 2 (= 포크) |

---

## 1. CCX가 무엇인가

CCX는 **다중 업스트림 AI API 프록시 + 프로토콜 변환 게이트웨이**다.

```text
Claude Code / Codex / Gemini CLI / Cursor
        │  (원래는 api.anthropic.com 으로 직행)
        ▼
   CCX :3000
        ├─ 프로토콜 변환 (Claude ↔ OpenAI ↔ Codex Responses ↔ Gemini)
        ├─ 채널 선택 (우선순위 / 건강도 / 비용 / 컨텍스트 적합성)
        ├─ 실패 시 자동 페일오버 + 키 로테이션
        └─ 전 요청 계측 (비용 / 지연 / 성공률)
        ▼
  Anthropic / OpenAI / Gemini / DeepSeek / Kimi / GLM / MiniMax / Qwen / Ollama ...
```

핵심 아이디어: **클라이언트는 손대지 않고, 뒤에 붙는 모델만 갈아끼운다.**
실제로 필요한 변경은 환경변수 한 줄뿐이다.

```bash
export ANTHROPIC_BASE_URL="http://localhost:3000"   # /v1 을 붙이지 않는다
export ANTHROPIC_API_KEY="내가-정한-PROXY_ACCESS_KEY"
```

### 쉬운 비유

- **통신사 대리점**: 전화기(Claude Code)는 그대로, 뒤에서 통신망(모델)을 자동으로 갈아끼움
- **실시간 통역사**: Claude 말투로 온 요청을 DeepSeek 말투로 번역, 응답도 역번역 (스트리밍 포함)
- **똑똑한 매니저**: 요청 난이도를 보고 "이건 싼 모델로 충분", "이건 컨텍스트 긴 모델로" 판단

---

## 2. 규모와 기술 스택

| 항목 | 값 |
|---|---|
| 추적 파일 | 1,773개 |
| Go | 953 파일 (테스트 `_test.go` 473개) |
| TypeScript | 260 파일 |
| Vue | 183 파일 |
| 문서 md | 110개 (docs 하위 79개, 한/영 이중) |
| CHANGELOG | 약 210KB (매우 빠른 개발 사이클) |

- 백엔드: Go 1.25 + Gin, SQLite(`modernc.org/sqlite`, 순수 Go), gjson/sjson, gorilla/websocket
- 프론트엔드: Vue 3 + Vuetify 3 + TypeScript + Vite, 패키지 매니저는 **Bun** (`bun.lock` 이 권위)
- 데스크톱: Wails 3 (Go + Vue), Microsoft Store / Homebrew cask / AppImage 배포
- 프론트 빌드 산출물은 `embed.FS` 로 Go 바이너리에 내장 → **단일 실행파일로 웹UI까지 동작**
- CI: GitHub Actions 5종 (release, docker-build, deploy-docs, update-homebrew, check-bindings)

---

## 3. 폴더 구조 해부

### `backend-go/internal/` — 32개 패키지 (코드량 순)

| 패키지 | 코드량 | 역할 |
|---|---|---|
| `autopilot/` | 97,637줄 / 254파일 | 자동 라우팅 두뇌. 요청 분석 → 모델/채널 선택 → 학습 |
| `handlers/` | 87,685줄 / 248파일 | 프록시 엔드포인트 + 관리 API 구현 |
| `config/` | 38,611줄 / 115파일 | 채널·키·모델매핑 설정 + 핫리로드 |
| `converters/` | 17,405줄 / 47파일 | 프로토콜 변환기 (전방향, 스트리밍 포함) |
| `providers/` | 12,918줄 / 37파일 | 업스트림별 요청 조립/응답 파싱 |
| `metrics/` | 11,036줄 / 31파일 | 요청 기록·통계·비용 집계 (SQLite) |
| `utils/` | 8,468줄 | 공용 유틸 |
| `scheduler/` | 7,269줄 | 우선순위·프로모션·서킷브레이커·페일오버 |
| `healthcheck/` | 3,718줄 | L1(모델목록) / L2(최저가 실호출) 생존 확인 |
| `presetstore/` | 2,876줄 | 채널 프리셋 배포 |
| `ratelimit/` | 2,448줄 | RPM / 동시성 제한 |
| `conversation/` | 2,294줄 | 대화 추적 |
| `quota/` | 2,264줄 | 쿼터 버킷 / 헤더 파싱 |
| `thinkingcache/` | 1,734줄 | 추론 블록 캐시 |
| `compression/` | 1,651줄 | 응답 압축 |
| `guardrails/` | 1,002줄 | 안전장치 |
| `keypool/` | 768줄 | API 키 로테이션 / 블랙리스트 |
| `session/` | 714줄 | Responses 멀티턴 세션 고정(sticky) |
| `racing/` | 546줄 | 경쟁 요청(hedging) 게이트 |

### 나머지 주요 폴더

| 경로 | 역할 |
|---|---|
| `frontend/` | 관리 웹 UI. 화면 7개, 컴포넌트 39개, 로케일 `en`/`zh-CN`/`id` (한국어 없음) |
| `desktop/` | Wails 3 트레이 앱. `internal/backend/manager.go` 가 `PORT..PORT+20` 스캔해서 기존 백엔드에 attach |
| `shared/` | 데이터 자산 (모델 레지스트리, 채널 프리셋, 구독 프리셋, 모델 우선순위) |
| `docs/` | VitePress 문서 사이트 (한/영), 프로바이더별 셋업 가이드 9종 |
| `.claude/skills/` | **이 저장소 개발용** Claude Code 스킬 5종: `model-update`, `store-update`, `upstream-check`, `version-bump`, `github-release` |
| `.github/workflows/` | CI 5종 |
| `packaging/homebrew` | Homebrew tap 템플릿 |
| `scripts/benchmark-sources` | 벤치마크 데이터 수집 스크립트 |

### 프론트엔드 화면 7개

| 화면 | 역할 |
|---|---|
| `ChannelsView` | 채널 CRUD, 드래그 우선순위 조정 |
| `AutopilotView` | 자동 라우팅 모드 / 추천 / Trace 열람 |
| `CockpitView` | 종합 관제탑 |
| `HealthCenterView` | 채널 건강 상태 |
| `CostReportView` | 비용 리포트 |
| `ConversationsView` | 대화별 채널 이동 경로 |
| `SubscriptionsView` | 구독 / 토큰플랜 관리 |

### `shared/` 데이터 자산 (재활용 가치 높음)

| 파일 | 내용 |
|---|---|
| `model-registry/ccx_model_registry.json` | **367KB.** 모델 98종의 컨텍스트 윈도우 / 최대출력 / 가격(캐시 히트·미스 구분) / 능력(vision·tool·reasoning·webSearch) + 벤치마크 프로필 68종 + 이미지 아레나 5종 |
| `channel-presets/*.json` | `claude-messages`, `openai-chat`, `openai-messages`, `codex-responses` 프리셋 |
| `subscription-preset.json` | 출처 등급 체계 (official_api / official_token_plan / relay / community / local_runtime) × 과금모드 (token_plan / pay_as_you_go / shared_free) |
| `capability-probe-schema.json` | 능력 탐지 스키마 |

레지스트리 프로바이더 분포: openai 17, anthropic 14, moonshot 9, dashscope 8, google 8, zai 6, volcengine 6, deepseek 5, minimax 4, agnes 4, xai 3, meta 3, xiaomi 2, 그 외 9종 각 1 — 총 **22개 프로바이더**.

---

## 4. 6개 채널 타입 (핵심 개념)

CCX는 업스트림을 6종으로 분류해 각각 별도 풀로 관리한다.

| 채널 | 프록시 경로 | 용도 |
|---|---|---|
| `messages` | `POST /v1/messages`, `POST /v1/messages/count_tokens` | Claude Code, Claude 계열 |
| `chat` | `POST /v1/chat/completions` | OpenAI 호환 전부 |
| `responses` | `POST /v1/responses`, `GET /v1/responses`(WS), `POST /v1/responses/compact` | Codex CLI 프로토콜 |
| `gemini` | `POST /v1beta/models/{model}:generateContent` / `:streamGenerateContent` | Gemini 네이티브 |
| `images` | `POST /v1/images/generations` `/edits` `/variations` | 이미지 생성 |
| `vectors` | `POST /v1/embeddings` | 임베딩 |

부가 엔드포인트: `GET /health`, `GET /v1/models`, `GET /v1/models/:model`, `POST /v1/alpha/*`(Codex 메모리 레이어 투명 전달).
관리 API는 전부 `/api/{타입}/channels/*` 로 대칭. 실제 라우팅의 사실 소스는 `backend-go/main.go` (2,090줄).

> 주의: capability-test 는 `messages`, `chat`, `responses`, `gemini` 에만 적용된다 (`images`, `vectors` 는 없음).
> L2 헬스체크(실호출)도 같은 4종만. 헬스체크 주기는 **최소 30분 하드 제한**.

---

## 5. Autopilot — 이 프로젝트의 본질

설계 문서: `docs/specs/autopilot.md`, `docs/design/channel-autopilot.md`

```text
[Client Request]
   ↓ 민감정보 제거 후 특징만 추출
[RequestProfile]  Model, Kind, HasImage, HasDocument, EstTokens,
                  QualityNeed, ContextNeed, Effort(off..ultra 8단계),
                  TaskClass, Domain
   ↓
[SmartRouter]  ←  Model Registry (capability / effort / context)
   ↓ 정렬된 후보 채널 목록
[Scheduler]    ←  Healthcheck (L1/L2 상태)
   ↓
[EndpointPolicy]  URL/Key 단위 재정렬·필터링
   ↓
[Upstream Call]
   ↓
[Trace Store + Learning Loop]  성공/실패 피드백으로 프로파일 갱신
```

### 눈에 띄는 세부 기능

| 파일 | 기능 |
|---|---|
| `rate_limit_discovery.go` | 429 응답 / 헤더 / TTFB 신호로 **업스트림의 실제 RPM·최대동시성을 스스로 학습** |
| `racing/` (+ `registry.go`) | p90 지연 초과 시 후보 채널에 **그림자 요청 병렬 발사**, 먼저 온 유효 응답 채택. kiro.rs 방식 참고. claim-once 게이트를 atomic 으로 락프리 구현 |
| `auto_discovery.go` | `/v1/models` 수집 → 모델 목록 자동 등록, 체크포인트로 중단 재개 |
| `protocol_discovery.go` | 같은 URL이 messages / chat / responses 중 무엇을 지원하는지 탐지 |
| `fast_decay.go` | 무료·임시 채널 점수를 빠르게 감쇠 |
| `local_model_runtime.go` | **Ollama / LM Studio / llama.cpp server / OpenAI 호환** 로컬 런타임 자동 탐지 + 상태 추적 (healthy/slow/unavailable/unknown) |
| `provider_quality_probe.go` | 고정 canary 로 프로바이더 품질 L3 탐지 |
| `newapi_*` | new-api 게이트웨이 연동 및 구독 동기화 |
| `request_correlation.go` | 주/그림자/페일오버 시도를 같은 correlation ID로 묶어 실사용자 요청수 대비 증폭 배수 관측 |

라우팅 모드: `assist`(추천만) / `auto`(하드 제약 필터 + 재정렬, fail-open 폴백).

---

## 6. 언제 쓰는가

| 상황 | CCX의 해법 |
|---|---|
| Claude Pro/Max 한도 초과 | 여러 계정·제공사 키를 채널로 등록해 자동 전환 |
| Claude API 비용 부담 | 저가 호환 모델(DeepSeek/GLM/Kimi)을 Claude 형식으로 변환해 그대로 사용 |
| 업스트림 5xx/429 빈발 | 서킷브레이커 + 자동 페일오버 + 키 로테이션 |
| 사용량·비용 파악 불가 | CostReport 로 채널·모델·키별 집계 |
| 쉬운 작업에 비싼 모델 낭비 | Autopilot 이 난이도 판단해 저가 모델로 라우팅 |
| 팀에 키 배포가 부담 | `PROXY_ACCESS_KEY` 하나만 배포, 상위 키는 CCX 내부에 은닉 |
| 사내 규정상 로컬 모델 필요 | Ollama / LM Studio 를 채널로 등록 |

---

## 7. 설치 및 사용법

### 설치 4가지

**① CCX Desktop (권장)**

| OS | 방법 |
|---|---|
| Windows | Microsoft Store 에서 "CCX Desktop" 검색 (자동 업데이트) 또는 Releases 의 `setup.exe` |
| macOS | `brew tap BenedictKing/ccx && brew install --cask ccx-desktop` 또는 `.dmg` |
| Linux | Releases 의 `.AppImage` 에 실행권한 부여 후 실행 |

**② 바이너리 단독**

```bash
# 바이너리 옆에 .env
PROXY_ACCESS_KEY=내가-정한-강한-비밀키
PORT=3000
ENABLE_WEB_UI=true
APP_UI_LANGUAGE=en
```

**③ Docker**

```bash
docker run -d --name ccx -p 3000:3000 \
  -e PROXY_ACCESS_KEY=내-비밀키 \
  -v $(pwd)/.config:/app/.config \
  crpi-i19l8zl0ugidq97v.cn-hangzhou.personal.cr.aliyuncs.com/bene/ccx:latest

# 또는
docker compose up -d
docker compose -f docker-compose.yml -f docker-compose.watchtower.yml up -d   # 자동 업데이트
```

> 기본 이미지 레지스트리가 알리바바 클라우드(중국)다. 네트워크 환경에 따라 막힐 수 있으니 그럴 때는 소스 빌드를 쓴다.

**④ 소스 빌드** (필요: Go 1.25+, Bun, Make)

```bash
cp backend-go/.env.example backend-go/.env
make install        # 프론트 + Go 모듈 + 개발도구
make run            # 프론트 빌드 후 Go 실행
make dev            # 개발: Vite HMR + air 핫리로드 동시
make build
make frontend-dev
```

### 사용 흐름

1. CCX 실행 → `http://localhost:3000` 웹UI
2. Messages 탭 → Add Channel → 서비스타입 `Claude`, Base URL, 업스트림 API Key
3. 모델 매핑: `opus` → 상위 모델, `sonnet` → 주력 모델, `haiku` → 경량 모델 (CCX는 **더 긴 키를 먼저 매칭**)
4. 클라이언트 환경변수 교체 (`ANTHROPIC_BASE_URL` 에 `/v1` 붙이지 않음)
5. `claude "안녕"`

업스트림 Base URL 예시:

| 업스트림 | 서비스 타입 | Base URL |
|---|---|---|
| Anthropic Claude | `Claude` | `https://api.anthropic.com` |
| DeepSeek (Anthropic 호환) | `Claude` | `https://api.deepseek.com/anthropic` |
| Kimi coding | `Claude` | `https://api.kimi.com/coding/` |
| GLM (Anthropic 호환) | `Claude` | `https://open.bigmodel.cn/api/anthropic` |

### 운영 주의사항

- `.config/config.json` 은 **핫리로드**, `.env` 는 **재시작 필요**
- Windows 에서 `localhost` 로 안 닿으면 호스트 IPv4 사용. `BIND_HOST` 가 비면 전 인터페이스 바인딩, 로컬 전용은 `BIND_HOST=127.0.0.1`
- 관리 권한 분리는 `ADMIN_ACCESS_KEY`
- 쿼리스트링 `?key=` 인증은 로그 노출 위험 → 프로덕션 비권장
- 실행 중 서비스 프로세스를 임의로 죽이지 않는 것이 이 저장소의 협업 규칙(`AGENTS.md`)

---

## 8. 플러그인? 스킬? MCP? — 전부 아니다

CCX는 **독립 실행되는 서버 프로그램 (리버스 프록시 / API 게이트웨이)** 다.

| 구분 | 해당? | 설명 |
|---|---|---|
| 플러그인 | 아니오 | Claude Code 에 설치하는 것이 아니라 별도 프로세스 |
| 스킬(Skill) | 아니오 | 스킬은 Claude 에게 주는 지침서. CCX는 네트워크 계층 |
| MCP 서버 | 아니오 | MCP는 "Claude 에게 도구를 추가". CCX는 "Claude 가 말 거는 상대를 교체" |
| HTTP 프록시 / 게이트웨이 | **예** | `ANTHROPIC_BASE_URL` 교체로 끼어든다 |

```text
MCP:  Claude ──(도구 호출)──▶ MCP 서버        "할 수 있는 일을 늘린다"
CCX:  Claude Code ──(모든 요청)──▶ CCX ──▶ 실제 AI   "대화 상대를 바꾼다"
```

혼동 주의: 저장소의 `.claude/skills/` 는 **CCX를 개발할 때 쓰는 Claude Code 스킬 5종**이며 제품 기능이 아니다.
그리고 MCP와 **동시 사용 가능**하다 (계층이 달라 서로 간섭하지 않는다).

---

## 9. API 토큰 구조

```text
[Claude Code] ──① PROXY_ACCESS_KEY──▶ [CCX] ──② 실제 업스트림 키──▶ [AI 업체]
                  (내가 임의로 정함)              (유료 발급)
```

| 구분 | 이름 | 발급자 | 비고 |
|---|---|---|---|
| 하류(내부) | `PROXY_ACCESS_KEY` | 내가 임의 문자열로 정함 | 무료. 외부인 차단용 문지기 |
| 관리용 | `ADMIN_ACCESS_KEY` | 내가 정함 (선택) | 웹UI/관리API 전용 |
| 상류(외부) | 각 업체 API Key | 업체 발급 (대개 유료) | Anthropic, OpenAI, DeepSeek, Kimi, GLM 등 |

**CCX는 토큰을 무료로 만들어주는 도구가 아니다.** 이미 가진 키/계정을 효율적으로 돌려쓰게 해주는 것이다.
비용을 실제로 0 에 가깝게 하려면: 로컬 모델(Ollama/LM Studio), 각 업체 무료 티어 로테이션, 저가 코딩 플랜 활용.

보안:
- `PROXY_ACCESS_KEY` 는 충분히 길게. 노출되면 내 상위 키로 타인이 과금한다
- 외부 공개 시 `BIND_HOST=127.0.0.1` + 터널링(Tailscale 등) 권장
- `.env`, `.config/config.json` 은 커밋 금지 (`.gitignore` 에 이미 포함)
- 로그 기록 시 API Key / Authorization / multipart 본문 마스킹 필수

---

## 10. 왜 GitHub 에서 인기 있는가 (코드·문서 근거 기반 추론)

> 본 세션의 GitHub 접근 범위가 `bmshin94/ccx` 로 제한되어 업스트림의 실시간 스타 수는 확인하지 못했다.
> 아래는 저장소 내부 증거에 기반한 추론이다.

1. **통증 정확 타격** — "Claude Code 가 비싸고 한도가 금방 찬다"는 2025~2026 최대 불만을 정면으로 해결
2. **배포 난이도 0** — Go 단일 바이너리 + 웹UI 내장, Docker / Store 설치까지
3. **프로토콜 변환 커버리지** — Claude / OpenAI Chat / Codex Responses / Gemini **전방향 + 스트리밍**. 경쟁 프로젝트는 대개 단방향
4. **중국 AI 생태계 완전 커버** — DeepSeek, Kimi, GLM, MiniMax, Qwen, 볼케이노엔진, MiMo 등. 중국어 README + QQ 그룹(642217364) 운영
5. **완성도** — CHANGELOG 210KB, Go 테스트 473개, 한/영 문서 79개, 스폰서 3곳(BytePlus·Youyun·RunAPI), SignPath 무료 코드사이닝, Actions 5종
6. **Autopilot 차별점** — 경쟁자(one-api / new-api / LiteLLM)는 라운드로빈+우선순위 수준. CCX는 요청 내용 기반 라우팅 + 실패 학습 + hedging
7. **데스크톱 앱** — Wails 3 트레이 앱 + Microsoft Store 등록으로 비개발자 진입 가능
8. **MIT 라이선스** — 기업 도입 / 포크 / 상업화 자유

### 리스크

- 코드베이스 규모가 매우 큼(autopilot 97K줄) → 기여 진입장벽, 버스 팩터 우려
- 주 문서·커밋·주석이 중국어 → 글로벌 기여자 유입 장벽
- "Claude Code 에 타 모델 주입"은 제공사 ToS 회색지대. 특히 **구독 계정 우회는 위험**
- 업스트림 프로토콜 변경 추격이 상시 부담 (`TODO.md` 에 gpt-5.6 대응, Codex 신기능 등 흔적)

---

## 11. 로컬 에이전트 구축에 도움이 되는가 — 된다 (역할 구분 전제)

### 도움이 되는 부분

**로컬 런타임 지원이 이미 내장**
`backend-go/internal/autopilot/local_model_runtime.go`:
`ollama`, `lmstudio`, `llama_server`, `openai_compatible` 타입 + 상태 추적.
Ollama 를 채널로 등록하면 "로컬 우선, 어려운 건 클라우드" 하이브리드 라우팅이 즉시 가능하다.

**권장 하이브리드 구성**

```text
내 에이전트 코드 (LangGraph / 자작)
   │  ← SDK 하나로만 개발
   ▼
  CCX
   ├─ 쉬운 일 (요약/분류) → Ollama  (무료, 프라이버시)
   ├─ 중간 일 (코딩)      → DeepSeek (저가)
   └─ 어려운 일 (설계)    → Claude Opus
```

**에이전트 운영의 실전 난제를 이미 해결해 둔 목록**

| 에이전트 개발 시 필연적 문제 | CCX의 해답 |
|---|---|
| 모델 장애로 루프 전체 실패 | 서킷브레이커 + 자동 페일오버 |
| 429 폭탄 | RPM 자동 학습 + 키풀 로테이션 |
| 비용 폭발 | 요청별 비용 기록 + CostReport |
| 어느 단계가 느린지 모름 | Trace + 지연 프로파일 |
| 멀티턴 상태 유지 | Responses 세션 sticky 라우팅 |
| 컨텍스트 초과 | 실제 컨텍스트 윈도우로 후보 필터링 |
| long-tail latency | racing (hedging) |

### CCX가 하지 않는 일

- 툴/함수 호출 오케스트레이션 (LangGraph, Agents SDK 영역)
- 계획 수립, ReAct 루프, 멀티 에이전트 협업
- 벡터 DB 관리 (embedding **프록시**는 하지만 저장소는 아님)
- 메모리 / RAG 파이프라인
- 프롬프트 템플릿 관리

### 권장 스택

```text
[UI]        React / Next.js
[에이전트]   LangGraph.js 또는 OpenAI Agents SDK
[게이트웨이] CCX          ← 여기를 CCX가 담당
[모델]      Ollama(로컬) + DeepSeek + Claude
[벡터]      Qdrant / pgvector
```

결론: CCX는 "에이전트의 두뇌 배선"이 아니라 **"에이전트의 전력·통신 인프라"** 로 쓴다.

---

## 12. React / PHP 로 만들 수 있는가

### 프론트엔드 (Vue → React): 쉽다

관리 UI는 REST API 를 호출하는 대시보드일 뿐이고 Vue 고유 기능에 의존하지 않는다.

| 현재 | React 대체 |
|---|---|
| Vue 3 + TS | React 18 + TS |
| Vuetify 3 | MUI / shadcn-ui (컴포넌트 39개 상당) |
| Pinia | Zustand + TanStack Query (서버 상태는 TanStack 이 유리) |
| vue-i18n | react-i18next (로케일 JSON 재사용 가능) |
| `services/api.ts` | 거의 그대로 이식 가능 |
| Vite | 동일 |

가장 가성비 높은 선택: **백엔드는 CCX 그대로 두고 React 대시보드만 새로 붙인다** (`/api/*` 가 이미 깔끔한 REST). 2~4주 규모.

### 백엔드 (Go → PHP): 가능하지만 비권장

| 요구사항 | Go | PHP (전통 FPM) |
|---|---|---|
| SSE 스트리밍 프록시 (핵심) | goroutine 으로 자연스럽게 | 워커 1개 점유. 동시 100명 = 워커 100개 |
| 장시간 연결(수 분) | 문제없음 | `max_execution_time` 제약 |
| 경쟁 요청 병렬 발사 | `sync/atomic` + context | 프로세스 모델과 부적합 |
| 백그라운드 워커 (헬스체크/탐지) | 내장 goroutine | 별도 크론/큐 필요 |
| 프로세스 간 상태 공유 (서킷브레이커/키풀) | 메모리 그대로 | Redis 필수 |
| WebSocket (`GET /v1/responses`) | 기본 지원 | Swoole / Ratchet 필요 |
| 배포 | 단일 바이너리 | PHP + FPM + Nginx + Redis |

PHP 로 강행한다면 `Swoole` 또는 `RoadRunner`(Laravel Octane) 같은 상주형 런타임이 필수다.
그 경우 "PHP 를 Go 처럼 쓰는" 형태가 되어 PHP 를 택한 이유(익숙함·배포 단순함)가 상쇄된다.

### 현실적 추천 순서

| 순위 | 방안 | 기간 |
|---|---|---|
| 1 | 백엔드 CCX 유지 + React 커스텀 UI | 2~4주 |
| 2 | TypeScript(Bun + Hono) 로 "미니 CCX" 직접 구현 | 1~2개월 (MVP) |
| 3 | PHP + Swoole 재작성 | 3~6개월 |
| 비권장 | PHP-FPM 그대로 전체 재작성 | 스트리밍에서 무너진다 |

### "미니 CCX (TypeScript)" MVP 범위

```text
POST /v1/messages
 1. PROXY_ACCESS_KEY 인증
 2. 채널 우선순위 정렬
 3. 모델 매핑 적용 (opus → deepseek-reasoner)
 4. 프로토콜 변환 (Claude → OpenAI)   ← 난이도 최상
 5. fetch() 업스트림 호출
 6. SSE 스트림 릴레이 (ReadableStream 변환)
 7. 실패 시 다음 채널 재시도
 8. SQLite 요청 기록
```

제외할 것: Autopilot 학습, racing, 자동 탐지, 구독 관리, 데스크톱 앱.
`shared/model-registry/ccx_model_registry.json` 은 MIT 라 그대로 재사용 가능 (원본 데이터 출처 표기 의무 준수).

---

## 13. 수익화 아이디어

### 먼저: 금지 영역

| 하면 안 되는 것 | 이유 |
|---|---|
| Claude Pro/Max **구독 계정**을 프록시로 감싸 API 재판매 | 대부분 제공사 ToS 위반. 계정 정지 + 환불 불가 + 법적 리스크 |
| 타인 명의 무료 계정 대량 생성 후 로테이션 | 약관 위반 / 사기 소지 |
| 업스트림 키 공유로 "무제한 AI" 광고 | 신고 시 서비스 중단 |
| 고객 API 키를 무책임하게 대리 보관 | 사고 시 전적 책임 (개인정보·보안 규제) |

안전선: **"고객이 자기 키를 넣고, 우리는 도구·운영·최적화를 판다"** 또는 **정식 리셀러 계약**.

### 아이디어 요약

| 방향 | 한 줄 | 난이도 | 수익성 |
|---|---|---|---|
| A. 매니지드 호스팅 | 설치 귀찮은 사람용 클라우드 CCX | 중 | 높음 |
| B. Enterprise Pro 애드온 | SSO / 감사로그 / 부서 예산 | 중상 | 매우 높음 |
| C. 비용 최적화 컨설팅 | "AI 청구서 40% 절감" | 낮음 | 높음 |
| D. 모델 데이터 서비스 | 가격·벤치마크 API | 중 | 보통 |
| E. 교육 콘텐츠 | 강의 / 전자책 / 유튜브 | 낮음 | 보통 |
| F. 수직 SaaS | 업종 특화 게이트웨이 | 높음 | 매우 높음 |

### A. 매니지드 CCX 호스팅 (SaaS)

- 타겟: CCX 를 쓰고 싶지만 서버 세팅·도커·업데이트가 부담인 개발자/1인 창업자
- 가치: 5분 온보딩, 항상 최신 버전(프로토콜 변경 추격), 백업·모니터링·알림, 고정 IP

| 플랜 | 월 | 포함 |
|---|---|---|
| Free | 0원 | 1채널, 10K req/월, 로그 7일 |
| Solo | 12,000원 | 5채널, 무제한 req, 로그 30일, Autopilot |
| Team | 49,000원 | 무제한 채널, 5석, 로그 90일, 예산 알림 |
| Business | 190,000원 | SSO, 감사로그, SLA, 전용 IP |

핵심: **키는 고객 소유**, 우리는 인프라만 판매 → ToS 리스크 회피.
원가는 Go 라 메모리 소비가 적어 VPS 1대에 멀티테넌트 수십 계정 가능 (마진 80%+).
난관: 키 저장 보안(KMS 봉투암호화 필수), 신뢰 확보, "셀프호스팅하면 되는데" 반론 → 업데이트 추격 + 지원으로 방어.

### B. Enterprise Pro 애드온 (오픈코어) — 최우선 추천

본체는 MIT 로 유지하고 기업용 기능만 상용 모듈로 판매한다.

| 기능 | 구매 동기 |
|---|---|
| SSO / SAML / OIDC | 대기업 필수 요건 |
| RBAC (부서·팀·개인) | "마케팅팀은 GPT만, 개발팀은 Claude" |
| 부서별 예산 한도 + 초과 차단 | 재무 담당자의 1순위 요구 |
| 감사 로그 (변조 방지·장기 보관) | ISMS-P / SOC2 심사 |
| PII 마스킹 / DLP | 민감정보 업스트림 유출 차단 |
| 차지백 리포트 (CSV/Excel) | 부서별 비용 정산 |
| 온프레미스 라이선스 + 기술지원 SLA | 금융·의료·공공 |
| 프롬프트 정책 엔진 | 규제 산업 |

- 가격: 연 500만원 ~ 5,000만원 (좌석수·트래픽 기준)
- 강점: `metrics` / `quota` / `guardrails` / `keypool` 기반이 이미 있어 **위에 얹기만** 하면 된다
- 난관: 엔터프라이즈 영업 사이클 6~12개월. 첫 레퍼런스 확보가 관건

### C. AI 비용 최적화 컨설팅 — 가장 빠른 현금화

```text
[진단 패키지]  300만원 / 2주
  CCX 설치 + 전 트래픽 계측 → 2주 실데이터 → 낭비 지점 리포트 → 절감 시뮬레이션(dry-run)
  예) "이 요약 기능은 Opus → Haiku 로 충분. 월 340만원 절감"
      "캐시 히트율 8% → 프롬프트 구조 개선 시 60% 가능"

[구축 패키지]  800만 ~ 2,000만원
  Autopilot 라우팅 정책 설계 + 페일오버 + 대시보드 + 교육

[운영 리테이너]  월 150만 ~ 300만원
  모니터링, 신규 모델 출시 시 재평가, 정책 튜닝
```

성과보수형 제안: **"절감액의 20%를 3개월간. 못 줄이면 0원."** — 고객 리스크가 0 이라 거절하기 어렵다.
장점: 코드 자산 없이 지식만으로 즉시 시작 가능.
단점: 노동집약적이라 확장성 낮음 → 사례 3~5건 후 B 또는 A 로 전환하는 것이 정석.

### D. 모델 인텔리전스 데이터 서비스

`shared/model-registry/ccx_model_registry.json` 자체가 자산이다 (모델 98종 × 가격/컨텍스트/최대출력/캐시가격/능력 + 벤치마크 68종).

```text
GET /v1/models/pricing
GET /v1/models/compare?task=coding&budget=low
GET /v1/models/changes?since=2026-09-01     ← 가격 변동 알림
```

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0 | 일 100 req, 주간 갱신 |
| Dev | $19/월 | 무제한, 일간 갱신, 웹훅 |
| Business | $199/월 | 실시간, 히스토리, 벤치마크 raw |

부가: 모델 가격 비교 사이트 → SEO 트래픽 → 광고 + 어필리에이트.
(CCX README 의 RunAPI·BytePlus 어필리에이트 링크가 이 모델의 실제 작동 증거다.)
난관: 데이터 신선도 유지(자동 수집 파이프라인 필수), Artificial Analysis 등 **원본 출처 표기 의무 준수**.

### E. 교육 콘텐츠

| 상품 | 가격 |
|---|---|
| 유튜브 "AI API 비용 90% 줄이기" | 광고 + 협찬 (리드 유입) |
| 강의 "Go 로 LLM 게이트웨이 만들기" | 99,000원 |
| 전자책 "실전 멀티 LLM 아키텍처" | 29,000원 |
| 온라인 워크숍 (4주, 15명) | 490,000원/인 |
| 기업 출강 (AI 인프라 1일) | 2,000,000원/회 |

강점: "멀티 LLM 게이트웨이"는 수요는 큰데 **한국어 자료가 희소**하고, 토이 예제가 아닌 **실제 프로덕션 코드**를 교재로 쓸 수 있다.
시너지: 교육 → 신뢰 → 컨설팅(C) → SaaS(A). 깔때기 최상단.

### F. 수직(버티컬) SaaS — 장기 최대 잠재력

CCX 를 엔진으로 숨기고 업종 전용 제품을 만든다.

| 예시 | 구성 | 가격대 |
|---|---|---|
| 법률사무소용 | 민감문서는 로컬 Ollama 강제, 판례검색은 클라우드, 사건번호별 비용 추적, 전체 감사로그 | 사무소당 월 30만 ~ 150만원 |
| 병원/의료용 | PHI 자동 마스킹, 온프레미스 요건 충족 | 병원당 연 3,000만원+ |
| 게임사 NPC 대화 | 캐릭터별 모델/프롬프트 프리셋, 동접 폭주 시 저가 모델 폴백(racing), 응답 필터(guardrails) | 타이틀당 MAU 과금 |

강점: 범용 게이트웨이 시장은 이미 경쟁 심함(LiteLLM, OpenRouter, Portkey)이지만 버티컬은 경쟁자가 적고 컴플라이언스 요구가 진입장벽이 되어 방어막이 된다.
난관: 도메인 전문성 필요 → 업종 파트너 확보가 핵심 (기술은 내가, 영업·도메인은 파트너).

### 로드맵

| 기간 | 할 일 | 목표 |
|---|---|---|
| 0~3개월 | E. 교육 콘텐츠로 인지도 확보 + CCX 숙련 | 월 50~200만원 |
| 3~9개월 | C. 컨설팅 2~3건 (성과보수 모델) | 건당 500~2,000만원 |
| 9~18개월 | B. 컨설팅에서 반복된 요구를 Pro 모듈로 제품화. 첫 고객은 컨설팅 고객 중에서 | 연 5,000만원+ |
| 18개월+ | A. 또는 F. 로 확장 | ARR 억 단위 |

원칙 3가지
1. 코드보다 고객 검증이 먼저다 — 컨설팅으로 "돈 낼 문제"를 확인한 뒤 제품을 만든다
2. 본체는 MIT 로 기여한다 — 업스트림 PR 이력이 곧 신뢰이자 영업력
3. ToS 선을 넘지 않는다 — 구독 재판매 유혹을 참는 것이 지속 가능성의 조건

---

## 14. 한 줄 요약

> **CCX = Claude Code 의 주소 한 줄만 바꾸면, 여러 AI 제공사를 자동으로 번역·전환·최적화해서 쓰게 해주는 나만의 중간 게이트웨이.**
> 그리고 그 안에는 Go 로 LLM 인프라를 만드는 방법에 대한 10만 줄짜리 정답지가 들어 있다.

## 참고 링크

- 내 저장소: https://github.com/bmshin94/ccx
- 업스트림 원본: https://github.com/BenedictKing/ccx
- 릴리스: https://github.com/BenedictKing/ccx/releases/latest
- 내부 문서: `docs/guide/architecture.md`, `docs/guide/development.md`, `docs/guide/environment.md`, `docs/specs/autopilot.md`, `docs/design/channel-autopilot.md`
- 클라이언트 연결 가이드: `docs/en/guide/clients/` (claude-code / codex / opencode / claude-desktop)
- 협업 규칙: `AGENTS.md`, `CLAUDE.md`
