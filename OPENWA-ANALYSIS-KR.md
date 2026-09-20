# 📱 OpenWA 전수조사 분석 정리 (한국어)

> 카리나(Claude Code)와 함께한 OpenWA 코드베이스 전수조사 + 활용/수익화 전략 정리 문서 ✨
>
> - **이 저장소(포크):** https://github.com/bmshin94/OpenWA
> - **원본 저장소(업스트림):** https://github.com/rmyndharis/OpenWA
> - **관련 링크:** [OpenWA-plugins](https://github.com/rmyndharis/OpenWA-plugins) · [whatsapp-web.js](https://github.com/pedroslopez/whatsapp-web.js) · [Baileys](https://github.com/WhiskeySockets/Baileys) · [Model Context Protocol](https://modelcontextprotocol.io)
> - **분석 기준:** `v0.23.5` / 커밋 `41d0932`
> - **작성일:** 2026-09-20

---

## 📑 목차

1. [OpenWA란 무엇인가](#1-openwa란-무엇인가)
2. [쉬운 설명 (비유 버전)](#2-쉬운-설명-비유-버전)
3. [FAQ 7문답](#3-faq-7문답)
4. [수익화 아이디어](#4-수익화-아이디어)
5. [리스크와 주의사항](#5-리스크와-주의사항)

---

## 1. OpenWA란 무엇인가

### 1.1 한 줄 정의

> **OpenWA = 내 서버에 직접 설치해서 쓰는 WhatsApp 자동화 API 게이트웨이**

유료 솔루션인 **WAHA Plus의 100% 무료 오픈소스 대체재**를 표방한다(`docs/01-project-overview.md`).
MIT 라이선스라 상업적 이용·재판매 모두 가능하다.

### 1.2 코드베이스 실측 규모

| 항목                 | 수치                                                           |
| -------------------- | -------------------------------------------------------------- |
| TypeScript 소스 파일 | 890개                                                          |
| REST API             | 158개 경로 / 200개 오퍼레이션                                  |
| 기능 모듈            | 30개                                                           |
| 공식 SDK             | 5개 언어 (JavaScript, Python, PHP, Go, Java)                   |
| 대시보드             | React 19 + Vite + TypeScript, 10개 언어(한국어 `ko.json` 포함) |
| 문서                 | `docs/` 31편 + CHANGELOG 269KB                                 |
| 설정                 | `.env.example` 59KB                                            |

### 1.3 기술 스택

| 레이어     | 기술                                                 |
| ---------- | ---------------------------------------------------- |
| 런타임     | Node.js 22 LTS                                       |
| 프레임워크 | NestJS 11.x                                          |
| 언어       | TypeScript 6.x                                       |
| WA 엔진    | `whatsapp-web.js` (기본) / `@whiskeysockets/baileys` |
| DB         | SQLite / PostgreSQL (TypeORM)                        |
| 캐시       | Redis (선택)                                         |
| 스토리지   | Local / S3 / MinIO                                   |
| 큐         | BullMQ + Bull Board                                  |
| 컨테이너   | Docker + Docker Compose + Helm(`charts/`)            |

### 1.4 아키텍처

```
[클라이언트 앱 / 워크플로]
        │ HTTP REST (X-API-Key)
        ▼
┌────────────────────────────────┐
│  OpenWA 게이트웨이 (포트 2785) │
│  ├ 세션 관리 (멀티 세션)       │
│  ├ 메시지 / 그룹 / 연락처      │
│  ├ 웹훅 (HMAC 서명 + 필터)     │
│  ├ 플러그인 샌드박스           │
│  ├ 자동응답 룰 엔진            │
│  └ MCP 서버 (AI 에이전트용)    │
└──────────────┬─────────────────┘
               │ 엔진 추상화 (교체 가능)
        ┌──────┴──────┐
        ▼             ▼
 whatsapp-web.js    Baileys
 (헤드리스 크롬)   (WebSocket 직통)
        └──────┬──────┘
               ▼
         WhatsApp 서버
```

### 1.5 엔진 2종 비교

| 엔진                     | 밴 위험 | 메모리           | 특징                                                            |
| ------------------------ | ------- | ---------------- | --------------------------------------------------------------- |
| `whatsapp-web.js` (기본) | 낮음    | 세션당 300~500MB | 실제 헤드리스 Chromium 구동 → 정상 WhatsApp Web 트래픽처럼 보임 |
| `baileys`                | 높음    | 세션당 30~80MB   | 멀티디바이스 WebSocket 직통 → 가볍지만 핑거프린팅 쉬움          |

### 1.6 폴더 구조 분석

```
src/
├── engine/          WhatsApp 엔진 추상화 (adapters/, builtin/, capability-matrix)
├── core/
│   ├── plugins/     플러그인 로더 + 샌드박스 + 매니페스트 검증
│   └── agent-tools/ AI 에이전트용 툴 정의 (message/group/contact/label/automation/webhook/session)
├── modules/         기능 모듈 30종
│   ├── mcp/         Model Context Protocol 서버
│   ├── automation/  조건 매칭 자동응답 룰 엔진 (쿨다운 지원)
│   ├── integration/ Integration Fabric (ingress / 큐 / 재전송 / 순서보장 락)
│   ├── takeover/    봇 ↔ 상담원 핸드오버 게이트
│   ├── queue/       BullMQ 작업 큐
│   ├── infra/       Docker로 Postgres/Redis/MinIO 자동 기동
│   └── (audit, auth, call, catalog, channel, chat-media, contact, docker,
│         events, group, health, label, media, message, metrics, plugins,
│         profile, search, session, settings, stats, status, template, webhook)
├── common/          캐시 / 스토리지 / 메트릭 / 보안 / 미들웨어
└── database/        TypeORM 마이그레이션

dashboard/   React 19 관리 UI (Sessions, Webhooks, ApiKeys, Chats, Templates,
             MessageTester, Logs, Plugins, Infrastructure, Dashboard)
sdk/         js · python · php · go · java 클라이언트
docs/        01~31 설계문서
charts/      Kubernetes Helm 차트
```

### 1.7 API 엔드포인트 분포 (openapi.json 실측)

| 태그                                | 오퍼레이션 수 |
| ----------------------------------- | ------------- |
| sessions                            | 28            |
| messages                            | 26            |
| groups                              | 21            |
| plugins                             | 14            |
| integration                         | 14            |
| infrastructure                      | 12            |
| contacts                            | 11            |
| channels                            | 10            |
| webhooks / labels / status          | 각 8          |
| auth                                | 7             |
| automation / templates              | 각 5          |
| profile / catalog                   | 각 4          |
| health / statistics / media         | 각 3          |
| calls                               | 2             |
| audit / settings / metrics / search | 각 1          |

### 1.8 활용 시나리오

1. 주문·배송 알림 자동 발송 (쇼핑몰)
2. 예약 확인 / 리마인더 (병원·미용실·학원)
3. 고객 상담 자동화 + 상담원 인계 (Chatwoot 연동)
4. 서버 장애 / 결제 실패 알림 채널
5. AI 챗봇 (MCP로 Claude가 직접 WhatsApp 조작)

### 1.9 도입 시 이점

- 월 구독료 0원 (WAHA Plus / Twilio 대체)
- 해외 시장(동남아·중동·남미·유럽) 공략용 필수 채널
- NestJS 모듈 설계 · 어댑터 패턴 · 플러그인 샌드박스 · 멀티 DB 추상화 학습 교재
- MCP 내장 → AI 에이전트에 "손발"을 달아주는 재료

---

## 2. 쉬운 설명 (비유 버전)

### 2.1 비유: "카톡 대신 답장해주는 알바생 고용하기"

**OpenWA 없을 때**: 손님 100명이 물어보면 사장이 직접 폰으로 타이핑.

**OpenWA 설치 후**:

```
손님 메시지 → WhatsApp 서버 → OpenWA 감지
   → 웹훅으로 "메시지 왔어요!" 신호를 내 서버에 전달
   → 내 코드가 DB 조회 후 답변 결정
   → REST API 한 번 호출 → OpenWA가 대신 답장
```

즉 **OpenWA = 내 코드와 WhatsApp 사이의 통역사 겸 알바생**.

### 2.2 핵심 개념 4가지

| 용어          | 쉬운 말                  | 비유                                |
| ------------- | ------------------------ | ----------------------------------- |
| 세션(Session) | WhatsApp 계정 1개 로그인 | 알바생 1명                          |
| API 키        | 내 서버 출입증           | 권한자만 가진 열쇠                  |
| 웹훅(Webhook) | "메시지 왔어요" 알림     | 알바생이 거는 전화                  |
| 엔진(Engine)  | WhatsApp 접속 방식       | 크롬 켜고 일하냐 vs 무전기로 일하냐 |

### 2.3 API vs 웹훅 — 방향이 반대

```
API 호출 :  나  ──"이거 보내"──▶  OpenWA    (내가 시킴)
웹훅     :  나  ◀──"이거 왔어"──  OpenWA    (얘가 알려줌)
```

이 둘이 왕복하며 챗봇이 완성된다.

### 2.4 세션 상태 흐름

```
STOPPED → (start) → SCAN_QR_CODE → (폰으로 QR 스캔) → WORKING
                                                      ↓ (로그아웃/폰 꺼짐)
                                                    FAILED
```

`WORKING` 상태부터 메시지 발송 가능.

---

## 3. FAQ 7문답

### Q1. 설치 및 사용법

**A. Docker (권장)**

```bash
git clone https://github.com/bmshin94/OpenWA.git
cd OpenWA
cp .env.minimal .env
docker compose -f docker-compose.dev.yml up -d
```

| 접속 대상                    | 주소                           |
| ---------------------------- | ------------------------------ |
| 대시보드                     | http://localhost:2785          |
| API                          | http://localhost:2785/api      |
| Swagger                      | http://localhost:2785/api/docs |
| 대시보드(dev, Vite 핫리로드) | http://localhost:2886          |

**B. 로컬 개발**

```bash
npm ci        # 락파일 그대로 설치 (의존성 변경 시에만 npm install)
npm run dev   # API 2785 + 대시보드 2886
```

**C. 실전 사용 흐름**

```bash
# 1) 세션 생성
curl -X POST http://localhost:2785/api/sessions \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"my-bot"}'

# 2) 세션 시작
curl -X POST http://localhost:2785/api/sessions/{sessionId}/start \
  -H "X-API-Key: YOUR_API_KEY"

# 3) QR 코드 조회 후 폰으로 스캔
curl http://localhost:2785/api/sessions/{sessionId}/qr \
  -H "X-API-Key: YOUR_API_KEY"

# 4) 메시지 발송
curl -X POST http://localhost:2785/api/sessions/{sessionId}/messages/send-text \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"chatId":"821012345678@c.us","text":"Hello from OpenWA!"}'

# 5) 웹훅 등록
curl -X POST http://localhost:2785/api/sessions/{sessionId}/webhooks \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"url":"https://your-server.com/webhook",
       "events":["message.received","session.status"],
       "secret":"your-hmac-secret"}'
```

**D. 프로덕션 프로파일**

```bash
docker compose up -d                      # SQLite + 로컬 스토리지
docker compose --profile postgres up -d   # + PostgreSQL
docker compose --profile redis up -d      # + Redis
docker compose --profile minio up -d      # + MinIO(S3)
docker compose --profile full up -d       # 전체
```

보안 구성: docker-socket-proxy 경유, dumb-init(PID 1) → entrypoint(root, chown) → `gosu`로 비root 실행.

---

### Q2. 플러그인? 스킬? MCP?

**정답: 본체는 "독립 서버 애플리케이션". 다만 플러그인 호스트이자 MCP 서버를 내장한다.**

| 구분     | OpenWA는           | 설명                                                                  |
| -------- | ------------------ | --------------------------------------------------------------------- |
| 본체     | 독립 서버          | Docker로 구동하는 NestJS 앱. Claude와 무관하게 단독 존재              |
| 플러그인 | 호스트(꽂는 쪽)    | `src/core/plugins/` — Chatwoot/Typebot을 샌드박스 플러그인으로 수용   |
| 스킬     | 아님               | Claude Skill(SKILL.md) 개념은 없음                                    |
| MCP      | 내장 기능으로 제공 | `MCP_ENABLED=true` → `POST /mcp` 에 Streamable-HTTP 트랜스포트 마운트 |

**MCP 설정 예시** (`.mcp.json`)

```json
{
  "mcpServers": {
    "openwa": {
      "type": "http",
      "url": "http://localhost:2785/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

- 기본: 읽기 전용 툴 **25개**
- `MCP_READONLY=false`: 전체 **51개** (발송/답장/그룹 조작 포함)
- `MCP_RATE_LIMIT_MAX` 기본 60회 / `MCP_RATE_LIMIT_WINDOW_MS` 기본 60000ms
- REST와 동일한 API 키 인증 · 역할 · 세션 스코프 적용

---

### Q3. API 토큰을 사용해야 하나

외부 유료 토큰(OpenAI/Anthropic 등)은 **불필요**. OpenWA가 자체 발급하는 API 키만 있으면 된다.

| 키               | 용도                      | 설정 위치           |
| ---------------- | ------------------------- | ------------------- |
| `API_MASTER_KEY` | 최초 부트스트랩 마스터 키 | `.env`              |
| 발급 키          | 앱/에이전트가 실제 사용   | 대시보드 → API Keys |

**역할 체계**

| 역할       | 권한                                                    |
| ---------- | ------------------------------------------------------- |
| `ADMIN`    | 전체 + 키 관리 (대시보드에서는 세션 스코프 없음)        |
| `OPERATOR` | 발송·세션 조작, `allowedSessions`로 세션 범위 제한 가능 |
| `VIEWER`   | 읽기 전용, 세션 범위 제한 가능                          |

**보안 권고**

- AI 에이전트용은 `OPERATOR` 이하 + 특정 세션 스코프 전용 키 발급
- 평문 키는 생성 시 1회만 노출 → 회전은 신규 발급 후 구 키 삭제
- MCP용 키에는 `allowedIps` 설정 금지 (MCP에는 실제 클라이언트 IP가 없어 거부됨)
- 웹훅에는 HMAC `secret` 필수 설정
- `/mcp`는 인증 프록시 없이 공개 인터넷에 노출 금지

> AI 답변 생성을 붙이려면 그때 별도로 Claude API 키 등이 필요하다. OpenWA 자체의 요구사항은 아니다.

---

### Q4. 왜 GitHub에서 유명할까

1. **거대한 시장** — WhatsApp MAU 20억+. 인도·브라질·인도네시아·멕시코·유럽·중동의 국민 메신저.
2. **유료 제품의 무료 대체재** — "WAHA Plus의 무료 대안" 포지셔닝. GitHub에서 반복 검증된 흥행 공식.
3. **공식 API의 진입장벽** — Meta Cloud API는 사업자 인증 + 템플릿 승인 + 건당 과금. OpenWA는 QR 스캔 한 번.
4. **비정상적으로 높은 완성도** — 890개 소스에 `.spec.ts` 테스트 동반, CI/CD, Helm 차트, Trivy 스캔, OpenAPI 자동 검증, SDK 라우트 커버리지 검사 스크립트, 31편 문서, 10개 언어 UI.
5. **정직한 문서** — README가 밴 리스크·개인번호 금지·규제 산업 부적합을 명시. 개발자 커뮤니티의 신뢰를 얻는 요소.
6. **AI 타이밍** — MCP 내장으로 "Claude가 WhatsApp을 직접 조작"이라는 최신 키워드에 정확히 부합.

> 참고: 본 분석 세션은 포크 저장소만 접근 가능해 실제 스타 수치는 확인하지 못했다. 위 항목은 코드/문서 근거 기반 해석이다.

---

### Q5. 로컬 에이전트 구축에 도움이 될까

**결론: 매우 도움 된다.**

1. **MCP 서버 완제품 내장** — `MCP_ENABLED=true` 한 줄로 Claude Code / Cursor / Claude Desktop에 툴 51개 노출. MCP 서버를 직접 구현할 필요 없음.
2. **에이전트 툴 설계의 교과서** — `src/core/agent-tools/`:
   - `tool-descriptor.ts` — 툴 스키마 정의
   - `tool-registry.service.ts` — 툴 등록/발견
   - `tool-invoker.ts` — 실행 + 에러 처리
   - `tool-output-sanitisation.spec.ts` — 출력 정제(에이전트 컨텍스트 오염 방지)
   - `tool-input-caps.spec.ts` — 입력 상한(토큰 폭발 방지)
   - `session-scope-invariant.spec.ts` — 권한 불변식 검증
3. **안전 기본값 패턴** — 기본 읽기 전용 + 쓰기는 옵트인 + 레이트리밋. "기본은 안전, 위험은 명시적 opt-in".
4. **에이전트에 실제 액션 부여** — 텍스트만 출력하던 에이전트가 현실 세계에 메시지를 보낼 수 있게 된다.

주의: 쓰기 권한 부여 시 **사람 승인 단계(HITL)** 를 반드시 끼울 것. 발송된 메시지는 되돌릴 수 없다.

---

### Q6. 수익화 아이디어

→ [4장 수익화 아이디어](#4-수익화-아이디어) 참조.

---

### Q7. React나 PHP로 만들 수 있을까

**이미 둘 다 포함되어 있다.**

**React** — `dashboard/` 가 React 19 + Vite + TypeScript.

```
dashboard/src/pages/  Sessions, Webhooks, ApiKeys, Chats, Templates,
                      MessageTester, Logs, Plugins, Infrastructure, Dashboard
스택: @tanstack/react-query, @tanstack/react-table, react-router-dom 7,
      recharts, socket.io-client, i18next(10개 언어), lucide-react
```

**PHP** — `sdk/php/` 에 공식 SDK (PHP 8.1+, Guzzle 7, PSR-4, PHPUnit).

```php
use OpenWA\Client;

$client = new Client('http://localhost:2785', 'YOUR_API_KEY');
$client->messages()->sendText($sessionId, [
    'chatId' => '821012345678@c.us',
    'text'   => 'Hello from PHP!',
]);
```

예외 클래스가 세분화되어 있다: `OpenWAAuthException`, `OpenWARateLimitException`,
`OpenWANotFoundException`, `OpenWAForbiddenException`, `OpenWATimeoutException`,
`OpenWAConflictException`, `OpenWAServiceUnavailableException`, `OpenWANotImplementedException`.

**만들 수 있는 것들**

| 만들 것                           | 언어       | 난이도                  |
| --------------------------------- | ---------- | ----------------------- |
| 커스텀 발송 UI                    | React      | 쉬움 (REST 호출만)      |
| 기존 대시보드 커스터마이징/한글화 | React      | 쉬움 (`ko.json` 존재)   |
| 웹훅 수신 + 자동응답 봇           | PHP        | 보통                    |
| 워드프레스 플러그인화             | PHP        | 보통 (시장성 큼)        |
| 라라벨 기반 고객관리 SaaS         | PHP        | 높음                    |
| OpenWA 코어 기능 추가             | TypeScript | 높음 (NestJS 숙련 필요) |

> 핵심: OpenWA는 **REST API 서버**이므로 **언어에 무관**하다. WhatsApp 프로토콜을 몰라도 HTTP만 알면 된다.

---

## 4. 수익화 아이디어

> 전제: 비공식 API이므로 계정 밴 가능성이 상시 존재한다. 고객에게 사전 고지하고 대체 채널(SMS/이메일)을 계약에 포함해야 분쟁을 피할 수 있다. 규제 산업(의료·금융·EU GDPR 대상)은 공식 Cloud API로 안내한다.

### 4.1 구축 대행 서비스 — 난이도 ★☆☆☆☆ / 수익 ★★★☆☆

| 항목      | 내용                                                |
| --------- | --------------------------------------------------- |
| 타깃      | 해외 쇼핑몰, 국내 기업 해외영업팀, 무역회사, 유학원 |
| 단가      | 구축 50~200만원 + 유지보수 월 10~30만원             |
| 원가      | VPS 월 2~3만원 (2GB RAM 기준)                       |
| 착수 시점 | 즉시 가능                                           |

개발자에게는 `docker compose up` 한 줄이지만, 일반 사업자에게는 Docker/서버/도메인/SSL/모니터링 전부가 진입장벽이다.

**패키지 예시**

```
[스타터  80만원] 설치 + 세션 1개 + 기본 자동응답 + 3개월 지원
[비즈니스 200만원] 세션 3개 + 웹훅 연동 + 대시보드 한글화 + CRM 연동
[월 구독  30만원] 서버 관리 + 모니터링 + 밴 발생 시 재연결 + 업데이트
```

### 4.2 멀티테넌트 SaaS — 난이도 ★★★★☆ / 수익 ★★★★★

| 항목       | 내용                                                       |
| ---------- | ---------------------------------------------------------- |
| 모델       | 월 구독 $29 / $99 / $299 (세션 수·메시지 수 기준)          |
| 100고객 시 | 월 $2,900+ 순환 매출                                       |
| 준비도     | `docs/28-multitenancy.md` 에 설계 초안 존재 (미구현 Draft) |

**구현 로드맵**

```
1단계  tenants 테이블 + 전 엔티티에 tenantId 추가
       (기존 "API 키 + allowedSessions" 스코프 구조가 확장 기반)
2단계  api-key.guard.ts 에 테넌트 검증 삽입 (문서가 훅 포인트를 명시)
3단계  결제(Stripe/토스) + 사용량 미터링
4단계  세션별 컨테이너 격리 (docker 모듈 재활용)
```

### 4.3 업종 특화 봇 — 난이도 ★★☆☆☆ / 수익 ★★★★☆ (추천 1순위)

| 업종           | 기능                         | 단가     |
| -------------- | ---------------------------- | -------- |
| 해외 병원/치과 | 예약 확인·노쇼 방지 리마인더 | 월 $99~  |
| 유학원/어학원  | 상담 자동응답·서류 안내      | 월 $79~  |
| 무역/물류      | 선적·통관 상태 알림          | 월 $199~ |
| 게스트하우스   | 체크인 안내·다국어 응대      | 월 $59~  |
| 해외직구 셀러  | 주문·배송·CS 자동화          | 월 $99~  |

**조합**: OpenWA(automation 룰 엔진) + Typebot(시나리오) + Chatwoot(상담 인계) — 셋 다 플러그인으로 지원된다.
노쇼 1건 방지 가치가 월 구독료를 상회하므로 ROI 설득이 쉽다.

### 4.4 플러그인 / 커넥터 판매 — 난이도 ★★★☆☆ / 수익 ★★★☆☆

플러그인 아키텍처(`docs/19-plugin-architecture.md`, `docs/30-plugin-sandboxing.md`)가 이미 존재한다.

| 만들 것                                     | 시장                             |
| ------------------------------------------- | -------------------------------- |
| 워드프레스 플러그인 (WooCommerce 주문 알림) | WP 유료 플러그인 시장            |
| Shopify / 카페24 앱                         | 앱스토어 등록 → 수동 영업 불필요 |
| n8n / Make 커스텀 노드                      | 기존 n8n 지원(`docs/22`) 확장    |
| 라라벨 패키지                               | PHP SDK 래핑                     |

PHP 역량이 있다면 워드프레스 플러그인이 코드 대비 시장 규모 면에서 가성비가 가장 높다.

### 4.5 AI 에이전트 결합 상품 — 난이도 ★★★☆☆ / 수익 ★★★★☆

```
고객 문의 → OpenWA 웹훅 → Claude(상품 DB RAG) → 답변 생성
         → 확신도 높으면 자동 발송 / 낮으면 상담원 인계(takeover 모듈)
```

- 차별화 포인트: `handover-gate` 가 구현되어 있어 AI → 사람 인계가 자연스럽다.
- 과금: 기본료 + AI 응답 건당 (LLM API 원가 위에 마진).

### 4.6 콘텐츠 / 교육 — 난이도 ★☆☆☆☆ / 수익 ★★☆☆☆

| 형태                        | 수익                          |
| --------------------------- | ----------------------------- |
| 유튜브 "WhatsApp 봇 만들기" | 광고 + 구축 문의 유입         |
| 인프런/유데미 강의          | 5~15만원 × N                  |
| 유료 노션 가이드            | 3~5만원                       |
| 블로그 SEO                  | 4.1 대행 서비스의 리드 깔때기 |

한국어 자료가 희소해 선점 효과가 크다.

### 4.7 추천 실행 루트

```
[1개월]  4.1 구축 대행 + 4.6 콘텐츠 → 현금흐름 확보 + 노하우 축적
[3개월]  4.3 업종 특화 봇          → 반복 가능한 템플릿으로 마진 상승
[6개월]  4.5 AI 결합               → 차별화
[1년+]   4.2 멀티테넌트 SaaS       → 스케일
```

---

## 5. 리스크와 주의사항

### 5.1 반드시 지킬 것

- **개인 / 주력 사업 번호 연결 금지.** 잃어도 되는 전용 번호만 사용.
- **신규 번호 워밍업.** 초기 며칠은 사람처럼 사용(지인과 대화, 그룹 참여, 프로필 사진 설정).
- **콜드 대량 발송 금지.** 한 번도 대화한 적 없는 번호에 일괄 첫 메시지 = 가장 확실한 제재 경로.
- **레이트리밋 사용.** `RATE_LIMIT_*` 환경변수 설정. 분당 수 건 수준이 지속 가능.
- **옵트인 수신자 중심.** OTP, 주문 상태, 지원 답장 등 기대된 메시지가 가장 안전.
- **폴백 채널 유지.** 인증/매출에 직결되는 흐름을 비공식 클라이언트에만 의존하지 말 것.
- **호스팅 IP 주의.** 저가 데이터센터 IP는 더 공격적으로 플래깅된다. 세션별 프록시 설정 지원.

### 5.2 플랫폼 동작 (버그 아님)

- 신규 연락처에 보내는 첫 메시지가 전달되지 않을 수 있다(WhatsApp 서버 측 정책, 업스트림 이슈 #830).
- 제재된 계정은 OpenWA 측에서 해제할 수 없다. WhatsApp 채널로 이의 제기해야 한다.

### 5.3 컴플라이언스

윤리·법률·규제 준수가 중요한 배포(의료, 금융, 대규모 상업 메시징, EU/EEA DMA·GDPR 대상)에서는
OpenWA를 **미승인**으로 간주하고 Meta 공식 WhatsApp Cloud API를 사용해야 한다.
OpenWA는 개인 프로젝트·사내 도구·자동화 취미·학습용으로는 훌륭하지만, 규제 환경에서 공식 API의
드롭인 대체재는 아니다.

상세 리스크 분석: [`docs/16-risk-management.md`](./docs/16-risk-management.md)

---

## 📎 참고 문서

| 문서                                                               | 설명                  |
| ------------------------------------------------------------------ | --------------------- |
| [docs/01-project-overview.md](./docs/01-project-overview.md)       | 프로젝트 개요         |
| [docs/03-system-architecture.md](./docs/03-system-architecture.md) | 시스템 아키텍처       |
| [docs/04-security-design.md](./docs/04-security-design.md)         | 보안 설계             |
| [docs/06-api-specification.md](./docs/06-api-specification.md)     | API 명세              |
| [docs/16-risk-management.md](./docs/16-risk-management.md)         | 리스크 관리           |
| [docs/18-sdk-design.md](./docs/18-sdk-design.md)                   | SDK 설계              |
| [docs/19-plugin-architecture.md](./docs/19-plugin-architecture.md) | 플러그인 아키텍처     |
| [docs/24-mcp-integration.md](./docs/24-mcp-integration.md)         | MCP 연동              |
| [docs/25-integration-fabric.md](./docs/25-integration-fabric.md)   | Integration Fabric    |
| [docs/28-multitenancy.md](./docs/28-multitenancy.md)               | 멀티테넌시 설계(초안) |
| [docs/30-plugin-sandboxing.md](./docs/30-plugin-sandboxing.md)     | 플러그인 샌드박싱     |

---

<div align="center">
<sub>🔗 https://github.com/bmshin94/OpenWA · 카리나와 함께 정리한 OpenWA 분석 문서 ✨</sub>
</div>
