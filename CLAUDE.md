# Firecrawl (firecrawl/firecrawl)

## 프로젝트 개요
인터넷의 어떤 복잡한 웹사이트라도 AI가 읽고 학습하기 가장 완벽한 깔끔한 문서로 순식간에 긁어오는 "AI 전용 웹 데이터 초고속 수확기"
로그인 장벽, 복잡한 자바스크립트 화면, 봇 차단 기술을 우회하여 웹페이지의 핵심 알맹이 텍스트만 깨끗하게 추출
최신 웹 정보를 실시간으로 AI 검색과 지식 베이스로 만들고 싶은 전 세계 18만 개발자들의 필수 웹 스크래퍼

## 핵심 특징 & 추천 분야
- AI웹데이터수확기
- 18만개발자선택
- 초고속웹스크래퍼
- 봇차단우회
- 지식베이스구축

---
*이 문서는 오픈소스 큐레이터(Curator-Agent)에 의해 자동 생성된 가이드 문서입니다.*


---
## 기존 CLAUDE.md 내용

Firecrawl is a web scraper API. The directory you have access to is a monorepo:
 - `apps/api` has the actual API and worker code
 - `apps/*-sdk` are various SDKs

When making changes to the API, here are the general steps you should take:
1. Write some end-to-end tests that assert your win conditions, if they don't already exist
  - 1 happy path (more is encouraged if there are multiple happy paths with significantly different code paths taken)
  - 1+ failure path(s)
  - Generally, E2E (called `snips` in the API) is always preferred over unit testing.
  - In the API, always use `scrapeTimeout` from `./lib` to set the timeout you use for scrapes.
  - These tests will be ran on a variety of configurations. You should gate tests in the following manner:
    - If it requires fire-engine: `!process.env.TEST_SUITE_SELF_HOSTED`
    - If it requires AI: `!process.env.TEST_SUITE_SELF_HOSTED || process.env.OPENAI_API_KEY || process.env.OLLAMA_BASE_URL`
2. Write code to achieve your win conditions
3. Run your tests using `pnpm harness jest ...`
  - `pnpm harness` is a command that gets the API server and workers up for you to run the tests. Don't try to `pnpm start` manually.
  - The full test suite takes a long time to run, so you should try to only execute the relevant tests locally, and let CI run the full test suite.
4. Push to a branch, open a PR, and let CI run to verify your win condition.
Keep these steps in mind while building your TODO list.

Never bypass `knip` failures (e.g. with `git commit --no-verify`). If the pre-commit `knip` check fails, fix the reported unused exports/files — even if they predate your change — before committing.