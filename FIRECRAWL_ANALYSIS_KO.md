# Firecrawl 전수조사 및 활용 가이드 (한국어 정리)

> 이 문서는 `bmshin94/firecrawl` 저장소를 폴더 단위로 전수조사한 결과와,
> 설치·활용·수익화에 대한 질의응답을 정리한 기록입니다.
> 작성일: 2026-10-03

## 관련 GitHub 주소

| 대상 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/firecrawl |
| 원본 저장소 | https://github.com/firecrawl/firecrawl |
| 스킬 카탈로그 (읽기 전용) | https://github.com/firecrawl/skills |
| CLI | https://github.com/firecrawl/cli |
| CLI 스킬 | https://github.com/firecrawl/cli/tree/main/skills |
| 워크플로 스킬 | https://github.com/firecrawl/firecrawl-workflows |
| MCP 서버 | https://github.com/firecrawl/firecrawl-mcp-server |
| Go SDK (서브모듈) | https://github.com/firecrawl/firecrawl-go |
| Go SDK 예제 (서브모듈) | https://github.com/firecrawl/firecrawl-go-examples |
| 공식 문서 | https://docs.firecrawl.dev |
| 서비스 / 플레이그라운드 | https://firecrawl.dev · https://firecrawl.dev/playground |

---

## 1. 한 줄 정의

**웹사이트를 AI가 바로 사용할 수 있는 마크다운·JSON으로 변환하는 웹 데이터 API 서버의 오픈소스 전체 소스코드.**

- 라이선스: 본체 API는 **AGPL-3.0**, SDK와 일부 UI는 **MIT**
- 기술 스택: TypeScript(API) + Go(HTML→Markdown) + Rust/WASI 네이티브 모듈 + Playwright + ClickHouse

## 2. 해결하는 문제

| 장벽 | 실제 증상 |
| --- | --- |
| JavaScript 렌더링 | `curl`로 받으면 빈 `<div id="root">`만 나옴 |
| 봇 차단 | Cloudflare·DataDome이 403 반환 |
| IP 차단 | 동일 IP 반복 요청 시 밴 |
| HTML 노이즈 | 광고·내비·푸터가 본문의 90% → 토큰 낭비 |
| 문서 파싱 | 웹에 올라간 PDF·DOCX는 별도 작업 필요 |

Firecrawl은 위 5가지를 "URL 하나 → 깨끗한 마크다운" 수준으로 추상화했다.

## 3. 저장소 구조 (전수조사 결과)

```
firecrawl/
├── apps/                          실제 제품 코드 (20개)
│   ├── api/              17MB   ★ 핵심: API 서버 + 다수 워커 (TypeScript)
│   ├── test-site/        14MB     테스트용 더미 웹사이트
│   ├── test-suite/       1.8MB    통합 테스트
│   ├── playwright-service-ts      셀프호스팅용 브라우저 렌더링
│   ├── go-html-to-md-service      HTML→마크다운 변환 (Go, 성능 목적 분리)
│   ├── nuq-postgres / redis       작업 큐·캐시 인프라
│   ├── siem                       보안 로그
│   ├── ui/ingestion-ui            React 18 + Vite + Tailwind + Radix 데모 UI
│   └── *-sdk (9개)                python js go java rust ruby php dot-net elixir
├── skills/ (5개)                ★ Claude Code용 Agent Skill 원본 (build 계열)
├── examples/ (58개)             ★ 실전 예제 — 사실상 검증된 유스케이스 목록
├── firecrawl-cli/                 README만 있는 포인터 (실 소스는 firecrawl/cli)
├── firecrawl-skills/              포인터
├── firecrawl-cli-skills/          포인터
├── firecrawl-workflows/           포인터
├── docker-compose.yaml            셀프호스팅 스택
├── SELF_HOST.md                   셀프호스팅 가이드 (경고 다수)
└── LICENSE                        AGPL-3.0
```

### 함정 1 — 루트의 4개 폴더는 빈 껍데기

`firecrawl-cli/`, `firecrawl-skills/`, `firecrawl-cli-skills/`, `firecrawl-workflows/`
안에는 README.md 하나씩만 있다. 전부 다른 저장소를 가리키는 안내판이다.

### 함정 2 — `apps/api`는 완전체가 아니다

```
apps/api/src/scraper/scrapeURL/engines/
├── fire-engine/    ← 상용 비공개 엔진 (봇 우회·프록시 담당)  [소스 없음]
├── playwright/     ← 오픈소스 폴백 (브라우저 렌더링)
├── fetch/          ← 단순 HTTP
├── pdf/ document/ image/
└── wikipedia/ x-twitter/   사이트 전용 특수 처리
```

봇 차단 우회의 핵심인 `fire-engine`은 **공개되지 않은 유료 서비스**다.
셀프호스팅하면 Playwright + fetch만 동작하므로 Cloudflare 등 방어된 사이트는 불가능하다.
README의 "96% 커버리지"는 유료 클라우드 기준이다.
(테스트 코드도 `!process.env.TEST_SUITE_SELF_HOSTED` 로 fire-engine 의존 테스트를 스킵한다.)

즉 이 제품은 **오픈코어(open-core)** 구조다.

## 4. 기능 = API 엔드포인트 (`apps/api/src/controllers/v2/`)

| 엔드포인트 | 하는 일 | 쓰는 순간 |
| --- | --- | --- |
| `/scrape` | URL 1개 → 마크다운/HTML/JSON/스크린샷 | "이 페이지 내용 줘" |
| `/search` | 검색어 → 결과 + 각 페이지 본문까지 | "최신 정보 찾아줘" |
| `/crawl` | 사이트 전체 자동 순회 | "문서 사이트 전부 학습" |
| `/map` | 사이트의 전체 URL 목록 즉시 추출 | "페이지 몇 개 있나" |
| `/batch-scrape` | URL 수천 개 비동기 처리 | 대량 수집 |
| `/agent` | 프롬프트만 주면 AI가 검색·탐색·추출 | URL을 몰라도 됨 |
| `/interact` | 스크랩 후 클릭·입력·스크롤 | 로그인·검색창·페이지네이션 |
| `/extract` | 스키마 기반 구조화 추출 (`/agent` 구버전) | 정해진 필드로 뽑기 |
| `/parse` | 업로드한 PDF/DOCX 파싱 | 파일 → 텍스트 |

운영용 엔드포인트: `credit-usage`, `concurrency-check`, `queue-status`,
`crawl-status-ws`(웹소켓 실시간 진행률), `browser-replay`, `agent-trace`.

### 특히 중요한 2개

**`/agent`** — URL을 몰라도 된다. `effort`를 `low/medium/high`로 주면 추론 예산이 바뀐다.
내부 모델은 `spark-2`(현행), `spark-1-mini`/`spark-1-pro`(레거시). `model`과 `effort` 동시 전송은 400.

```python
app.agent(prompt="Find the founders of Firecrawl", schema=FoundersSchema)
```

**`/interact`** — 스크랩한 브라우저 세션을 유지한 채 자연어로 조작한다.
`liveViewUrl`로 실시간 화면도 볼 수 있다.

```python
result = app.scrape("https://amazon.com")
app.interact(result.metadata.scrape_id, prompt="Search for 'mechanical keyboard'")
app.interact(result.metadata.scrape_id, prompt="Click the first result")
```

액션 타입은 Zod로 검증된다: `wait` `click` `screenshot` `write` `press` `scroll`
(+ 총 대기시간 상한, `waitFor <= timeout/2` 같은 refine 규칙).

## 5. `examples/` 58개 — 가장 실용적인 폴더

- **모델별 크롤러 세트**: `o1_web_crawler`, `gpt-4.1-web-crawler`, `claude3.7-web-crawler`,
  `gemini-2.5-crawler`, `deepseek-v3-crawler`, `llama-4-maverick-web-crawler`,
  `grok_web_crawler`, `groq_web_crawler`, `mistral-small-3.1-crawler`
- **비즈니스 유스케이스**: `crm_lead_enrichment`, `sales_web_crawler`, `o1_job_recommender`,
  `job-resource-analyzer`, `o3-mini-deal-finder`, `deep-research-apartment-finder`,
  `claude_stock_analyzer`, `find_internal_link_opportunites`, `internal_link_assistant`
- **콘텐츠 자동화**: `aginews-ai-newsletter`, `ai-podcast-generator`, `blog-articles`
- **RAG / 지식베이스**: `web_data_rag_with_llama3`, `website_qa_with_gemini_caching`,
  `turning_docs_into_api_specs`
- **멀티에이전트 / 실시간**: `openai_swarm_firecrawl`, `openai-realtime-firecrawl`
- **인프라**: `kubernetes/cluster-install/`, `kubernetes/firecrawl-helm/`

→ 수익 모델을 새로 발명할 필요가 없다. 이 58개가 이미 검증된 목록이다.

## 6. `skills/` — Agent Skill 5개

```
skills/
├── firecrawl-build/              라우터 (어느 엔드포인트를 쓸지 결정)
│   └── references/  project-intake, endpoint-selection, integration-patterns,
│                    sdk-installation, auth-and-env, verification
├── firecrawl-build-onboarding/   API 키 발급 + .env + SDK 설치 (references 3개)
├── firecrawl-build-scrape/       (references: freshness-and-liveness)
├── firecrawl-build-search/
└── firecrawl-build-interact/
```

- 각 `SKILL.md`에 `name` / `description` / `inputs` / `references` YAML frontmatter.
- `inputs`: `FIRECRAWL_API_KEY`(필수), `FIRECRAWL_API_URL`(셀프호스팅용, 선택)
- `firecrawl-build`의 description에는 **`"fire girl"` 축약 트리거**가 들어있고,
  사용자가 Firecrawl을 언급하지 않고 "웹 데이터가 필요하다"고만 해도 발동하도록 설계됨.

### 스킬 저장소 역할 분담

| 종류 | 원본 위치 |
| --- | --- |
| build 스킬 (앱 코드에 통합) | **이 저장소 `skills/`** → CI가 카탈로그로 미러링 |
| CLI 스킬 (터미널 즉시 작업) | `firecrawl/cli` |
| 워크플로 스킬 (반복 산출물) | `firecrawl/firecrawl-workflows` |
| 통합 카탈로그 | `firecrawl/skills` — **읽기 전용, 직접 PR 금지** |

SKILL.md 내 상호 링크는 설치 시 평면 디렉터리를 가정해 `../firecrawl-x/SKILL.md` 형태로 유지된다.

## 7. 인프라 (`docker-compose.yaml`)

```
api (3002, 호스트에 유일하게 공개)
 ├─ playwright-service        브라우저 렌더링
 ├─ redis (또는 valkey)        캐시·레이트리밋
 ├─ rabbitmq (3-management)   메시지 큐
 ├─ nuq-postgres              작업 큐 기본값 (pg_cron 사용)
 └─ foundationdb 7.3.63       대안 큐 백엔드 (NUQ_BACKEND=fdb)
```

워커 종류(`apps/api/package.json` scripts): `queue-worker`, `nuq-worker`, `nuq-fdb-worker`,
`nuq-prefetch-worker`, `nuq-reconciler-worker`, `extract-worker`, `index-worker`,
`cclog-worker`, `zdr-worker`.

### SELF_HOST.md가 경고하는 것 (중요)

- 기본 API는 **인증이 없다** (`USE_DB_AUTHENTICATION=false`). 외부 노출 금지.
- 루트 compose에 **영속 볼륨이 없다**. 컨테이너 삭제 시 데이터 소실.
- AI 기능은 **모델 제공자 미연결** 상태. OpenAI / OpenAI 호환 / Ollama를 따로 연결.
- 큐 관리 UI는 꺼져 있고, 켤 때 강한 `BULL_AUTH_KEY` + 네트워크 제한 필요.
- 특정 **릴리스 태그**로 체크아웃할 것. `main` + floating 이미지 태그는 따로 움직인다.
- 루트 `.env`는 compose가 참조하는 변수만 덮어쓴다. `apps/api/.env.example`은 compose 계약이 아니다.
- 쿠버네티스/Helm 예제는 프로덕션 검증 아키텍처가 아니라 "버전 맞춰진 출발점".

## 8. 설치 및 사용법 — 4가지 경로

### 경로 A. 클라우드 API (권장)

```bash
pip install firecrawl-py                   # Python
npm install firecrawl                      # Node
composer require firecrawl/firecrawl-sdk   # PHP
```

```python
from firecrawl import Firecrawl
app = Firecrawl(api_key="fc-YOUR_KEY")

doc     = app.scrape("https://firecrawl.dev")
results = app.search("best AI tools 2026", limit=10)
docs    = app.crawl("https://docs.firecrawl.dev", limit=50)
result  = app.agent(prompt="Find the founders of Stripe")
```

```bash
curl -X POST 'https://api.firecrawl.dev/v2/scrape' \
  -H 'Authorization: Bearer fc-YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"url": "firecrawl.dev"}'
```

설치 전 https://firecrawl.dev/playground 에서 먼저 테스트할 것.

### 경로 B. CLI

```bash
npx -y firecrawl-cli@latest init --all --browser
firecrawl scrape https://firecrawl.dev
firecrawl search "firecrawl" --limit 5
firecrawl interact exec --prompt "Click the first result"
```

### 경로 C. Agent Skill

```bash
npx skills add firecrawl/skills --skill firecrawl-build
# 설치 후 에이전트 재시작
```

에이전트 온보딩 스킬: `curl -s https://firecrawl.dev/agent-onboarding/SKILL.md`

### 경로 D. 셀프호스팅 (무료, 기능 제한)

```bash
git clone https://github.com/bmshin94/firecrawl
cd firecrawl
docker compose up -d        # → localhost:3002
curl -X POST 'http://localhost:3002/v2/scrape' \
  -H 'Content-Type: application/json' -d '{"url":"example.com"}'
```

### 개발 기여 규칙 (CLAUDE.md)

- `pnpm harness jest <경로>` 사용. `pnpm start` 직접 실행 금지.
- E2E(`snips`)를 단위 테스트보다 우선. 스크랩 타임아웃은 `./lib`의 `scrapeTimeout` 사용.
- 테스트 게이팅: fire-engine 필요 → `!process.env.TEST_SUITE_SELF_HOSTED`,
  AI 필요 → `!process.env.TEST_SUITE_SELF_HOSTED || OPENAI_API_KEY || OLLAMA_BASE_URL`
- **`knip` 실패를 `--no-verify`로 우회 금지.** 미사용 export는 기존 것이라도 수정 후 커밋.

## 9. 플러그인인가, 스킬인가, MCP인가

본체는 **API 서버**이고, 그것을 쓰는 포장지가 4종류다.

```
            Firecrawl API (이 저장소, AGPL)
                        ▲
      ┌─────────┬───────┴────────┬────────────┐
   SDK 9개     CLI            MCP 서버      Agent Skill
 (라이브러리)  (터미널)      (별도 repo)   (이 repo skills/)
```

| 구분 | 이 저장소에 있나 | 위치 |
| --- | --- | --- |
| API 서버 | 있음 (본체) | `apps/api/` |
| SDK 9개 | 있음 | `apps/*-sdk/` |
| Agent Skill | 있음 (build 계열 5개) | `skills/` |
| MCP 서버 | 없음 | `firecrawl/firecrawl-mcp-server` |
| CLI | 없음 (포인터만) | `firecrawl/cli` |
| 플러그인 | 해당 개념 없음 | — |

외부 플랫폼 통합: Zapier, n8n, Lovable.

MCP 설정 예:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": { "FIRECRAWL_API_KEY": "fc-YOUR_KEY" }
    }
  }
}
```

**MCP vs Skill**
- MCP = AI에게 "손"을 준다. 대화 중 실시간 웹 접근. 범용 챗봇에 적합.
- Skill = AI에게 "설명서"를 준다. 토큰 소모가 적고, 코딩 에이전트가 제품 코드를 작성할 때 적합.
- 둘 다 설치 가능. `npx -y firecrawl-cli@latest init --all --browser` 하나로 CLI+스킬 동시 설치.

## 10. API 토큰이 꼭 필요한가

| 상황 | 키 필요 | 비고 |
| --- | --- | --- |
| 클라우드 API | 필요 | `fc-` 접두사, `Authorization: Bearer` |
| Playground | 불필요 | 브라우저에서 바로 테스트 |
| 호스티드 MCP | 부분적으로 불필요 | keyless 티어 (IP당 쿼터) |
| 셀프호스팅 기본 | 불필요 | `USE_DB_AUTHENTICATION=false` |
| 셀프호스팅 인증 활성화 | 필요 | DB 스키마 직접 구축 필요 |

`apps/api/src/controllers/v2/keyless-eligibility.ts` — 호스티드 MCP가 키 없는 사용자의
IP 자격을 쿼터 소모 없이 사전 확인하는 내부 엔드포인트. `KEYLESS_PROXY_SECRET`으로 보호되고
`x-firecrawl-keyless-ip` 헤더로 IP를 전달한다. 즉 MCP 경유 시 키 없이도 일부 무료 동작.

**키 관리 수칙**
- `.env`에 보관하고 커밋 금지.
- **프런트엔드(React)에 키를 넣지 말 것.** 반드시 백엔드 경유.
- 셀프호스팅 사용 시 SDK의 `FIRECRAWL_API_URL`만 변경.
- 사용량은 `/v2/credit-usage`, 동시성은 `/v2/concurrency-check`로 모니터링.

## 11. AI 에이전트 구축에 도움이 되는가 — 된다

README 자체가 `"The web data API ... your agents can ship with"`,
`"built for real-time agents"`로 표현한다. P95 지연 3.4초.

| 에이전트의 약점 | Firecrawl 해법 |
| --- | --- |
| 학습 시점 이후를 모름 | `search` → 실시간 정보 |
| 특정 사이트 내용을 못 봄 | `scrape` → 본문 |
| 토큰 낭비 (HTML 노이즈) | 마크다운 출력 → 토큰 절약 |
| 구조화 데이터 필요 | `agent` + 스키마 → 검증된 JSON |
| 웹 조작 불가 | `interact` → 클릭·입력·로그인 |
| URL을 모름 | `agent` → AI가 탐색 |

```python
# RAG 지식베이스
docs = app.crawl("https://docs.mycompany.com", limit=500)

# 리서치 에이전트
sources = app.search(user_question, limit=5)

# 자율 수집
data = app.agent(prompt="2026년 서울 신축 아파트 분양가 정리", schema=MySchema)
```

**제약**: 호출당 크레딧이 소모되고 에이전트 루프에서 폭증한다.
**캐싱 + 호출 상한을 반드시 설계할 것.** 수집 데이터의 저작권·robots.txt 책임은 사용자에게 있다.

## 12. React / PHP로 만들 수 있는가

**(A) Firecrawl 자체를 React·PHP로 재구현 — 비현실적**
- TypeScript + Go + Rust/WASI 혼합이며 언어 선택이 성능 때문이다.
- 봇 우회 핵심(`fire-engine`)이 비공개라 참조할 소스가 없다.
- 프록시 풀 운영비, 분산 큐 운영 부담이 크다.
- PHP는 장시간 브라우저 세션·대규모 비동기 큐에 구조적으로 불리하다.

**(B) React·PHP로 Firecrawl 기반 제품 만들기 — 공식 지원, 권장**

PHP SDK (`apps/php-sdk/composer.json` 확인 결과):
- 패키지 `firecrawl/firecrawl-sdk`, **MIT 라이선스**, PHP ^8.1, Guzzle ^7.9
- **Laravel 10/11/12 지원**: `FirecrawlServiceProvider` 자동 등록 + `Firecrawl` Facade
- `laravel/ai` 툴 클래스 포함 (`Firecrawl\Laravel\Tools`, PHP 8.3+/Laravel 12+)
- 테스트 Pest + PHPUnit, 정적분석 PHPStan

```php
$client = FirecrawlClient::create(apiKey: 'fc-YOUR_KEY');
$doc = $client->scrape('https://firecrawl.dev', ScrapeOptions::with(formats: ['markdown']));
echo $doc->getMarkdown();
```

React: `apps/ui/ingestion-ui/` 에 React 18 + Vite + TS + Tailwind + Radix + shadcn 패턴
데모 UI가 이미 있다. 포크해서 제품 프런트엔드로 사용 가능.

```bash
cd apps/ui/ingestion-ui && npm install && npm run dev
```

권장 아키텍처:

```
[React 프런트] --fetch--> [PHP/Laravel 백엔드] --SDK--> [Firecrawl API]
   화면·입력·표시           API 키 보관 / 과금·제한 / 캐싱
```

React에서 Firecrawl을 직접 호출하면 API 키가 브라우저에 노출된다. 금지.

### 라이선스 주의 (상업화 시 핵심)

| 사용 방식 | 라이선스 영향 |
| --- | --- |
| 클라우드 API를 SDK로 호출 | 문제 없음 (SDK는 MIT) |
| 셀프호스팅을 내부에서만 사용 | 문제 없음 |
| **셀프호스팅을 수정해 SaaS로 서비스** | **AGPL-3.0 → 소스 공개 의무** |

## 13. 유튜브 강의 제작 — 적합

**근거**: 30초 만에 시각적 결과가 나오고, 한국어 콘텐츠가 희박하며,
오픈소스라 화면에 코드를 띄우는 데 제약이 없고, 예제가 58개 준비되어 있다.

| 회차 | 제목 | 길이 |
| --- | --- | --- |
| 1 | 웹 스크래핑이 왜 지옥인가 (403 체험) | 8분 |
| 2 | Firecrawl 5분 만에 첫 스크랩 | 10분 |
| 3 | 5대 기능 완전정복 (scrape/search/map/crawl/batch) | 15분 |
| 4 | MCP 연동 — Claude에게 실시간 눈 달아주기 | 12분 |
| 5 | Agent Skill — 코딩 에이전트에게 설명서 주기 | 12분 |
| 6 | 회사 문서 → RAG 챗봇 (crawl + 벡터DB) | 20분 |
| 7 | 경쟁사 가격 모니터링 + 슬랙 알림 | 18분 |
| 8 | React + Laravel 실전 SaaS | 25분 |
| 9 | Docker 셀프호스팅 (fire-engine 한계 포함) | 20분 |
| 10 | `/interact`로 로그인 필요한 사이트 다루기 | 18분 |

**반드시 다룰 윤리·법 파트**: robots.txt 준수(기본 동작), 사이트 이용약관,
개인정보보호법, 저작권. README 문구 인용 —
"웹사이트 정책 준수는 전적으로 최종 사용자의 책임".
"차단 우회" 중심 구성은 피하고 "공개 데이터 수집 + 자동화" 프레임으로 갈 것.

**제작 실무**: 녹화 중 `crawl limit` 과다 설정으로 크레딧 소모 주의,
API 키 화면 노출 금지 및 녹화 후 재발급, 셀프호스팅 편에서 fire-engine 한계를 솔직히 설명,
버전 변경이 잦으므로(v1→v2, extract→agent, spark-1→spark-2) 설명란에 촬영 시점 버전 명시.

---

## 14. 수익화 아이디어

> 원칙: Firecrawl은 "원유 시추기"다. 원유(스크래핑 대행)를 파는 것은 단가 싸움이고,
> 정제한 휘발유(특정 업종의 의사결정 데이터)를 파는 것이 돈이 된다.

### 티어 0 — 자본 0원, 1~4주

**1. 데이터 수집 외주 (프리랜싱)** — ★☆☆☆☆
크몽·숨고·Upwork의 "상품 3만개 수집" 류 의뢰. 남들은 Selenium으로 3일, 당신은 `batch_scrape`로 3시간.
건당 10~150만원, 월 300~800만원 현실적. 템플릿: `examples/web_data_extraction/`

**2. 유튜브 + 강의 + 제휴** — ★★☆☆☆
애드센스 + 자체 강의(5~15만원) + 제휴 링크. 3~6개월 후 수익화.

**3. AI 리서치 보고서 판매** — ★★☆☆☆
`/agent`로 시장 조사 자동화 → PDF 보고서. 건당 3~30만원 또는 월 구독.
수집은 자동, 해석·편집만 사람이 추가 → 1인 리서치 회사.

### 티어 1 — 작은 SaaS (1~3개월, React+Laravel로 충분)

**4. 경쟁사 가격 모니터링 — 1순위 추천** — ★★☆☆☆
타겟: 쿠팡·스마트스토어 셀러, 호텔, 항공, 이커머스.
경쟁사 URL 등록 → 매시간 크롤 → 변동 시 슬랙/카톡 알림 + 그래프. 월 3~30만원.
`scrape` + `changeTracking` 옵션(API 내장) + cron. MVP 2~3주.
가격 변동은 곧 매출이므로 결제 저항이 가장 낮다. 참고: `examples/o3-mini-deal-finder/`

**5. 영업 리드 보강 (Lead Enrichment)** — ★★★☆☆
회사명 목록 → 대표자·직원수·기술스택·채용공고·최근뉴스 자동 보강.
구현체 존재: `examples/crm_lead_enrichment/`, `sales_web_crawler/`
Apollo·ZoomInfo는 비싸고 한국 기업 데이터가 빈약 → 로컬라이즈가 경쟁력.

**6. 사내 문서 AI 챗봇 (RAG as a Service)** — ★★★☆☆
사이트 URL → crawl → 벡터DB → 임베드 챗봇 위젯.
구축비 100~500만원 + 월 유지 5~30만원. 구축·유지보수 이중 수익.
참고: `examples/web_data_rag_with_llama3/`, `website_qa_with_gemini_caching/`

**7. SEO 내부링크 최적화 도구** — ★★☆☆☆
전용 예제 2개 존재: `find_internal_link_opportunites/`, `internal_link_assistant/`
사이트 크롤 → 키워드 분석 → 내부링크 제안. 월 2~20만원.
SEO 업계는 도구 비용 지불에 익숙하다(Ahrefs 월 10만원+).

### 티어 2 — 버티컬 특화 (3~12개월, 진짜 수익)

**8. 업종 특화 데이터 상품** — ★★★★☆
범용 스크래퍼는 Firecrawl과 경쟁하게 되므로 금지. 좁고 깊게 간다.

| 버티컬 | 수집 대상 | 고객 | 가격대 |
| --- | --- | --- | --- |
| 부동산 | 매물·실거래·분양 (`deep-research-apartment-finder/`) | 중개사·투자자 | 월 5~50만원 |
| 채용 | 공고·연봉·스택 (`o1_job_recommender/`) | 헤드헌터·구직자 | 월 2~20만원 |
| 금융 | 공시·뉴스·감성 (`claude_stock_analyzer/`) | 개인투자자 | 월 3~30만원 |
| 이커머스 | 리뷰·랭킹·재고 | 셀러 | 월 5~50만원 |
| 법률/입찰 | 판례·나라장터 공고 | 로펌·조달업체 | 월 20~200만원 |

핵심은 수집이 아니라 **해석**을 파는 것 ("시세보다 12% 저렴", "요건 92% 일치").
도메인 지식 + 누적 히스토리는 복제가 어렵다.

**9. AI 콘텐츠 자동화 파이프라인** — ★★★☆☆
예제 3개: `aginews-ai-newsletter/`, `ai-podcast-generator/`, `blog-articles/`
`search` → LLM 작성 → 자동 발행. 운영 인력 0명.
단, 저품질 자동 생성은 검색 스팸 정책에 걸린다. **사람 감수 레이어 필수.**

**10. n8n/Zapier 템플릿 + 노코드 컨설팅** — ★★☆☆☆
공식 통합(Zapier, n8n, Lovable) 활용. 템플릿 2~10만원 + 기업 컨설팅 월 100~500만원.

### 티어 3 — 고수익·고난도

**11. 셀프호스팅 구축 대행 (온프레미스)** — ★★★★☆
타겟: 금융·공공·의료 등 데이터 외부 유출 금지 조직.
구축 500~3000만원 + 연 유지보수 20%.
근거: `examples/kubernetes/cluster-install/`, `firecrawl-helm/`
SELF_HOST.md가 경고한 항목들(무인증 기본값, 영속 볼륨 없음, DB 스키마 미비, TLS·네트워크 정책)이
그대로 유료 작업 항목이 된다.
라이선스: 고객사 내부 사용은 문제 없으나 **수정 후 네트워크 서비스 제공은 AGPL 소스 공개 의무.**
계약서에 명시할 것.

**12. 데이터 API 재판매** — ★★★★★
정제·정규화한 데이터셋을 API로 판매. 월 구독 + 호출당 과금.
저작권·DB권·개인정보보호법 리스크가 크다. 공개·사실 데이터로 한정하고 법률 자문 필수.

### 실행 순서

```
1단계 (1개월)   프리랜싱으로 현금 + 실력 (1) / 동시에 유튜브 시작 (2)
2단계 (2~3개월) 가격 모니터링 또는 SEO 도구 MVP (4, 7)
                → React + Laravel, apps/ui/ingestion-ui 포크
3단계 (6개월+)  2단계 고객 피드백으로 버티컬 특화 (8) ← 실제 수익 구간
```

### 비용 구조 설계 (필수)

- 매출 대비 크레딧 원가를 **20% 이하**로 유지
- **캐싱이 생명.** 동일 URL 재요청은 DB에서 반환
- `/v2/credit-usage` 실시간 모니터링, 고객 티어별 호출 상한 하드코딩
- 규모가 커지면 "평범한 사이트는 셀프호스팅 / 방어된 사이트만 클라우드" 하이브리드

### 피해야 할 것

- Firecrawl을 더 싸게 재판매하는 서비스 (본사와 정면 경쟁)
- 범용 "뭐든 긁어주는 도구" (차별화 불가, 가격 경쟁)
- 로그인 우회·차단 돌파 중심 서비스 (법적 리스크 + 플랫폼 제재)
- 개인정보 수집 (한국 개인정보보호법 과징금은 매출 기준)

---

## 15. 핵심 요약 10줄

1. 본체는 웹 → 마크다운/JSON 변환 **API 서버**다. 플러그인이 아니다.
2. MCP와 Agent Skill, CLI는 "포장지"이며 MCP·CLI 소스는 별도 저장소에 있다.
3. 루트의 `firecrawl-cli`, `firecrawl-skills`, `firecrawl-cli-skills`, `firecrawl-workflows`는 포인터 README뿐이다.
4. 봇 우회 핵심 `fire-engine`은 비공개 유료다. 셀프호스팅은 Playwright + fetch만 동작한다.
5. 클라우드 사용에는 `fc-` API 키가 필요하고, 호스티드 MCP에는 keyless 무료 티어가 있다.
6. `/agent`가 현재의 주력 — URL 없이 프롬프트만으로 수집·구조화한다.
7. AI 에이전트의 "실시간 눈" 문제를 해결하며, 애초에 에이전트용으로 설계됐다.
8. React·PHP로 **제품을 만드는 것**은 공식 지원(PHP SDK는 Laravel 통합 + MIT, React 데모 UI 포함)이고, 엔진 재구현은 비현실적이다.
9. 상업 SaaS는 클라우드 API 구독이 안전하다. 셀프호스팅 수정 후 서비스는 AGPL 소스 공개 의무가 생긴다.
10. 수익화는 "범용 스크래퍼"가 아니라 **버티컬 특화 + 해석 제공**으로 간다. `examples/` 58개가 검증된 출발점이다.
