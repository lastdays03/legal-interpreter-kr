# 법률 해석 서비스 설계 문서

**프로젝트명**: 법안해석기 (Legal Interpreter)
**작성일**: 2026-02-01
**버전**: 1.0

## 1. 프로젝트 개요

### 1.1 목적
입법 예고/진행 중인 법률을 일반 국민이 쉽게 이해할 수 있도록 LLM 기반 해석을 제공하는 웹 서비스

### 1.2 핵심 가치
- **접근성**: 법률 지식이 없는 일반인도 새로운 법안의 내용을 쉽게 이해
- **투명성**: 법안 발의자 정보와 사회적 영향을 명확하게 제시
- **신속성**: 국회 의안정보시스템과 실시간 연동하여 최신 법안 정보 제공

### 1.3 주요 사용자
- 법률 지식이 없는 일반 국민
- 새로 발의되는 법안이 자신의 삶에 미치는 영향을 알고 싶은 사람들

### 1.4 MVP 범위
- 국회 의안정보시스템 API 연동
- 온디맨드 LLM 분석 (캐싱 포함)
- 조문 해석 + 사회적 영향 + 찬반 의견 제공
- 발의자 정보 표시
- 검색 및 최근 법안 목록

### 1.5 향후 확장 가능성
- 사용자 프로필 기반 맞춤형 영향 분석
- 법안 알림 구독 기능
- 커뮤니티 의견 수렴

---

## 2. 시스템 아키텍처

### 2.1 기술 스택

| 영역 | 기술 | 선택 이유 |
|------|------|-----------|
| Frontend | Next.js 15 (App Router), React, TailwindCSS | 빠른 개발, SSR/SSG 지원, Vercel 최적화 |
| Backend | Next.js API Routes | 서버리스 아키텍처, 간단한 배포 |
| Database | Supabase (PostgreSQL) | 실시간 기능, RLS 보안, 무료 티어 |
| LLM | OpenAI GPT-4 Turbo | 한국어 법률 해석 성능, 안정성 |
| Deployment | Vercel | Next.js 최적화, 자동 배포, Edge Functions |
| External API | 국회 의안정보시스템 Open API | 공식 법안 데이터 |

### 2.2 데이터 흐름

```
사용자 요청
    ↓
Next.js 페이지
    ↓
API Route 호출
    ↓
Supabase 캐시 확인
    ↓
[캐시 있음] → 즉시 반환
[캐시 없음] → 국회 API 호출 → LLM 분석 → Supabase 저장 → 반환
```

### 2.3 시스템 구성도

```
┌─────────────────┐
│   사용자 브라우저   │
└────────┬────────┘
         │
    ┌────▼────┐
    │ Vercel  │
    │ (Next.js)│
    └────┬────┘
         │
    ┌────▼─────────────────┐
    │  API Routes          │
    │  - /api/bills/*      │
    │  - /api/assembly/*   │
    └─┬────────────────┬───┘
      │                │
┌─────▼─────┐    ┌────▼──────┐
│ Supabase  │    │ OpenAI    │
│ (캐시/저장)│    │ (LLM 분석)│
└───────────┘    └───────────┘
      │
┌─────▼────────────┐
│ 국회 의안정보시스템 │
│ Open API         │
└──────────────────┘
```

---

## 3. 데이터베이스 설계 (Supabase)

### 3.1 테이블 스키마

#### `bills` (법안 기본 정보)
```sql
CREATE TABLE bills (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id VARCHAR(50) UNIQUE NOT NULL,  -- 국회 의안번호
  bill_name TEXT NOT NULL,              -- 법안명
  proposer_type VARCHAR(20),            -- 의원발의/정부제출
  propose_date DATE,                    -- 발의일
  status VARCHAR(20),                   -- 발의/계류중/통과/폐기
  committee VARCHAR(100),               -- 소관위원회
  summary TEXT,                         -- 법안 요지
  bill_url TEXT,                        -- 국회 의안정보 링크
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_bills_bill_id ON bills(bill_id);
CREATE INDEX idx_bills_propose_date ON bills(propose_date DESC);
```

#### `proposers` (발의자 정보)
```sql
CREATE TABLE proposers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id UUID REFERENCES bills(id) ON DELETE CASCADE,
  member_name VARCHAR(50) NOT NULL,     -- 의원명
  party VARCHAR(50),                    -- 소속 정당
  is_representative BOOLEAN DEFAULT FALSE, -- 대표발의자 여부
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_proposers_bill_id ON proposers(bill_id);
```

#### `analyses` (LLM 분석 결과 - 캐시)
```sql
CREATE TABLE analyses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id UUID REFERENCES bills(id) ON DELETE CASCADE UNIQUE,
  easy_interpretation JSONB,            -- 조문별 쉬운 해석 배열
  social_impact TEXT,                   -- 사회적 영향 분석
  pros_cons JSONB,                      -- {pros: string, cons: string}
  llm_model VARCHAR(50),                -- 사용한 LLM 모델명
  analyzed_at TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_analyses_bill_id ON analyses(bill_id);
```

### 3.2 인덱스 전략
- `bills.bill_id`: 중복 방지 및 빠른 조회
- `bills.propose_date`: 최신순 정렬
- `proposers.bill_id`: 조인 최적화
- `analyses.bill_id`: 캐시 조회 최적화

### 3.3 캐싱 전략
1. 법안 상세 조회 시 `analyses` 테이블 확인
2. 캐시 있음 → 즉시 반환 (200ms 이내)
3. 캐시 없음 → LLM 분석 시작 → 완료 후 저장 → 반환

---

## 4. API 설계

### 4.1 엔드포인트 목록

#### `GET /api/bills/search`
법안 검색

**Query Parameters**:
- `q` (string): 검색어 (법안명, 의원명)
- `startDate` (date): 발의일 시작
- `endDate` (date): 발의일 종료
- `status` (string): 법안 상태 필터
- `page` (number): 페이지 번호 (default: 1)
- `limit` (number): 페이지당 항목 수 (default: 20)

**Response**:
```json
{
  "bills": [
    {
      "id": "uuid",
      "bill_id": "2101234",
      "bill_name": "근로기준법 일부개정법률안",
      "proposer_type": "의원발의",
      "propose_date": "2026-01-15",
      "status": "계류중",
      "committee": "환경노동위원회",
      "hasAnalysis": true
    }
  ],
  "total": 150,
  "page": 1,
  "totalPages": 8
}
```

#### `GET /api/bills/[billId]`
특정 법안 상세 정보

**Response**:
```json
{
  "bill": {
    "id": "uuid",
    "bill_id": "2101234",
    "bill_name": "근로기준법 일부개정법률안",
    "summary": "주 52시간 근로시간 단축...",
    "bill_url": "https://..."
  },
  "proposers": [
    {
      "member_name": "홍길동",
      "party": "국민의힘",
      "is_representative": true
    },
    {
      "member_name": "김철수",
      "party": "더불어민주당",
      "is_representative": false
    }
  ],
  "hasAnalysis": true
}
```

#### `POST /api/bills/[billId]/analyze`
법안 LLM 분석 (온디맨드 생성)

**Request Body**: 없음

**Response** (Streaming 가능):
```json
{
  "easy_interpretation": [
    {
      "article": "제1조",
      "original": "이 법은 근로조건의 기준을...",
      "simplified": "이 법은 일하는 사람들의 최소한의 권리를..."
    }
  ],
  "social_impact": "주 52시간제 도입으로 근로자의 삶의 질이...",
  "pros_cons": {
    "pros": "근로자 건강권 보장, 일자리 창출 가능성",
    "cons": "중소기업 인건비 부담 증가, 생산성 저하 우려"
  },
  "analyzed_at": "2026-02-01T10:30:00Z"
}
```

**처리 로직**:
1. `analyses` 테이블에서 캐시 확인
2. 캐시 있음 → 즉시 반환
3. 캐시 없음:
   - 동시 요청 방지 lock 설정
   - GPT-4 API 호출 (3단계 프롬프트)
   - 결과를 Supabase에 저장
   - lock 해제 후 반환

#### `GET /api/bills/recent`
최근 발의 법안 목록

**Query Parameters**:
- `limit` (number): 항목 수 (default: 20)

**Response**: `/api/bills/search`와 동일

#### `GET /api/assembly/bills` (내부용)
국회 의안정보시스템 API 래퍼

**목적**:
- CORS 문제 해결
- API Key 서버 측 관리
- 응답 캐싱 및 에러 처리

---

## 5. LLM 분석 프롬프트 전략

### 5.1 분석 단계

#### 1단계: 조문별 쉬운 해석
```
당신은 법률 전문가입니다. 아래 법률 조문을 초등학생도 이해할 수 있도록
쉬운 말로 풀어서 설명해주세요.

조문: {article_text}

출력 형식:
- 핵심 내용을 3문장 이내로 요약
- 법률 용어는 일상 용어로 변환
- 구체적인 예시 포함
```

#### 2단계: 사회적 영향 분석
```
이 법안이 통과되었을 때 대한민국 사회에 미칠 영향을 분석해주세요.

법안 요지: {summary}

다음 관점에서 설명:
1. 일반 국민의 일상생활 변화
2. 경제적 영향 (소비자, 기업)
3. 장기적 사회 변화

300자 이내로 작성
```

#### 3단계: 찬반 의견 정리
```
이 법안에 대한 예상되는 찬성/반대 입장을 공정하게 정리해주세요.

찬성 입장:
- 주요 논리 2-3가지
- 기대 효과

반대 입장:
- 주요 우려 2-3가지
- 예상 부작용

각 입장당 150자 이내
```

### 5.2 LLM 설정
- Model: `gpt-4-turbo-preview`
- Temperature: 0.3 (일관성 있는 해석)
- Max Tokens: 2000
- 예상 비용: 법안당 약 $0.05-0.10

---

## 6. 페이지 구조 및 UI/UX

### 6.1 페이지 라우팅

```
app/
├── page.tsx              # 홈 페이지
├── bills/
│   └── [billId]/
│       └── page.tsx      # 법안 상세
└── search/
    └── page.tsx          # 검색 결과
```

### 6.2 홈 페이지 (`/`)

**구성**:
- Hero 섹션: "새로 발의되는 법, 쉽게 이해하기"
- 검색 바 (자동완성 지원)
- 최근 발의 법안 카드 리스트 (20건)

**법안 카드 정보**:
- 법안명 (클릭 시 상세 페이지 이동)
- 대표발의자 (이름, 정당)
- 발의일
- 상태 뱃지 (발의/계류중/통과/폐기)
- 분석 상태: "분석 완료" or "분석하기" 버튼

### 6.3 법안 상세 페이지 (`/bills/[billId]`)

#### 상단: 법안 기본 정보
- 법안명 (h1)
- 의안번호, 발의일, 상태
- 소관위원회
- 국회 원문 보기 링크

#### 발의자 정보 카드
- **대표발의**: 이름(정당) - 강조 표시
- **공동발의**: 이름(정당) 리스트
  - 5명 이상일 경우 "펼쳐보기" 버튼

#### 분석 섹션

**분석 전**:
- "AI로 쉽게 해석하기" 버튼
- 예상 소요 시간 안내: "약 20초 소요됩니다"

**분석 중**:
- Skeleton UI
- 진행 메시지: "AI가 법안을 분석하고 있습니다..."
- (선택) Streaming으로 실시간 텍스트 표시

**분석 완료**:
3개 탭 구성

**탭 1: 쉬운 해석**
- 조문별 아코디언
- 각 조문마다:
  - 원문 (작은 폰트, 회색)
  - 쉬운 풀이 (큰 폰트, 강조)

**탭 2: 사회적 영향**
- 이 법이 통과되면 우리 사회에 어떤 변화가 생길까?
- 일반 국민 관점의 영향 설명
- 아이콘과 함께 시각화

**탭 3: 찬반 의견**
- 좌우 2단 레이아웃
- 왼쪽: 찬성 입장 (녹색 테마)
- 오른쪽: 반대 입장 (빨간색 테마)
- 각 입장의 주요 논리 정리

### 6.4 검색 결과 페이지 (`/search`)

**필터**:
- 발의일 범위 선택
- 상태 (발의/계류중/통과/폐기)
- 소관위원회 선택

**결과 리스트**:
- 홈 페이지 카드와 동일한 형태
- 페이지네이션 (20건/페이지)

### 6.5 UI 컴포넌트 라이브러리
- **shadcn/ui**: 기본 컴포넌트
- **Lucide Icons**: 아이콘
- **TailwindCSS**: 스타일링

---

## 7. 에러 처리 및 엣지 케이스

### 7.1 국회 API 호출 실패

**원인**:
- API 서버 다운
- 네트워크 오류
- Rate Limit 초과

**처리**:
1. 3회 재시도 (exponential backoff: 1초 → 2초 → 4초)
2. 실패 시 사용자에게 Toast 메시지:
   - "국회 서버와 연결할 수 없습니다. 잠시 후 다시 시도해주세요."
3. Supabase에 캐시된 데이터가 있으면:
   - "최근 저장된 정보를 표시합니다" 배너와 함께 제공

### 7.2 LLM API 호출 실패

**원인**:
- OpenAI API 오류
- Timeout (30초 초과)
- Rate Limit 초과

**처리**:
1. 명확한 에러 메시지:
   - "AI 분석 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요."
2. 부분 완료된 분석은 저장하지 않음 (원자성 보장)
3. "다시 분석하기" 버튼 제공
4. 에러 로그를 Vercel Analytics로 전송

### 7.3 동시 요청 중복 분석 방지

**문제**:
여러 사용자가 동시에 같은 법안 분석을 요청하면 LLM API 중복 호출 발생

**해결 방법**:
```typescript
// Supabase에 임시 lock 테이블 사용
1. 분석 시작 시 bill_id로 lock 레코드 생성
2. 이미 lock이 있으면:
   - "다른 사용자가 분석 중입니다" 상태 반환
   - 클라이언트는 5초마다 polling하여 완료 확인
3. 분석 완료 시 lock 삭제
4. lock 타임아웃: 60초 (강제 해제)
```

### 7.4 법안 데이터 없음

**케이스**:
- 사용자가 잘못된 의안번호 입력
- 너무 오래된 법안 (데이터 없음)

**처리**:
- 404 페이지 대신 안내 페이지:
  - "해당 법안을 찾을 수 없습니다"
  - "최근 발의된 법안 보기" 버튼
  - 검색 바 제공

### 7.5 LLM 비용 폭증 방지

**제한 사항**:
1. IP별 일일 분석 요청 제한: 10건/일
   - Vercel Edge Middleware + Upstash Redis 사용
2. 초과 시 안내 메시지:
   - "오늘의 분석 한도를 초과했습니다. 내일 다시 시도해주세요."
   - 이미 분석된 법안은 제한 없이 조회 가능
3. 법안 1개당 분석 비용 추적
   - OpenAI Usage API로 모니터링
4. 월 예산 초과 시 관리자 알림 (이메일)

### 7.6 사용자 피드백

**로딩 상태**:
- 예상 소요 시간 표시: "약 20초 소요됩니다"
- Skeleton UI로 로딩 중임을 명확히 표시

**진행 상태**:
- (선택) Streaming으로 실시간 분석 텍스트 표시
- 진행률 표시: "조문 해석 중... (1/3)"

**에러 발생 시**:
- 구체적인 에러 메시지
- 해결 방법 제시
- 대안 제공 (캐시 데이터 표시 등)

---

## 8. 보안 및 배포

### 8.1 보안 고려사항

#### API Key 보호
- OpenAI API Key는 절대 클라이언트 노출 금지
- Next.js API Routes에서만 사용
- 환경변수로 관리 (Vercel Environment Variables)

```env
# .env.local (로컬 개발)
OPENAI_API_KEY=sk-...
ASSEMBLY_API_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...

# Vercel에서 동일하게 설정
```

#### Rate Limiting
```typescript
// middleware.ts (Vercel Edge Middleware)
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "1 d"), // 10 requests per day
  analytics: true,
});

export async function middleware(request: Request) {
  const ip = request.headers.get("x-forwarded-for") ?? "anonymous";
  const { success } = await ratelimit.limit(ip);

  if (!success) {
    return new Response("Rate limit exceeded", { status: 429 });
  }
}
```

#### Supabase Row Level Security (RLS)
```sql
-- analyses 테이블: 모든 사용자 읽기 가능
CREATE POLICY "Public read access on analyses"
ON analyses FOR SELECT
USING (true);

-- 쓰기는 서버만 가능 (service_role key 사용)
-- RLS 정책 없음 = 기본 차단
```

#### CORS 및 CSP
- 국회 API 호출은 서버 사이드에서만 (CORS 우회)
- Content Security Policy 헤더 설정:

```typescript
// next.config.js
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  connect-src 'self' https://*.supabase.co https://api.openai.com;
  frame-ancestors 'none';
`;
```

### 8.2 환경 변수 설정

```env
# Public (클라이언트에서 접근 가능)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJ...

# Private (서버에서만 접근)
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJ...
OPENAI_API_KEY=sk-...
ASSEMBLY_API_KEY=...

# Upstash Redis (Rate Limiting)
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...
```

### 8.3 Vercel 배포 설정

#### 프로젝트 설정
- **Framework Preset**: Next.js
- **Root Directory**: `./`
- **Build Command**: `npm run build`
- **Output Directory**: `.next`

#### 환경 설정
- **Region**: Seoul (`icn1`) - 한국 사용자 대상
- **Node.js Version**: 20.x
- **Function Region**: `icn1` (Seoul)

#### Function 설정
```typescript
// app/api/bills/[billId]/analyze/route.ts
export const runtime = 'nodejs';
export const maxDuration = 60; // LLM 분석용 60초
```

#### 환경 분리
- **Production**: 실제 서비스 (main 브랜치)
- **Preview**: PR마다 자동 생성 (테스트용)
- **Development**: 로컬 개발

### 8.4 모니터링

#### Vercel Analytics
- 페이지뷰 추적
- Core Web Vitals 모니터링
- 사용자 흐름 분석

#### Supabase Logs
- DB 쿼리 성능 모니터링
- 슬로우 쿼리 감지
- RLS 정책 위반 추적

#### OpenAI Usage Dashboard
- LLM API 호출 횟수
- 비용 추적 (일/월별)
- 토큰 사용량 모니터링

#### 커스텀 로그
```typescript
// lib/logger.ts
export function logAnalysis(billId: string, cost: number, duration: number) {
  console.log(JSON.stringify({
    type: 'analysis',
    billId,
    cost,
    duration,
    timestamp: new Date().toISOString()
  }));
}
```

### 8.5 초기 배포 체크리스트

#### 1. Supabase 설정
- [ ] 프로젝트 생성 (Seoul region)
- [ ] 테이블 생성 (bills, proposers, analyses)
- [ ] RLS 정책 설정
- [ ] API Keys 복사 (anon key, service_role key)

#### 2. 외부 API 키 발급
- [ ] 국회 의안정보시스템 API 키 발급 (공공데이터포털)
- [ ] OpenAI API 키 발급
- [ ] 사용량 제한 설정 ($50/month)

#### 3. Upstash Redis
- [ ] 프로젝트 생성
- [ ] REST API URL 및 Token 복사

#### 4. Vercel 설정
- [ ] GitHub 연동
- [ ] 프로젝트 생성
- [ ] 환경변수 설정 (Production, Preview)
- [ ] Region을 Seoul로 설정

#### 5. 첫 배포 및 테스트
- [ ] main 브랜치에 푸시 → 자동 배포
- [ ] 배포 완료 확인
- [ ] 테스트 법안으로 End-to-End 테스트:
  1. 법안 검색
  2. 상세 페이지 이동
  3. AI 분석 실행
  4. 결과 확인
  5. 캐시 확인 (동일 법안 재조회 시 즉시 로딩)

#### 6. 모니터링 설정
- [ ] Vercel Analytics 활성화
- [ ] OpenAI Usage Alert 설정
- [ ] Supabase 알림 설정

---

## 9. 비용 추정

### 9.1 예상 사용량 (월간)

**가정**:
- 일일 방문자: 100명
- 법안 조회: 200회/일
- 신규 법안 분석: 20회/일 (나머지는 캐시 조회)

### 9.2 서비스별 비용

| 서비스 | 무료 티어 | 예상 비용 | 비고 |
|--------|-----------|-----------|------|
| Vercel | 100GB 대역폭, 100GB 시간 | $0 | 무료 범위 내 |
| Supabase | 500MB DB, 1GB 파일 저장 | $0 | 무료 범위 내 |
| OpenAI GPT-4 | - | $3-6/월 | 20건 × $0.10 × 30일 |
| Upstash Redis | 10,000 명령/일 | $0 | 무료 범위 내 |
| **총계** | - | **$3-6/월** | LLM 비용만 발생 |

### 9.3 비용 최적화 전략

1. **캐싱 극대화**: 한 번 분석한 법안은 영구 저장
2. **Rate Limiting**: IP별 일일 10건 제한으로 남용 방지
3. **모델 선택**: 간단한 법안은 GPT-3.5 Turbo 사용 고려
4. **사용량 모니터링**: 월 $50 초과 시 알림

---

## 10. 개발 일정 (예상)

### Phase 1: 기본 인프라 (3-4일)
- [ ] Next.js 프로젝트 초기 설정
- [ ] Supabase 설정 및 테이블 생성
- [ ] 국회 API 연동 테스트
- [ ] 기본 UI 컴포넌트 (shadcn/ui)

### Phase 2: 핵심 기능 (5-7일)
- [ ] 법안 검색 API 구현
- [ ] 법안 상세 페이지
- [ ] LLM 분석 API 구현
- [ ] 캐싱 로직 구현

### Phase 3: UI/UX 완성 (3-4일)
- [ ] 홈 페이지 디자인
- [ ] 검색 결과 페이지
- [ ] 분석 결과 탭 UI
- [ ] 로딩/에러 상태 처리

### Phase 4: 보안 및 최적화 (2-3일)
- [ ] Rate Limiting 구현
- [ ] 에러 처리 강화
- [ ] 성능 최적화 (캐싱, 이미지 최적화)
- [ ] SEO 설정

### Phase 5: 테스트 및 배포 (2일)
- [ ] End-to-End 테스트
- [ ] 실제 법안으로 검증
- [ ] Vercel 배포
- [ ] 모니터링 설정

**총 예상 기간**: 15-20일 (1인 개발 기준)

---

## 11. 향후 확장 계획

### Phase 2 (v2.0)
- **사용자 맞춤형 분석**: 직업, 관심 분야 기반 영향 분석
- **법안 알림**: 관심 키워드 구독 및 이메일/푸시 알림
- **배치 분석**: 주요 법안 새벽 시간대 자동 분석

### Phase 3 (v3.0)
- **커뮤니티**: 사용자 의견 수렴 및 토론
- **비교 기능**: 유사 법안 비교, 개정 전후 비교
- **AI 챗봇**: 법안에 대한 Q&A

### Phase 4 (v4.0)
- **다국어 지원**: 외국인을 위한 영어 번역
- **모바일 앱**: React Native로 네이티브 앱 개발
- **Open API 제공**: 다른 서비스에서 분석 결과 활용 가능

---

## 12. 리스크 및 대응 방안

### 12.1 기술적 리스크

| 리스크 | 영향 | 대응 방안 |
|--------|------|-----------|
| 국회 API 장애 | 높음 | 캐시 데이터 제공, 재시도 로직 |
| LLM API 비용 폭증 | 중간 | Rate Limiting, 예산 알림 |
| Supabase 무료 티어 초과 | 낮음 | 사용량 모니터링, 유료 전환 고려 |
| 법안 데이터 구조 변경 | 중간 | 국회 API 스펙 정기 확인 |

### 12.2 법률적 리스크

| 리스크 | 대응 방안 |
|--------|-----------|
| LLM 해석 오류 | 면책 조항 명시: "AI 분석 결과는 참고용이며, 정확한 법률 해석은 전문가와 상담 필요" |
| 저작권 문제 | 국회 공공데이터는 공공저작물 자유이용 허락 표시 |
| 개인정보 보호 | 사용자 개인정보 미수집 (IP 기반 Rate Limiting만 사용) |

### 12.3 운영 리스크

| 리스크 | 대응 방안 |
|--------|-----------|
| 급격한 트래픽 증가 | Vercel 자동 스케일링, CDN 캐싱 |
| 악의적 사용 (봇) | Rate Limiting, Captcha 추가 고려 |
| 유지보수 부담 | 모니터링 자동화, 에러 알림 설정 |

---

## 13. 성공 지표 (KPI)

### 초기 3개월 목표
- **MAU (월간 활성 사용자)**: 500명
- **법안 분석 수**: 100건 이상
- **재방문율**: 30% 이상
- **평균 체류 시간**: 3분 이상
- **분석 정확도**: 사용자 피드백 4.0/5.0 이상

### 측정 방법
- Vercel Analytics: 방문자 수, 페이지뷰
- Supabase: 법안 분석 수, 캐시 히트율
- 사용자 설문: 분석 품질 평가

---

## 14. 참고 자료

### API 문서
- [국회 의안정보시스템 (국회 국회사무처_의안정보 통합 API)](https://www.data.go.kr/data/15126134/openapi.do)
- [국회 의안정보시스템 (국회 국회사무처_국회의원 발의법률안)](https://www.data.go.kr/data/15125946/openapi.do)
- [국회 의안정보시스템 (국회 국회사무처_국회의원 본회의 표결정보)](https://www.data.go.kr/data/15125948/openapi.do)
- [OpenAI API Documentation](https://platform.openai.com/docs)
- [Supabase Documentation](https://supabase.com/docs)

### 기술 스택 문서
- [Next.js 15 Documentation](https://nextjs.org/docs)
- [Vercel Deployment](https://vercel.com/docs)
- [TailwindCSS](https://tailwindcss.com/docs)
- [shadcn/ui](https://ui.shadcn.com/)

### 법안 통계
- 22대 국회 법안 발의 통계: 하루 평균 25-30건
- [비즈한국 - 22대 국회의원 발의 법안 역대 최다](https://www.bizhankook.com/bk/article/28996)

---

## 15. 설계 승인 및 다음 단계

### 설계 승인
- [ ] 기술 스택 승인
- [ ] 데이터베이스 스키마 승인
- [ ] API 설계 승인
- [ ] UI/UX 플로우 승인

### 다음 단계
1. **구현 계획 수립**: 상세 구현 계획 작성 (`/pdca plan`)
2. **개발 환경 설정**: Git 저장소, 로컬 개발 환경
3. **Phase 1 개발 시작**: 기본 인프라 구축

---

**문서 버전**: 1.0
**최종 수정일**: 2026-02-01
**작성자**: AI Assistant
**검토자**: 박종만
