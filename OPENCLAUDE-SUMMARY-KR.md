# OpenClaude 분석 정리 (한국어)

> 이 문서는 OpenClaude 저장소를 직접 열어보고 분석한 내용을 정리한 것입니다.
> 작성일: 2026-09-11

---

## 📌 관련 링크

| 구분 | 주소 |
| --- | --- |
| 원본 저장소 (upstream) | https://github.com/Gitlawb/openclaude |
| 내 포크 저장소 | https://github.com/bmshin94/openclaude |
| npm 패키지 | https://www.npmjs.com/package/@gitlawb/openclaude |
| 스킬 레지스트리 (Skill Hub) | https://github.com/Gitlawb/openclaude-skills |
| GitHub Discussions | https://github.com/Gitlawb/openclaude/discussions |
| Discord | https://discord.gg/k68zFR6AcB |
| X (Twitter) | https://x.com/gitlawb |
| GitLawb 미러 | https://gitlawb.com/node/repos/z6MkqDnb/openclaude |

---

## 1. 이게 뭐야?

**한 줄 요약: "Claude Code를 아무 AI 모델에서나 쓸 수 있게 개조한 오픈소스 CLI"**

터미널에서 `openclaude` 를 실행하면 AI 비서가 켜지고, 파일을 읽고 코드를 고치고
명령어를 실행하고 테스트까지 돌려줍니다. Claude Code와 거의 같은 경험인데,
**뒤에서 돌아가는 모델을 마음대로 고를 수 있다**는 게 핵심 차별점입니다.

비유하자면:

- **Claude Code** = 닌텐도 스위치 (정품 카트리지만 꽂힘)
- **OpenClaude** = 에뮬레이터 게임기 (아무 카트리지나 꽂힘 — ChatGPT, Gemini, 로컬 AI...)

### 코드베이스 규모 (실제 확인한 수치)

| 항목 | 수치 |
| --- | --- |
| TypeScript 파일 | 3,263개 |
| `src/` 총 코드 라인 | 약 876,000줄 |
| 슬래시 명령어 | 133개 |
| AI 도구(tools) | 약 50개 |
| MCP 관련 파일 | 39개 |
| 지원 프로바이더 | 25개 이상 |

장난감 수준이 아니라 대형 프로덕션 프로젝트입니다.

---

## 2. 폴더 구조

| 폴더 | 역할 |
| --- | --- |
| `src/tools/` | AI의 손발 — Bash, 파일 읽기/쓰기, Grep, 웹검색, 웹브라우저, 서브에이전트 등 |
| `src/commands/` | 슬래시 명령어 133개 (`/model`, `/cost`, `/commit`, `/doctor` ...) |
| `src/services/api/` | **프로바이더 어댑터** — 이 프로젝트의 핵심. 회사별 API를 하나로 통일 |
| `src/services/mcp/` | MCP 서버 연동 (OAuth 인증 포함) |
| `src/skills/` | 스킬 로딩/설치/검증 시스템 |
| `src/plugins/` | 플러그인 시스템 |
| `src/components/`, `src/ink/` | 터미널 UI (React + Ink) |
| `src/grpc/`, `src/proto/` | 헤드리스 gRPC 서버 (CI/CD·외부 앱 연동용) |
| `src/buddy/` | 픽셀아트 캐릭터 (엔터 칠 때마다 화살 발사) |
| `web/` | 문서 사이트 (Astro) |
| `vscode-extension/` | VS Code 확장 |
| `docs/` | 초보자용 ~ 고급 설정 가이드 |
| `.env.example` | 환경변수 예시 (28KB — 설정 항목이 그만큼 많음) |

---

## 3. 설치 및 사용법

### 준비물
- Node.js **22 이상** (필수)
- `ripgrep` (없으면 설치 안내가 뜸)
- Bun은 **소스 빌드할 때만** 필요

### 설치

```bash
npm install -g @gitlawb/openclaude@latest
```

Arch Linux는 AUR 패키지도 있습니다: `paru -S openclaude`

### 실행 및 초기 설정

```bash
openclaude
```

실행 후 터미널 안에서:

```
/provider          # AI 프로바이더 선택 마법사 (설정이 .openclaude-profile.json 에 저장됨)
/onboard-github    # GitHub Models 온보딩
```

### 자주 쓰는 명령어

| 명령어 | 하는 일 |
| --- | --- |
| `/provider` | 프로바이더(AI 회사) 변경 |
| `/model` | 모델 변경 |
| `/cost` | 지금까지 쓴 비용 확인 |
| `/doctor` | 설정 진단 |
| `/commit` | 커밋 메시지 자동 작성 + 커밋 |
| `/init` | 프로젝트 분석해서 규칙 파일 생성 |
| `/repomap` | 코드베이스 구조 맵 확인 |
| `/buddy` | 픽셀 캐릭터 소환 |

### 백그라운드 실행

```bash
openclaude --bg "테스트 깨진 거 고쳐줘"
openclaude --bg --name auth-refactor "인증 미들웨어 리팩터링"
openclaude ps                    # 실행 중인 세션 목록
openclaude logs auth-refactor -f # 실시간 로그
openclaude kill auth-refactor    # 종료
```

### 세션 이어하기 / 분기

```bash
openclaude --resume <session-id>
openclaude --continue
openclaude --continue --fork-session   # 대화 히스토리를 새 세션으로 분기
```

### 설정 폴더 주의사항

OpenClaude는 `~/.openclaude` 와 `~/.openclaude.json` 을 사용합니다.
**`~/.claude` 나 프로젝트의 `.claude/` 를 읽지 않습니다.** 즉 Claude Code와
완전히 분리되어 있어서 같이 설치해도 서로 간섭하지 않습니다.

---

## 4. 플러그인? 스킬? MCP? — 정답: **본체**

셋 다 아닙니다. OpenClaude는 **그 셋을 전부 꽂을 수 있는 본체 프로그램**입니다.

| 개념 | 비유 | OpenClaude 지원 |
| --- | --- | --- |
| 스킬(Skill) | AI에게 주는 "레시피 종이" (지시문 문서) | ✅ `openclaude skills install` + 전용 레지스트리 |
| MCP | AI에 꽂는 "USB 장치" (외부 도구 연결) | ✅ `src/services/mcp/` 파일 39개, OAuth 포함 |
| 플러그인 | 앱에 설치하는 확장 프로그램 | ✅ `src/plugins/` |
| **OpenClaude** | **이 모든 걸 꽂는 "휴대폰 본체"** | — |

### 스킬 설치 방법

```bash
openclaude skills list                    # 설치된 스킬 목록
openclaude skills install gitlawb/ci-fix  # 레지스트리에서 설치
openclaude skills install ./my-skill      # 로컬 폴더에서 설치
openclaude skills validate ./my-skill     # 배포 전 검증
openclaude skills verify                  # 폐기(revocation) 목록 대조
```

스킬은 `SKILL.md` 파일 하나가 들어있는 폴더입니다. 레지스트리 설치 시에는
`sha256` 다이제스트 검증 + 폐기 목록 확인이 돌아가지만, **로컬 경로 설치는
그 검증을 건너뜁니다.** 남이 만든 스킬은 직접 읽어보고 설치하세요.

---

## 5. API 토큰 필요해?

**두 가지 길이 있습니다.**

### A. 유료 (API 키 필요)

OpenAI, Gemini, DeepSeek 등 클라우드 모델은 각 회사 API 키가 필요하고 쓴 만큼 과금됩니다.

```bash
export CLAUDE_CODE_USE_OPENAI=1
export OPENAI_API_KEY=sk-your-key-here
export OPENAI_MODEL=gpt-4o
openclaude
```

### B. 무료 (키 불필요) — 추천

**Ollama**로 내 컴퓨터에서 모델을 돌리면 API 키도, 비용도 필요 없습니다.

```bash
ollama pull qwen2.5-coder:7b

export CLAUDE_CODE_USE_OPENAI=1
export OPENAI_BASE_URL=http://localhost:11434/v1
export OPENAI_MODEL=qwen2.5-coder:7b
openclaude
```

> Ollama 경로에서는 네이티브 chat API를 쓰고 요청마다 32768 토큰 컨텍스트를
> 요청합니다. 바꾸려면 `OPENCLAUDE_OLLAMA_NUM_CTX` 또는 `OLLAMA_CONTEXT_LENGTH`.

### C. 계정 로그인 방식 (키 없이)

- **GitHub Models** — `/onboard-github`
- **Codex OAuth** — `/provider` → 브라우저에서 ChatGPT 로그인

### 주의

로컬 소형 모델은 여러 단계 도구 호출을 잘 못합니다. README에도 명시되어 있습니다.
도구/함수 호출을 잘하는 모델을 쓰는 게 중요합니다.

---

## 6. 왜 GitHub에서 유명할까?

1. **"Claude Code를 싸게/공짜로" 수요가 폭발적** — 제목빨이 강력합니다.
2. **지원 모델이 압도적** — OpenAI, Gemini, Ollama, Z.AI GLM, 샤오미 MiMo,
   Fireworks, Cloudflare Workers AI, NEAR AI 등 25개 이상.
   "내가 쓰는 AI도 되네?" 하고 유입됩니다.
3. **완성도가 높음** — 87만 줄, 테스트 다수, 초보자/윈도우/맥 별도 문서,
   안드로이드 설치 가이드까지 존재.
4. **파트너사 마케팅** — README 상단에 파트너 로고 10개 이상.
   AI 회사들이 "우리 모델도 지원해주세요" 하며 밀어주는 구조라 노출이 빠릅니다.
5. **바이럴 요소** — 엔터 칠 때마다 화살 쏘는 픽셀 캐릭터(`/buddy`).
   SNS에 올리기 좋은 재미 포인트.

README에 Trendshift 배지가 달려 있는데, 이는 GitHub 트렌딩에 올랐다는 표시입니다.

---

## 7. 로컬 에이전트 구축에 도움될까? → **매우 도움됨**

### A. 완제품으로 쓰기
Ollama 설치 → `openclaude` 실행. 직접 만들면 몇 달 걸릴 걸 30분에 얻습니다.

### B. 교과서로 쓰기 (이게 진짜 가치)

에이전트 만들 때 어려운 문제들의 **실제 프로덕션 정답지**가 들어있습니다.

| 어려운 문제 | 참고할 위치 |
| --- | --- |
| AI가 도구를 어떻게 골라 쓰나? | `src/tools/` (도구 50개 구현체) |
| 대화가 길어지면 토큰 초과 문제 | `src/context.ts`, `/compact` |
| 회사마다 다른 API를 어떻게 통일? | `src/services/api/` ← **핵심 보물창고** |
| 프로젝트 구조를 AI에 어떻게 알려주나? | 리포맵 (PageRank로 중요 파일 랭킹) — `docs/repo-map.md` |
| 위험한 명령 실행을 어떻게 막나? | 권한(permission) 시스템 |
| 작업별로 다른 모델 쓰기 | `agentRouting`, `agentModels` — `docs/agent-routing.md` |

### 주의
- 코드가 커서 전체 이해는 비현실적. 필요한 부분만 골라 보세요.
- **라이선스 문제로 "보고 배우기"까지만.** 복붙해서 제품에 넣으면 안 됩니다.

---

## 8. React / PHP로 만들 수 있어?

### React → **이미 React로 만들어져 있습니다**

`package.json` 확인 결과:

```json
"react": "19.2.4",
"react-reconciler": "0.33.0"
```

`Ink` 라이브러리로 **터미널 화면 자체를 React로 렌더링**하고 있습니다.
React를 안다면 UI 계층은 바로 읽을 수 있습니다.

### PHP → 가능하지만 비추천

| 필요 기능 | PHP 적합성 |
| --- | --- |
| AI API 호출 | ✅ 쉬움 (그냥 HTTP) |
| 파일 읽기/쓰기 | ✅ 쉬움 |
| 터미널 명령 실행 | ✅ 가능 (`proc_open`) |
| 실시간 스트리밍 | ⚠️ 까다로움 (요청-응답 구조라 어색) |
| 터미널 UI | ❌ 매우 힘듦 (생태계 없음) |
| MCP 연동 | ❌ 힘듦 (공식 SDK 없음) |

### 추천 아키텍처 — 역할 분담

```
🎨 React (프론트)      → 채팅 UI, 코드 뷰어
        ↕
🐘 PHP (백엔드)        → 로그인, 사용자 관리, 결제, DB   ← PHP 강점 영역
        ↕
🤖 Node.js (에이전트)  → AI 호출, 도구 실행, 파일 조작   ← 여긴 Node가 맞음
```

하나의 언어로 통일하고 싶다면 **TypeScript(Node.js)** 를 추천합니다. 생태계가 압도적입니다.

---

## 9. 라이선스 — 반드시 알아야 할 것

`LICENSE` 파일에 **프로젝트 스스로 이렇게 명시**하고 있습니다:

> "This repository contains code derived from Anthropic's Claude Code CLI."
>
> "This project does not have Anthropic's authorization to distribute
> their proprietary source. Users and contributors should evaluate their
> own legal position."

정리하면:

- OpenClaude 기여자들의 **수정분만** MIT
- 파생된 Claude Code 원본 코드는 **Anthropic 저작권 유지**
- **배포 권한 없음을 스스로 인정**

| 용도 | 판단 |
| --- | --- |
| 개인 학습 / 실험 | 괜찮음 |
| 코드 구조 참고해서 내 코드 작성 | 괜찮음 |
| 회사 제품에 탑재 / 상업적 배포 | **위험 — 하지 말 것** |
| 포크해서 이름만 바꿔 판매 | **위험 — 하지 말 것** |

README에도 명시: *"OpenClaude는 독립 커뮤니티 프로젝트이며 Anthropic과
제휴·승인·후원 관계가 없습니다."*

---

## 10. 수익화 아이디어

### 큰 그림

```
❌ 본체 만들어서 팔기      → 레드오션 + 법적 리스크
✅ 본체에 "꽂는 것" 팔기    → 블루오션, 경쟁자 거의 없음
✅ 본체 "깔아주고" 돈 받기  → 국내 수요 많음
```

---

### 🥇 1위. 도메인 특화 스킬 / MCP 판매

에이전트 본체에는 이미 "앱스토어" 구조(스킬·MCP)가 있는데,
**한국 실정에 맞는 물건이 거의 없습니다.**

| 아이템 | 타겟 | 왜 팔리나 |
| --- | --- | --- |
| 카페24 / 네이버 커머스 MCP | 쇼핑몰 개발사 | 국내 커머스 API는 외국 AI가 모름 |
| 전자정부프레임워크 스킬 | SI 업체, 공공 외주 | 공공 표준인데 AI가 전혀 모름 |
| 한국 세무/회계 검증 MCP | 세무법인, ERP | 세법이 매년 바뀜 |
| 의료 EMR 코드 분석 스킬 | 병원 IT팀 | 데이터 반출 금지 → 로컬 필수 |
| 레거시 JSP → React 마이그레이션 스킬 | 중견 SI | 국내에 해당 코드 산더미 |

**수익 모델**

```
무료 공개 (GitHub) → 유료 Pro (월 2~5만원)
   → 기업 라이선스 (연 300~1,000만원)
   → 커스터마이징 용역 (건당 500~3,000만원)  ← 실제 수익 대부분
```

**평가**

| 항목 | 평가 |
| --- | --- |
| 초기 비용 | 거의 0원 |
| 개발 기간 | 1~2주 (스킬은 사실상 잘 쓴 지시문 문서) |
| 법적 리스크 | 0 (내 코드만 사용) |
| 경쟁자 | 거의 없음 |
| 진입 장벽 | 도메인 지식 — 경력자에게 절대 유리 |

**첫 걸음**
1. 내가 가장 잘 아는 도메인 하나 선택
2. 그 분야에서 반복되는 귀찮은 작업 3개 목록화
3. 스킬 1개 만들어 GitHub 공개 → 커뮤니티 공유
4. 반응 보고 유료화 판단

---

### 🥈 2위. 폐쇄망 로컬 AI 구축 컨설팅

**시장:** 금융권, 공공기관, 방산, 병원, 대기업 연구소, 게임사
→ "소스코드를 외부에 못 올리는데 AI 코딩은 쓰고 싶다"는 수요가 많고,
이걸 해줄 수 있는 사람이 국내에 드뭅니다.

**패키지 예시**

```
기본 구축 (500~1,500만원)
  - 사내 서버에 Ollama + 에이전트 설치
  - 회사 코딩 컨벤션 학습시키기
  - 권한/보안 설정 (위험 명령 차단)
  - 개발자 교육 2회

고급 (2,000~5,000만원)
  - 위 전부 + 사내 문서/위키 연동 MCP 개발
  - CI/CD 파이프라인 연동, 팀별 커스텀 에이전트

월 유지보수 (월 100~300만원)  ← 안정적 수입원
```

**영업 비법:** 노트북에 세팅해가서 **회의실에서 와이파이를 끄고 시연**하세요.
"지금 인터넷 끊겼는데도 됩니다" — 이게 계약률을 크게 올립니다.

**주의:** OpenClaude 자체를 납품하면 안 됩니다(라이선스).
**"오픈소스 도구 세팅 + 자체 커스터마이징 + 교육"** 으로 계약하고,
납품물은 직접 만든 스킬/MCP/설정/문서여야 합니다.

---

### 🥉 3위. AI 비용 관리 SaaS

**문제:** 회사에서 AI 도구를 쓰기 시작하면 반드시 "이번 달 800만원 나왔는데
누가 뭘 한 거야?" 상황이 옵니다.

**해결:** 사용자/프로젝트별 비용 추적, 부서 예산 한도 + 초과 알림,
"이 작업은 싼 모델로 충분" 자동 추천, 월간 리포트.

**근거:** OpenClaude에 이미 씨앗이 있습니다 — `/cost`,
`src/cost-tracker.ts`, `agentRouting`(작업별 모델 배정).

**가격:** Free(5명) / Team 월 1만원×인원 / Enterprise 연 계약
→ 50명 회사 1곳 = 월 50만원, 10곳이면 월 500만원

**난이도:** 가장 높음. 서버·DB·결제·보안 필요, 3~6개월.
대신 한번 만들면 구독 수익이 지속됩니다.

---

### 🏅 4위. 콘텐츠 / 교육

한국어로 된 "AI 에이전트 만들기" 콘텐츠가 거의 없습니다.

| 채널 | 수익 | 난이도 |
| --- | --- | --- |
| 인프런/유데미 강의 | 회당 300~2,000만원 | ⭐⭐⭐ |
| 유튜브 | 광고 + 협찬 | ⭐⭐⭐⭐ |
| 기업 출강 | 시간당 30~100만원 | ⭐⭐ |
| 전자책/뉴스레터 | 소액, 꾸준 | ⭐ |

**진짜 목적은 신뢰 자산:** 글/영상 → "저 사람 잘 아네" → 컨설팅 문의 →
스킬 제품 판매. **4번은 1번·2번의 영업 엔진입니다.**

추천 주제: "인터넷 없이 AI로 코딩하는 법", "AI 코딩 비용 90% 줄인 방법",
"Claude Code는 어떻게 내 파일을 고칠까 — 소스 분석", "MCP 서버 30분 만에 만들기"

---

### ❌ 하지 말아야 할 것

| 하지 말 것 | 이유 |
| --- | --- |
| OpenClaude 포크해서 이름 바꿔 판매 | LICENSE에 배포 권한 없음 명시 → 소송 리스크 |
| 코드 복붙해서 제품에 탑재 | 위와 동일 |
| "Claude Code 무료로 쓰는 법" 유료 강의 | 상표권 + 약관 위반 소지 |
| 본체 CLI 직접 만들어 정면 경쟁 | 거대 기업들과의 싸움 |

**안전한 선:** "오픈소스를 보고 배워서, 내 코드로, 내 도메인 지식을 담아 만든다"

---

### 로드맵

```
1개월차 — 실력 쌓기 + 신뢰 만들기
  - Ollama + 로컬 에이전트 완벽 세팅
  - 시행착오를 블로그 글 2~3개로 정리
  - 수익: 0원 (투자 기간)

2~3개월차 — 첫 제품
  - 내 도메인 스킬 1개 GitHub 공개
  - 커뮤니티 공유 → 피드백
  - 데모 영상 제작
  - 수익: 소액 (대신 문의가 들어오기 시작)

4~6개월차 — 수익화 시작
  - 컨설팅 첫 계약 (500~1,500만원)
  - 스킬 Pro 유료화, 기업 출강
  - 목표: 월 300~800만원

7개월~ — 확장
  - 컨설팅 수익으로 SaaS 개발 착수
  - 안정적 구독 수익 구축
```

> ⚠️ 위 금액은 국내 SI/컨설팅 단가 기준 **추정치**이며, 경력·인맥·시장 상황에
> 따라 크게 달라질 수 있습니다. 보장된 수치가 아닙니다.

---

### 딱 하나만 고른다면

**"1번(스킬/MCP)으로 시작 → 2번(컨설팅)으로 수익화"**

1. 투자금 0원 — 실패해도 잃을 게 없음
2. 2주면 시장 반응 확인 가능
3. 법적 리스크 0
4. 도메인 지식이 그대로 경쟁 우위가 됨

대부분이 "본체 만들기"만 쳐다보고 있어서, 1번은 크게 저평가된 기회입니다.

---

## 11. 전체 요약

| 질문 | 한 줄 답 |
| --- | --- |
| 이게 뭐야? | Claude Code의 오픈소스 파생판 + 25개 이상 모델 어댑터 |
| 설치법 | `npm i -g @gitlawb/openclaude` → `openclaude` → `/provider` |
| 플러그인/스킬/MCP? | 셋 다 아님. **그 셋을 꽂는 본체** |
| API 토큰 필요? | 선택. **Ollama 쓰면 키 없이 무료** |
| 왜 유명? | 비용 절감 수요 + 모델 다양성 + 높은 완성도 + 파트너 마케팅 + 바이럴 요소 |
| 로컬 에이전트에 도움? | 완제품으로도, 교과서로도 매우 유용 |
| React/PHP 가능? | React는 **이미 사용 중**(Ink). PHP는 백엔드 역할로만 |
| 수익화 | 스킬/MCP 판매 > 로컬AI 컨설팅 > 비용 대시보드 > 콘텐츠 (본체 복제는 금지) |
| 라이선스 | 학습·개인용 OK, **상업적 배포는 위험** |
