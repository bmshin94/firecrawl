# Firecrawl 전수조사 · Q&A · 수익화 대화 정리 (한국어)

> 작성일: 2026-10-04
> 저장소 전수조사 기준 커밋: `claude/github-project-analysis-jrcerm` 브랜치 시점
> 이 문서는 Firecrawl 저장소를 직접 열어 확인한 내용 + 질의응답 + 수익화 아이디어를 한 번에 정리한 요약본입니다.

---

## 📎 관련 GitHub 주소

| 대상 | 주소 |
| --- | --- |
| **Firecrawl 본체 (업스트림)** | https://github.com/firecrawl/firecrawl |
| **이 작업 저장소 (포크)** | https://github.com/bmshin94/firecrawl |
| 작업 브랜치 | https://github.com/bmshin94/firecrawl/tree/claude/github-project-analysis-jrcerm |
| Firecrawl CLI | https://github.com/firecrawl/cli |
| Agent Skills 카탈로그 | https://github.com/firecrawl/skills |
| Workflow Skills | https://github.com/firecrawl/firecrawl-workflows |
| MCP 서버 | https://github.com/firecrawl/firecrawl-mcp-server |
| 공식 사이트 / 문서 / 플레이그라운드 | https://firecrawl.dev · https://docs.firecrawl.dev · https://firecrawl.dev/playground |
| 셀프호스팅 가이드 | https://docs.firecrawl.dev/contributing/self-host |

---

## 1. 한 줄 정의

**Firecrawl = 어떤 웹사이트든 AI(LLM)가 바로 쓸 수 있는 깨끗한 Markdown / JSON으로 바꿔주는 웹 데이터 API.**

LLM은 "추론"은 잘하지만 "최신 웹을 읽는 눈"이 없다. Firecrawl이 그 눈을 담당한다.

```
🧠 추론 = LLM (Claude / GPT)
👀 인지 = Firecrawl   ← 이 프로젝트
🦾 행동 = Tool / MCP
```

---

## 2. 해결하는 문제

| 웹 스크래핑의 고통 | Firecrawl의 처리 |
| --- | --- |
| JS로 렌더링되는 SPA → `curl`은 빈 껍데기 | 실제 브라우저(Chromium) 렌더링 후 추출 |
| Cloudflare 등 봇 차단 | 프록시 로테이션 + stealth 엔진 자동 재시도 |
| 광고·메뉴·푸터 HTML 쓰레기 | 본문만 추출해 Markdown 변환 |
| 사이트 전체 크롤러 직접 구현 | `/crawl` 한 번으로 전체 수집 |
| PDF / DOCX 링크 | 파싱해 텍스트로 |
| 셀렉터 깨짐 · 프록시 관리 | 전부 서비스 측에서 관리 |

**체감 차이**: 직접 구현 시 2주 + 영구 유지보수 → Firecrawl은 코드 3줄.

```python
from firecrawl import Firecrawl
app = Firecrawl(api_key="fc-...")
print(app.scrape("https://example.com").markdown)
```

---

## 3. 저장소 구조 (전수조사 결과)

```
firecrawl/
├── apps/                          # 실제 코드 (18개 디렉토리)
│   ├── api/                       # ★ 핵심 API 서버 + 워커 (의존성 156개)
│   ├── playwright-service-ts/     # 브라우저 렌더링 마이크로서비스
│   ├── go-html-to-md-service/     # HTML→Markdown 변환 (Go, 속도 목적 분리)
│   ├── nuq-postgres/              # 자체 작업 큐 (Postgres 기반)
│   ├── redis/, siem/              # 캐시 / 보안 로그
│   ├── {python,js,go,java,php,ruby,rust,dot-net,elixir}-sdk/   # 공식 SDK 9종
│   ├── ui/ingestion-ui/           # 데모용 React UI
│   └── test-site/, test-suite/
├── skills/                        # ★ Agent Skill 5개 (실물 존재)
├── examples/                      # ★ 실전 예제 58개
├── docker-compose.yaml            # 셀프호스팅 스택
├── LICENSE                        # AGPL-3.0 (중요)
├── README.md / SELF_HOST.md / CONTRIBUTING.md / AGENTS.md / CLAUDE.md
└── firecrawl-cli/, firecrawl-skills/, firecrawl-cli-skills/, firecrawl-workflows/
```

### 함정 1 — 루트의 4개 폴더는 빈 껍데기

`firecrawl-cli/`, `firecrawl-skills/`, `firecrawl-cli-skills/`, `firecrawl-workflows/`는 **README 한 장뿐**이고 각각 외부 저장소를 가리키는 표지판이다. CLI·스킬 소스를 이 폴더에서 찾으면 안 된다.

### 함정 2 — `apps/api`는 완전체가 아니다 (오픈코어)

`apps/api/src/scraper/scrapeURL/engines/index.ts`의 실제 엔진 목록:

```ts
export type Engine =
  | "fire-engine;chrome-cdp"            // 🔒 비공개 (상용)
  | "fire-engine;chrome-cdp;stealth"    // 🔒 비공개 (봇 차단 우회 핵심)
  | "fire-engine;tlsclient"             // 🔒 비공개
  | "playwright"                        // ✅ 오픈소스
  | "fetch"                             // ✅ 오픈소스
  | "pdf" | "image" | "document" | "index" | "wikipedia" | "x-twitter" | "exchange" ...
```

> **"96% 웹 커버리지 · 봇 차단 우회"의 주역 `fire-engine`은 오픈소스가 아니다.**
> 셀프호스팅 시에는 `playwright` + `fetch`만 사용 가능 → 어려운 사이트는 실패한다.
> 소스는 공개, 가장 강한 부품은 클라우드 전용인 **오픈코어 모델**.

---

## 4. 기능 = API 엔드포인트 (`apps/api/src/controllers/v2/`, 라우트 73개)

| 엔드포인트 | 하는 일 | 쓰는 순간 |
| --- | --- | --- |
| `/scrape` | URL 1개 → Markdown / JSON / HTML / 스크린샷 | "이 페이지 내용 줘" |
| `/search` | 웹 검색 + 결과 본문까지 한 번에 | URL을 모를 때 |
| `/crawl` | 사이트 전체 재귀 크롤 (비동기 job) | 문서 사이트 → RAG DB |
| `/map` | 사이트 URL 목록만 초고속 수집 | 전체 크롤 전 사전 조사 |
| `/batch-scrape` | URL 수천 개 병렬 | 대량 수집 |
| `/agent` | **프롬프트만 주면 AI가 검색·탐색·추출** | "Notion 요금제 찾아줘" |
| `/interact` | 클릭·입력·스크롤 (AI 프롬프트 가능) | 로그인 후 데이터, 검색창 조작 |
| `/extract` | 스키마 기반 구조화 추출 (레거시) | 정형 JSON |
| `/parse` | PDF·DOCX 업로드 파싱 | 문서 → 텍스트 |

부가 기능: `credit-usage`(크레딧 조회), `concurrency-check`, `crawl-status-ws`(웹소켓 실시간), `change-tracking`(변경 감지), `keyless-eligibility`(무료 체험 경로) 등.

### 특히 중요한 2개

**`/agent` — URL을 몰라도 된다**

```python
r = app.agent(prompt="Compare pricing of Firecrawl, Apify, ScrapingBee", effort="high")
# effort: low / medium / high  (모델은 spark-2, effort는 추론 예산을 조절)
# 레거시 model 파라미터: spark-1-mini / spark-1-pro(기본) / spark-2
# model과 effort를 함께 보내면 400 에러
```

**`/interact` — 스크랩 후 조작**

```python
res = app.scrape("https://amazon.com")
app.interact(res.metadata.scrape_id, prompt="Search for 'mechanical keyboard'")
app.interact(res.metadata.scrape_id, prompt="Click the first result")
# 응답에 liveViewUrl 포함 → 실제 브라우저 화면 관찰 가능
```

### 숨은 보석: `transformers/`

`llmExtract.ts`, `product.ts`(상품), `menu.ts`(메뉴), `diff.ts`(변경), `redactPII.ts`(개인정보 마스킹), `video.ts`, `audio.ts` — 가격 비교·리뷰 수집 같은 서비스에 바로 쓸 수 있는 특화 로직이 이미 들어 있다.

---

## 5. `examples/` 58개 — 가장 실용적인 폴더

| 분류 | 예제 |
| --- | --- |
| 기업 리서치 | `R1_company_researcher`, `gpt-4.1-company-researcher`, `deepseek-v3-company-researcher` |
| 영업 / CRM | `crm_lead_enrichment`, `sales_web_crawler` |
| SEO | `find_internal_link_opportunites`, `internal_link_assistant` |
| 콘텐츠 자동화 | `aginews-ai-newsletter`, `ai-podcast-generator`, `blog-articles` |
| 금융 | `claude-3.7-stock-analyzer`, `claude_stock_analyzer`, `o3-mini-deal-finder` |
| 채용 | `o1_job_recommender`, `job-resource-analyzer` |
| RAG / QA | `web_data_rag_with_llama3`, `website_qa_with_gemini_caching` |
| 실용 | `deep-research-apartment-finder`, `scrape_and_analyze_airbnb_data_e2b` |
| 멀티에이전트 | `openai_swarm_firecrawl`, `openai-realtime-firecrawl` |
| 인프라 | `kubernetes/` (매니페스트 + Helm 차트) |

모델별 버전(Claude / GPT / Gemini / Llama / Mistral / Grok / DeepSeek)이 모두 준비되어 있다.

---

## 6. `skills/` — Agent Skill 5개

```
skills/
├── firecrawl-build/            (+ references 6개: project-intake, endpoint-selection,
│                                 integration-patterns, sdk-installation, auth-and-env, verification)
├── firecrawl-build-scrape/     (+ references/freshness-and-liveness.md)
├── firecrawl-build-search/
├── firecrawl-build-interact/
└── firecrawl-build-onboarding/ (+ references 3개)
```

- `SKILL.md`의 description은 **"사용자가 Firecrawl을 언급하지 않고 '웹 데이터가 필요하다'고만 말해도 발동"**하도록 설계되어 있고, `fire girl`이라는 별칭까지 트리거로 등록돼 있다.
- 엔드포인트 라우팅 규칙도 스킬 안에 명시: 알고 있는 URL 1개 → `/scrape`, 쿼리 → `/search`, 클릭·폼이 필요하면 → `/interact`. 추가로 논문 인덱스·개발자 인덱스는 `/search` 대상이 아니라는 안내도 있다.

### 스킬 저장소 역할 분담

| 종류 | 용도 | 위치 |
| --- | --- | --- |
| **build 스킬** | 앱 코드에 Firecrawl 통합 | 이 저장소 `skills/` → CI가 카탈로그로 미러링 |
| **CLI 스킬** | 터미널에서 즉석 웹 작업 | `firecrawl/cli` |
| **workflow 스킬** | 경쟁사 분석 등 반복 산출물 | `firecrawl/firecrawl-workflows` |
| 카탈로그 | 설치 지점 (읽기 전용) | `firecrawl/skills` |

---

## 7. 인프라 (`docker-compose.yaml`)

| 서비스 | 역할 |
| --- | --- |
| `api` / `worker` | API 서버와 작업 워커 (호스트에 `3002`만 공개) |
| `playwright-service` | 브라우저 렌더링 |
| `redis` | 캐시 / 레이트리밋 (valkey 대체 가능) |
| `rabbitmq` | 메시지 큐 (관리 UI 포함 이미지) |
| `nuq-postgres` | 기본 작업 큐 백엔드 |
| `foundationdb` (+init) | 선택적 큐 백엔드 (`NUQ_BACKEND=fdb`) |

CI 워크플로(`.github/workflows/`)에는 SDK 9종 자동 퍼블리시, 이미지 배포, `scrape-evals`, `npm-audit` 자동 수정 등이 구성돼 있다.

### SELF_HOST.md가 경고하는 것 (중요)

| 항목 | 기본값 | 의미 |
| --- | --- | --- |
| API 인증 | `USE_DB_AUTHENTICATION=false` | **인증 없음 → 외부 노출 금지** |
| 스크래핑 엔진 | Playwright + fetch | **fire-engine 없음 → 어려운 사이트 실패** |
| AI 기능 | 모델 제공자 없음 | OpenAI / OpenAI 호환 / Ollama 별도 연결 |
| 영구 저장 | **볼륨 정의 없음** | 컨테이너 교체 시 데이터 소실 |
| 큐 관리 UI | 꺼짐 | 강한 `BULL_AUTH_KEY` + 망 제한 후에만 |
| 운영 책임 | 전부 본인 | 보안·가용성·용량·업그레이드·보존·컴플라이언스 |

또한 루트 `.env`는 `docker-compose.yaml`이 참조하는 변수만 덮어쓰며, `apps/api/.env.example`을 Compose 설정으로 그대로 쓰면 안 된다고 명시돼 있다.

> 결론: **학습·테스트는 셀프호스팅, 사업은 클라우드 API.**

---

## 8. 설치 및 사용법 — 4가지 경로

### 경로 A. 클라우드 API (권장, 약 5분)

```bash
# https://firecrawl.dev 가입 → fc-로 시작하는 API 키 발급
pip install firecrawl-py              # Python
npm install firecrawl                 # Node.js
composer require firecrawl/firecrawl  # PHP
```

```python
from firecrawl import Firecrawl
app = Firecrawl(api_key="fc-YOUR_KEY")

doc   = app.scrape("https://firecrawl.dev", formats=["markdown"])
res   = app.search("best AI data tools 2025", limit=10)
docs  = app.crawl("https://docs.firecrawl.dev", limit=50)   # SDK가 폴링 자동 처리
job   = app.batch_scrape([...], formats=["markdown"])
agent = app.agent(prompt="Find the founders of Stripe", effort="high")
```

```bash
curl -X POST 'https://api.firecrawl.dev/v2/scrape' \
  -H 'Authorization: Bearer fc-YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"url": "firecrawl.dev"}'
```

> 코드 작성 전에 https://firecrawl.dev/playground 에서 먼저 결과를 확인하는 게 빠르다.

### 경로 B. CLI

```bash
npx -y firecrawl-cli@latest init --all --browser

firecrawl search "firecrawl" --limit 5
firecrawl scrape https://firecrawl.dev --only-main-content
firecrawl interact exec --prompt "Click the first result"
```

### 경로 C. Agent Skill

```bash
npx skills add firecrawl/skills --skill firecrawl-build
# 설치 후 에이전트 재시작 필수
```

에이전트가 스스로 온보딩하는 경로도 제공된다: `curl -s https://firecrawl.dev/agent-onboarding/SKILL.md`

### 경로 D. 셀프호스팅 (무료, 기능 제한)

```bash
git clone https://github.com/firecrawl/firecrawl
cd firecrawl
cp apps/api/.env.example .env     # 편집
docker compose up -d              # → localhost:3002
curl -X POST 'http://localhost:3002/v2/scrape' \
  -H 'Content-Type: application/json' -d '{"url":"example.com"}'
```

### 개발 기여 규칙 (CLAUDE.md 기준)

- `apps/api` = API + 워커, `apps/*-sdk` = SDK
- E2E(`snips`) 테스트 우선, 해피패스 1개 + 실패 경로 1개 이상
- 스크랩 타임아웃은 `./lib`의 `scrapeTimeout` 사용
- 테스트 게이팅: fire-engine 필요 → `!process.env.TEST_SUITE_SELF_HOSTED`, AI 필요 → `!TEST_SUITE_SELF_HOSTED || OPENAI_API_KEY || OLLAMA_BASE_URL`
- 실행은 `pnpm harness jest ...` (직접 `pnpm start` 금지)
- `knip` 실패를 `--no-verify`로 우회 금지

---

## 9. 플러그인인가, 스킬인가, MCP인가

**본질은 "API 서비스"이고, 나머지는 그것을 감싸는 포장지다.**

```
                🔥 Firecrawl API (본체)
          api.firecrawl.dev / 셀프호스팅 :3002
                        ↑
   ┌──────────┬─────────┴────────┬──────────────┐
 ① SDK      ② CLI            ③ MCP          ④ Skill
(코드용)   (터미널용)      (에이전트 도구)  (에이전트 판단력)
```

| 형태 | 정체 | 설치 | 언제 |
| --- | --- | --- | --- |
| SDK | 라이브러리 9종 | `pip install firecrawl-py` / `npm i firecrawl` | 앱에 통합 |
| CLI | 터미널 도구 | `npx -y firecrawl-cli@latest init` | 즉석 웹 작업 |
| **MCP** | MCP 서버 **있음** | `npx -y firecrawl-mcp` | Claude Desktop / Cursor 등 |
| **Skill** | Agent Skill **있음** (`skills/` 5개) | `npx skills add firecrawl/skills` | 에이전트가 사용법을 학습 |
| 플러그인 | **해당 형태 아님** | — | — |

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

### MCP vs Skill 차이

- **MCP = 손(도구)을 달아준다.** 에이전트가 직접 `scrape`를 호출할 수 있다.
- **Skill = 설명서를 준다.** 언제 어떤 엔드포인트를 쓰고, 키를 어디에 두고, 어떻게 통합하는지 판단력을 준다.

추천 조합: 터미널 작업 → CLI + CLI 스킬 / 앱 개발 → build 스킬 + SDK / Cursor·Claude Desktop → MCP.

---

## 10. API 토큰이 꼭 필요한가

| 상황 | 토큰 | 설명 |
| --- | --- | --- |
| 클라우드 API | **필수** | `Authorization: Bearer fc-...` |
| 셀프호스팅 (기본) | 불필요 | `USE_DB_AUTHENTICATION=false` → 인증 자체가 없음 |
| 셀프호스팅 + 인증 | DB 스키마·설정 직접 구축 | 변수 하나만 바꿔선 완성되지 않음 |
| AI 기능(agent/extract) | 별도 LLM 키 | Firecrawl에 자체 모델 없음 |

```bash
# .env — 코드 하드코딩 금지
FIRECRAWL_API_KEY=fc-xxxxx
FIRECRAWL_API_URL=http://localhost:3002   # 셀프호스팅일 때만
```

> **프론트엔드(React)에 키를 두면 안 된다.** 브라우저 코드는 공개되므로 반드시 백엔드를 경유한다.

---

## 11. AI 에이전트 구축에 도움이 되는가 — 된다

README에도 "Agent ready"로 명시돼 있고, 사실상 주 용도다.

```python
# ① RAG 지식베이스
docs = app.crawl("https://docs.내제품.com", limit=500)
vector_db.add([d.markdown for d in docs.data])

# ② 리서치 에이전트 (검색 + 본문 동시)
res = app.search("2026 AI 시장 전망", limit=20)

# ③ 자율 수집 (URL 불필요)
r = app.agent(prompt="경쟁사 3곳 요금제 비교", effort="high")

# ④ 웹 조작
s = app.scrape("https://site.com")
app.interact(s.metadata.scrape_id, prompt="로그인 버튼 클릭")
```

### 한계 (솔직하게)

- **비용**: 크레딧 소모 → 무한 크롤 시 요금 폭증. 캐싱 필수
- **지연**: P95 약 3.4초 → 실시간 음성 대화에는 느림
- **로그인 벽**: 2FA가 걸린 사이트는 여전히 난이도 높음
- **합법성**: robots.txt · 이용약관 · 저작권 준수는 사용자 책임

---

## 12. React / PHP로 만들 수 있는가

둘 다 가능하다. PHP는 공식 SDK(`apps/php-sdk`)까지 있다.

### 절대 규칙: 키는 서버에만

```
금지:  React → Firecrawl           (키 노출)
정답:  React → 내 백엔드 → Firecrawl (키 보호 + 캐싱 + 과금 제어)
```

### PHP (Laravel) 백엔드

```php
Route::post('/scrape', function (Request $r) {
    $r->validate(['url' => 'required|url']);
    $key = 'scrape:' . md5($r->url);

    return Cache::remember($key, 3600, function () use ($r) {   // 캐싱이 생명
        return Http::withToken(env('FIRECRAWL_API_KEY'))
            ->post('https://api.firecrawl.dev/v2/scrape', [
                'url' => $r->url, 'formats' => ['markdown'],
            ])->json();
    });
})->middleware('throttle:10,1');   // 레이트리밋 필수
```

### React 프론트

```jsx
const run = async (url) => {
  const res = await fetch('/api/scrape', {        // 내 백엔드 호출
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url }),
  });
  setMarkdown((await res.json()).data.markdown);
};
```

오래 걸리는 `crawl`은 **job 등록 → 큐 워커가 Firecrawl 호출 → webhook/폴링 → 진행률 표시** 구조로 설계한다. Firecrawl 본체도 RabbitMQ + nuq-postgres 큐 구조를 쓰므로 설계를 참고할 수 있다.

### 라이선스 주의 (상업화 핵심)

| 대상 | 라이선스 | 상업적 의미 |
| --- | --- | --- |
| 본체 (루트 `LICENSE`) | **AGPL-3.0** | 수정해 SaaS로 제공 시 **소스 공개 의무** |
| SDK (js / python 등) | **MIT** | 자유로운 상업 이용 가능 |
| 클라우드 API 사용 | 상용 약관 | 공개 의무 없음 |

> 안전한 길: **클라우드 API + MIT SDK**로 서비스 구축. AGPL 본체를 포크·개조해 비공개 SaaS로 운영하는 것은 위험하다.

---

## 13. 유튜브 강의 제작 — 적합

- 결과가 시각적으로 즉시 보임(코드 3줄 → 깔끔한 Markdown)
- "AI + 자동화 + 수익화" 조합은 조회수에 유리
- 한국어 강의가 거의 없어 선점 가능
- 무료 크레딧으로 시청자가 바로 따라할 수 있어 완주율이 높음

### 10부작 커리큘럼

| 회차 | 주제 |
| --- | --- |
| 1 | 5분 만에 웹사이트를 AI 데이터로 (훅: 코드 3줄 시연) |
| 2 | scrape / search / crawl / map 완전정복 |
| 3 | 사이트 전체 크롤 → 나만의 ChatGPT (RAG) |
| 4 | URL 없이 AI가 찾아오는 `/agent` |
| 5 | 로그인·클릭 자동화 `/interact` |
| 6 | React + Laravel 실전 웹앱 |
| 7 | Claude Code에 MCP·Skill 붙이기 |
| 8 | Docker 셀프호스팅 (한계까지 솔직히) |
| 9 | 경쟁사 가격 모니터링 봇 (수익형 실전) |
| 10 | 크레딧 절약 + 법적 주의사항 |

**제작 팁**: 1~2분 내 완성 화면 먼저 노출 / `examples/` 58개 = 에피소드 소재 / 매 영상 키 보안 경고 / 코드 GitHub 공개 → 설명란 링크.
**주의**: "무한 무료 크롤링" 같은 과장 금지(fire-engine은 유료), 스크래핑 합법성 면책 문구, 버전·날짜 표기.

---

## 14. 수익화 아이디어

> **사고의 틀**: Firecrawl은 "수집(1단계)" 인프라다. 사업은 "가공·해석·배달(2단계)"에서 생긴다.
> 실패 공식: "스크래핑 API 재판매" (고객이 Firecrawl을 직접 쓰면 됨 → 중간상 가치 0)
> 성공 공식: 특정 직군의 특정 고통을 수집 데이터로 해결 → 월 구독
> 가격 공식: 원가(크레딧) × 10~100배. 데이터는 싸고 "안 봐도 되는 시간"은 비싸다.

### 티어 0 — 자본 0원, 1~4주

1. **외주 데이터 수집 대행** — "경쟁사 500개 상품 가격·리뷰 엑셀 정리" 건당 10~50만원, 원가 수천원. 채널: 크몽·숨고·Upwork·Fiverr. 시작점: `examples/sales_web_crawler`
2. **데이터셋 판매** — 업종별 업체 리스트, 채용공고 아카이브, 가격 히스토리. 10~100만원 → 업데이트 구독으로 MRR 전환. 개인정보·저작권 확인 필수(`transformers/redactPII.ts` 참고)
3. **콘텐츠 자동화** — `/search` + `/crawl` → AI 요약 → 뉴스레터·유튜브. 애드센스·스폰서·유료 구독. 시작점: `examples/aginews-ai-newsletter`, `ai-podcast-generator`

### 티어 1 — 작은 SaaS (1~3개월, React + Laravel로 충분)

4. **경쟁사 가격·변경 모니터링** ⭐ 1순위 추천 — URL 등록 → 매일 크롤 → 변경 감지 → 슬랙/이메일 알림. Free 3개 / Basic 2.9만원(30개) / Pro 9.9만원(300개). 참고 로직: `lib/change-tracking-diff.ts`
5. **AI SEO 어시스턴트** — 내부링크 기회, 메타태그 누락, 키워드 갭 리포트. 1회 3만원 / 월 9.9만원. 시작점: `examples/find_internal_link_opportunites`, `internal_link_assistant`
6. **영업 리드 발굴·보강** — 도메인 입력 → 대표자·규모·기술스택·채용 여부. 건당 500~2000원 또는 월 29~99만원. 시작점: `examples/crm_lead_enrichment` (개인정보보호법 준수)
7. **사이트 전용 AI 챗봇 구축** — 고객사 사이트 크롤 → RAG → 임베드 위젯. 구축비 50~200만원 + 월 5~20만원. 시작점: `examples/website_qa_with_gemini_caching`

### 티어 2 — 버티컬 특화 (3~12개월, 실질 수익 구간)

> 범용 툴은 경쟁이 심하다. **한 업종만 깊게** 파면 단가가 10배가 된다.

8. **부동산 매물 인텔리전스** — 다중 플랫폼 수집 → 급매·이상치 알림 → 투자 점수. 월 10~50만원. 시작점: `examples/deep-research-apartment-finder`
9. **채용시장 데이터(HR Tech)** — 직무별 연봉 추이·기술스택 트렌드·경쟁사 채용 동향. 월 30~100만원, 단건 리포트 수백만원. 시작점: `examples/o1_job_recommender`, `job-resource-analyzer`
10. **이커머스 가격 전략(Repricing)** — 경쟁사 가격 → 최적가 제안 → 자동 조정. 매출 직결이라 가격 저항이 낮다. 월 30~100만원 또는 매출 연동. 참고: `transformers/product.ts`
11. **금융·투자 시그널** — 뉴스·공시·커뮤니티 감성분석 리포트. 월 5~30만원. 투자자문 규제 주의("정보 제공" 선 유지). 시작점: `examples/claude-3.7-stock-analyzer`, `o3-mini-deal-finder`
12. **규제·공시 모니터링(RegTech)** — 법령·고시·인증 변경 알림. 미확인 시 과태료 리스크가 있어 기업 지불 의사가 가장 높다. 월 50~300만원

### 티어 3 — 고수익·고난도

13. **수직 AI 에이전트(Agent-as-a-Service)** — "경쟁사 분석 담당 AI 직원" 구독. 인건비 대체 포지셔닝으로 월 100~500만원. 컨셉 참고: `firecrawl/firecrawl-workflows`
14. **데이터 파이프라인 구축 외주** — 기업 전용 수집 시스템. 1,000~5,000만원 + 유지보수. AGPL 주의 → 클라우드 API + MIT SDK 조합 권장
15. **교육·콘텐츠** — 유튜브 무료 → 유료 강의(5~30만원) → 코드 템플릿 판매 → 컨설팅. 한국어 Firecrawl 1인자 포지션 선점

### 실행 순서 (추천)

```
Week 1-2   유튜브 1~3편 + 크몽 외주 등록        → 현금 + 시장 반응
Week 3-6   아이디어 4(가격 모니터링) MVP        → React + Laravel
Month 2-3  결제 연동 후 유료 전환               → 토스페이먼츠 / Stripe
Month 4+   반응 좋은 버티컬 하나로 티어 2 집중
```

### 비용 구조 설계 (필수)

1. **캐싱** — 동일 URL 24시간 캐시 → 크레딧 최대 90% 절감
2. **티어별 쿼터** — Free 10회/월 등 하드 리밋
3. **원가 계산** — 크레딧 단가 × 고객당 호출수 < 구독료 × 0.3
4. **`/map` 선행** — 전체 크롤 전 URL 목록만 확인
5. **증분 크롤** — 변경분만 재수집(change-tracking)
6. **사용량 알림** — 일일 임계치 초과 경보

### 피해야 할 것

- 스크래핑 API 단순 재판매(가치 0, 약관 위반 가능)
- 개인정보 수집·판매
- robots.txt / 이용약관 무시
- 저작권 콘텐츠 복제 재배포
- AGPL 본체 개조 후 비공개 SaaS 운영
- 무제한 요금제(원가 폭주)

---

## 15. 핵심 요약 10줄

1. Firecrawl = 웹을 LLM용 Markdown/JSON으로 바꿔주는 웹 데이터 API.
2. 핵심 기능 5개: `scrape`(1개) · `search`(검색) · `crawl`(전체) · `interact`(조작) · `agent`(자율).
3. 실제 코드는 `apps/`에 있고, 루트의 4개 폴더는 외부 저장소 표지판(빈 껍데기).
4. 가장 강한 엔진 `fire-engine`은 비공개 → 셀프호스팅은 playwright/fetch만, 어려운 사이트 실패.
5. 접근 방식은 SDK(9종) · CLI · MCP · Agent Skill 4가지, 본질은 하나의 API.
6. 클라우드는 API 키 필수, 셀프호스팅 기본은 인증 없음(외부 노출 금지).
7. AI 에이전트의 "눈"으로 최적 — RAG·리서치·자율 수집·웹 조작 전부 커버.
8. React/PHP 모두 가능하되 **키는 반드시 서버에만**, 캐싱·레이트리밋·쿼터는 생존 조건.
9. 라이선스: 본체 AGPL-3.0, SDK MIT → 상업화는 **클라우드 API + MIT SDK** 조합이 안전.
10. 수익화는 "수집"이 아니라 "해석"에서 나온다 → 가격 모니터링 SaaS부터 시작, 버티컬로 심화.

---

*이 문서는 저장소를 직접 전수조사한 결과와 질의응답 내용을 정리한 것입니다. 기능·요금·엔드포인트는 변경될 수 있으니 최신 정보는 https://docs.firecrawl.dev 에서 확인하세요.*
