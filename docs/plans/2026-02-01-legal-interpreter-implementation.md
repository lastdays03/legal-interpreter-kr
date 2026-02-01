# 법률 해석 서비스 구현 계획

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Next.js 15 + Supabase + OpenAI를 활용한 법안 해석 서비스 MVP 구축

**Architecture:** 서버리스 풀스택 아키텍처 (Next.js App Router + API Routes + Supabase PostgreSQL + OpenAI GPT-4)

**Tech Stack:** Next.js 15, React, TypeScript, TailwindCSS, shadcn/ui, Supabase, OpenAI API, Vercel

---

## Task 1: 프로젝트 초기 설정

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `next.config.ts`
- Create: `tailwind.config.ts`
- Create: `.env.local.example`
- Create: `.gitignore`

**Step 1: Next.js 프로젝트 생성**

Run: `npx create-next-app@latest legal-interpreter --typescript --tailwind --app --src-dir --import-alias "@/*"`

옵션 선택:
- TypeScript: Yes
- ESLint: Yes
- Tailwind CSS: Yes
- `src/` directory: Yes
- App Router: Yes
- Import alias: @/*

**Step 2: 필요한 패키지 설치**

Run:
```bash
npm install @supabase/supabase-js openai
npm install -D @types/node
```

**Step 3: 환경 변수 템플릿 생성**

Create `.env.local.example`:
```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key

# OpenAI
OPENAI_API_KEY=your_openai_api_key

# 국회 의안정보시스템 API
ASSEMBLY_API_KEY=your_assembly_api_key
```

**Step 4: Git 초기화 및 첫 커밋**

```bash
git init
git add .
git commit -m "feat: initialize Next.js project with TypeScript and Tailwind

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Supabase 데이터베이스 설정

**Files:**
- Create: `supabase/migrations/20260201000001_create_bills_table.sql`
- Create: `supabase/migrations/20260201000002_create_proposers_table.sql`
- Create: `supabase/migrations/20260201000003_create_analyses_table.sql`
- Create: `supabase/migrations/20260201000004_create_rls_policies.sql`

**Step 1: Supabase 프로젝트 생성**

1. https://supabase.com 접속
2. 새 프로젝트 생성 (Region: Seoul)
3. API Keys 복사 (anon key, service_role key)
4. `.env.local` 파일 생성 및 키 설정

**Step 2: bills 테이블 마이그레이션 작성**

File: `supabase/migrations/20260201000001_create_bills_table.sql`
```sql
-- bills 테이블 생성
CREATE TABLE IF NOT EXISTS bills (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id VARCHAR(50) UNIQUE NOT NULL,
  bill_name TEXT NOT NULL,
  proposer_type VARCHAR(20),
  propose_date DATE,
  status VARCHAR(20),
  committee VARCHAR(100),
  summary TEXT,
  bill_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스 생성
CREATE INDEX idx_bills_bill_id ON bills(bill_id);
CREATE INDEX idx_bills_propose_date ON bills(propose_date DESC);

COMMENT ON TABLE bills IS '법안 기본 정보';
COMMENT ON COLUMN bills.bill_id IS '국회 의안번호';
COMMENT ON COLUMN bills.bill_name IS '법안명';
```

**Step 3: proposers 테이블 마이그레이션 작성**

File: `supabase/migrations/20260201000002_create_proposers_table.sql`
```sql
-- proposers 테이블 생성
CREATE TABLE IF NOT EXISTS proposers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id UUID REFERENCES bills(id) ON DELETE CASCADE,
  member_name VARCHAR(50) NOT NULL,
  party VARCHAR(50),
  is_representative BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스 생성
CREATE INDEX idx_proposers_bill_id ON proposers(bill_id);

COMMENT ON TABLE proposers IS '법안 발의자 정보';
```

**Step 4: analyses 테이블 마이그레이션 작성**

File: `supabase/migrations/20260201000003_create_analyses_table.sql`
```sql
-- analyses 테이블 생성
CREATE TABLE IF NOT EXISTS analyses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bill_id UUID REFERENCES bills(id) ON DELETE CASCADE UNIQUE,
  easy_interpretation JSONB,
  social_impact TEXT,
  pros_cons JSONB,
  llm_model VARCHAR(50),
  analyzed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스 생성
CREATE UNIQUE INDEX idx_analyses_bill_id ON analyses(bill_id);

COMMENT ON TABLE analyses IS 'LLM 분석 결과 캐시';
```

**Step 5: RLS 정책 마이그레이션 작성**

File: `supabase/migrations/20260201000004_create_rls_policies.sql`
```sql
-- RLS 활성화
ALTER TABLE bills ENABLE ROW LEVEL SECURITY;
ALTER TABLE proposers ENABLE ROW LEVEL SECURITY;
ALTER TABLE analyses ENABLE ROW LEVEL SECURITY;

-- 모든 사용자 읽기 가능 정책
CREATE POLICY "Public read access on bills"
ON bills FOR SELECT
USING (true);

CREATE POLICY "Public read access on proposers"
ON proposers FOR SELECT
USING (true);

CREATE POLICY "Public read access on analyses"
ON analyses FOR SELECT
USING (true);

-- 쓰기는 서버만 가능 (service_role key 사용)
-- RLS 정책 없음 = 기본 차단
```

**Step 6: Supabase에서 마이그레이션 실행**

Supabase Dashboard → SQL Editor에서 각 파일 내용 실행

**Step 7: 커밋**

```bash
git add supabase/
git commit -m "feat: add Supabase database schema and migrations

- Create bills, proposers, analyses tables
- Set up indexes for performance
- Configure RLS policies

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Supabase 클라이언트 설정

**Files:**
- Create: `src/lib/supabase/client.ts`
- Create: `src/lib/supabase/server.ts`
- Create: `src/types/database.types.ts`

**Step 1: 타입 정의 작성**

File: `src/types/database.types.ts`
```typescript
export type Json =
  | string
  | number
  | boolean
  | null
  | { [key: string]: Json | undefined }
  | Json[]

export interface Database {
  public: {
    Tables: {
      bills: {
        Row: {
          id: string
          bill_id: string
          bill_name: string
          proposer_type: string | null
          propose_date: string | null
          status: string | null
          committee: string | null
          summary: string | null
          bill_url: string | null
          created_at: string
          updated_at: string
        }
        Insert: {
          id?: string
          bill_id: string
          bill_name: string
          proposer_type?: string | null
          propose_date?: string | null
          status?: string | null
          committee?: string | null
          summary?: string | null
          bill_url?: string | null
          created_at?: string
          updated_at?: string
        }
        Update: {
          id?: string
          bill_id?: string
          bill_name?: string
          proposer_type?: string | null
          propose_date?: string | null
          status?: string | null
          committee?: string | null
          summary?: string | null
          bill_url?: string | null
          created_at?: string
          updated_at?: string
        }
      }
      proposers: {
        Row: {
          id: string
          bill_id: string
          member_name: string
          party: string | null
          is_representative: boolean
          created_at: string
        }
        Insert: {
          id?: string
          bill_id: string
          member_name: string
          party?: string | null
          is_representative?: boolean
          created_at?: string
        }
        Update: {
          id?: string
          bill_id?: string
          member_name?: string
          party?: string | null
          is_representative?: boolean
          created_at?: string
        }
      }
      analyses: {
        Row: {
          id: string
          bill_id: string
          easy_interpretation: Json | null
          social_impact: string | null
          pros_cons: Json | null
          llm_model: string | null
          analyzed_at: string
          created_at: string
        }
        Insert: {
          id?: string
          bill_id: string
          easy_interpretation?: Json | null
          social_impact?: string | null
          pros_cons?: Json | null
          llm_model?: string | null
          analyzed_at?: string
          created_at?: string
        }
        Update: {
          id?: string
          bill_id?: string
          easy_interpretation?: Json | null
          social_impact?: string | null
          pros_cons?: Json | null
          llm_model?: string | null
          analyzed_at?: string
          created_at?: string
        }
      }
    }
  }
}
```

**Step 2: 클라이언트 사이드 Supabase 클라이언트 작성**

File: `src/lib/supabase/client.ts`
```typescript
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database.types'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey)
```

**Step 3: 서버 사이드 Supabase 클라이언트 작성**

File: `src/lib/supabase/server.ts`
```typescript
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database.types'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseServiceKey = process.env.SUPABASE_SERVICE_ROLE_KEY!

export const supabaseAdmin = createClient<Database>(
  supabaseUrl,
  supabaseServiceKey
)
```

**Step 4: 커밋**

```bash
git add src/lib/supabase/ src/types/
git commit -m "feat: add Supabase client configuration

- Set up client-side and server-side Supabase clients
- Add database type definitions

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 4: 국회 API 클라이언트 구현

**Files:**
- Create: `src/lib/assembly/client.ts`
- Create: `src/lib/assembly/types.ts`
- Create: `src/app/api/assembly/bills/route.ts`

**Step 1: 국회 API 타입 정의 작성**

File: `src/lib/assembly/types.ts`
```typescript
// 국회 의안정보시스템 API 응답 타입
export interface AssemblyBillResponse {
  BILL_ID: string
  BILL_NAME: string
  PROPOSE_DT: string
  PROPOSER: string
  COMMITTEE: string
  PROC_RESULT: string
  LINK_URL: string
}

export interface AssemblyBillListResponse {
  nzmimeepazxkubdpuns: {
    list_total_count: number
    row: AssemblyBillResponse[]
  }
}
```

**Step 2: 국회 API 클라이언트 작성**

File: `src/lib/assembly/client.ts`
```typescript
import type { AssemblyBillListResponse } from './types'

const ASSEMBLY_API_BASE_URL = 'https://open.assembly.go.kr/portal/openapi'
const ASSEMBLY_API_KEY = process.env.ASSEMBLY_API_KEY!

export class AssemblyAPIClient {
  async searchBills(params: {
    keyword?: string
    startDate?: string
    endDate?: string
    pageIndex?: number
    pageSize?: number
  }): Promise<AssemblyBillListResponse> {
    const queryParams = new URLSearchParams({
      KEY: ASSEMBLY_API_KEY,
      Type: 'json',
      pIndex: (params.pageIndex || 1).toString(),
      pSize: (params.pageSize || 20).toString(),
    })

    if (params.keyword) {
      queryParams.append('BILL_NAME', params.keyword)
    }
    if (params.startDate) {
      queryParams.append('PROPOSE_DT_START', params.startDate)
    }
    if (params.endDate) {
      queryParams.append('PROPOSE_DT_END', params.endDate)
    }

    const url = `${ASSEMBLY_API_BASE_URL}/nzmimeepazxkubdpuns?${queryParams}`

    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
      },
    })

    if (!response.ok) {
      throw new Error(`Assembly API error: ${response.statusText}`)
    }

    return response.json()
  }

  async getBillDetail(billId: string): Promise<AssemblyBillResponse> {
    const response = await this.searchBills({ keyword: billId, pageSize: 1 })
    const bills = response.nzmimeepazxkubdpuns.row

    if (!bills || bills.length === 0) {
      throw new Error('Bill not found')
    }

    return bills[0]
  }
}

export const assemblyAPI = new AssemblyAPIClient()
```

**Step 3: 국회 API 프록시 엔드포인트 작성**

File: `src/app/api/assembly/bills/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { assemblyAPI } from '@/lib/assembly/client'

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams
    const keyword = searchParams.get('keyword') || undefined
    const startDate = searchParams.get('startDate') || undefined
    const endDate = searchParams.get('endDate') || undefined
    const pageIndex = parseInt(searchParams.get('page') || '1')
    const pageSize = parseInt(searchParams.get('limit') || '20')

    const response = await assemblyAPI.searchBills({
      keyword,
      startDate,
      endDate,
      pageIndex,
      pageSize,
    })

    return NextResponse.json(response)
  } catch (error) {
    console.error('Assembly API error:', error)
    return NextResponse.json(
      { error: 'Failed to fetch bills from Assembly API' },
      { status: 500 }
    )
  }
}
```

**Step 4: 테스트 실행**

```bash
npm run dev
# 브라우저에서 http://localhost:3000/api/assembly/bills?keyword=근로기준법 접속
# 응답 확인
```

**Step 5: 커밋**

```bash
git add src/lib/assembly/ src/app/api/assembly/
git commit -m "feat: implement Assembly API client

- Add Assembly API types and client
- Create API proxy endpoint for CORS

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 5: OpenAI 클라이언트 설정

**Files:**
- Create: `src/lib/openai/client.ts`
- Create: `src/lib/openai/prompts.ts`
- Create: `src/lib/openai/types.ts`

**Step 1: OpenAI 타입 정의 작성**

File: `src/lib/openai/types.ts`
```typescript
export interface ArticleInterpretation {
  article: string
  original: string
  simplified: string
}

export interface ProsCons {
  pros: string
  cons: string
}

export interface BillAnalysis {
  easy_interpretation: ArticleInterpretation[]
  social_impact: string
  pros_cons: ProsCons
}
```

**Step 2: 프롬프트 템플릿 작성**

File: `src/lib/openai/prompts.ts`
```typescript
export const ARTICLE_INTERPRETATION_PROMPT = (articleText: string) => `
당신은 법률 전문가입니다. 아래 법률 조문을 초등학생도 이해할 수 있도록 쉬운 말로 풀어서 설명해주세요.

조문: ${articleText}

출력 형식:
- 핵심 내용을 3문장 이내로 요약
- 법률 용어는 일상 용어로 변환
- 구체적인 예시 포함

JSON 형식으로 응답:
{
  "simplified": "쉬운 해석 내용"
}
`

export const SOCIAL_IMPACT_PROMPT = (summary: string) => `
이 법안이 통과되었을 때 대한민국 사회에 미칠 영향을 분석해주세요.

법안 요지: ${summary}

다음 관점에서 설명:
1. 일반 국민의 일상생활 변화
2. 경제적 영향 (소비자, 기업)
3. 장기적 사회 변화

300자 이내로 작성하고 JSON 형식으로 응답:
{
  "impact": "사회적 영향 분석"
}
`

export const PROS_CONS_PROMPT = (summary: string) => `
이 법안에 대한 예상되는 찬성/반대 입장을 공정하게 정리해주세요.

법안 요지: ${summary}

찬성 입장:
- 주요 논리 2-3가지
- 기대 효과

반대 입장:
- 주요 우려 2-3가지
- 예상 부작용

각 입장당 150자 이내로 작성하고 JSON 형식으로 응답:
{
  "pros": "찬성 입장",
  "cons": "반대 입장"
}
`
```

**Step 3: OpenAI 클라이언트 작성**

File: `src/lib/openai/client.ts`
```typescript
import OpenAI from 'openai'
import {
  ARTICLE_INTERPRETATION_PROMPT,
  SOCIAL_IMPACT_PROMPT,
  PROS_CONS_PROMPT,
} from './prompts'
import type { ArticleInterpretation, BillAnalysis, ProsCons } from './types'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
})

export class OpenAIClient {
  private async chat(prompt: string): Promise<string> {
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
      max_tokens: 2000,
      response_format: { type: 'json_object' },
    })

    return completion.choices[0].message.content || '{}'
  }

  async analyzeArticle(
    article: string,
    originalText: string
  ): Promise<ArticleInterpretation> {
    const prompt = ARTICLE_INTERPRETATION_PROMPT(originalText)
    const response = await this.chat(prompt)
    const parsed = JSON.parse(response)

    return {
      article,
      original: originalText,
      simplified: parsed.simplified,
    }
  }

  async analyzeSocialImpact(summary: string): Promise<string> {
    const prompt = SOCIAL_IMPACT_PROMPT(summary)
    const response = await this.chat(prompt)
    const parsed = JSON.parse(response)

    return parsed.impact
  }

  async analyzeProsCons(summary: string): Promise<ProsCons> {
    const prompt = PROS_CONS_PROMPT(summary)
    const response = await this.chat(prompt)
    const parsed = JSON.parse(response)

    return {
      pros: parsed.pros,
      cons: parsed.cons,
    }
  }

  async analyzeBill(
    articles: Array<{ article: string; text: string }>,
    summary: string
  ): Promise<BillAnalysis> {
    // 조문별 해석
    const interpretations = await Promise.all(
      articles.map((a) => this.analyzeArticle(a.article, a.text))
    )

    // 사회적 영향
    const socialImpact = await this.analyzeSocialImpact(summary)

    // 찬반 의견
    const prosCons = await this.analyzeProsCons(summary)

    return {
      easy_interpretation: interpretations,
      social_impact: socialImpact,
      pros_cons: prosCons,
    }
  }
}

export const openaiClient = new OpenAIClient()
```

**Step 4: 커밋**

```bash
git add src/lib/openai/
git commit -m "feat: add OpenAI client for bill analysis

- Set up OpenAI client with GPT-4 Turbo
- Add prompt templates for analysis
- Implement bill analysis methods

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 6: 법안 검색 API 구현

**Files:**
- Create: `src/app/api/bills/search/route.ts`
- Create: `src/lib/utils/date.ts`

**Step 1: 날짜 유틸리티 작성**

File: `src/lib/utils/date.ts`
```typescript
export function formatDate(dateString: string): string {
  const date = new Date(dateString)
  return date.toISOString().split('T')[0]
}

export function parseDate(dateString: string): Date {
  return new Date(dateString)
}
```

**Step 2: 법안 검색 API 작성**

File: `src/app/api/bills/search/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { supabaseAdmin } from '@/lib/supabase/server'
import { assemblyAPI } from '@/lib/assembly/client'

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams
    const query = searchParams.get('q') || ''
    const startDate = searchParams.get('startDate') || undefined
    const endDate = searchParams.get('endDate') || undefined
    const status = searchParams.get('status') || undefined
    const page = parseInt(searchParams.get('page') || '1')
    const limit = parseInt(searchParams.get('limit') || '20')

    // Supabase에서 먼저 검색
    let supabaseQuery = supabaseAdmin
      .from('bills')
      .select('*, proposers(*)')
      .order('propose_date', { ascending: false })
      .range((page - 1) * limit, page * limit - 1)

    if (query) {
      supabaseQuery = supabaseQuery.or(
        `bill_name.ilike.%${query}%,bill_id.ilike.%${query}%`
      )
    }

    if (startDate) {
      supabaseQuery = supabaseQuery.gte('propose_date', startDate)
    }

    if (endDate) {
      supabaseQuery = supabaseQuery.lte('propose_date', endDate)
    }

    if (status) {
      supabaseQuery = supabaseQuery.eq('status', status)
    }

    const { data: bills, error, count } = await supabaseQuery

    if (error) {
      throw error
    }

    // 각 법안의 분석 여부 확인
    const billsWithAnalysis = await Promise.all(
      (bills || []).map(async (bill) => {
        const { data: analysis } = await supabaseAdmin
          .from('analyses')
          .select('id')
          .eq('bill_id', bill.id)
          .single()

        return {
          ...bill,
          hasAnalysis: !!analysis,
        }
      })
    )

    return NextResponse.json({
      bills: billsWithAnalysis,
      total: count || 0,
      page,
      totalPages: Math.ceil((count || 0) / limit),
    })
  } catch (error) {
    console.error('Search bills error:', error)
    return NextResponse.json(
      { error: 'Failed to search bills' },
      { status: 500 }
    )
  }
}
```

**Step 3: 테스트 실행**

```bash
# 테스트용 데이터를 Supabase에 먼저 삽입
npm run dev
# 브라우저에서 http://localhost:3000/api/bills/search?q=근로 접속
# 응답 확인
```

**Step 4: 커밋**

```bash
git add src/app/api/bills/search/ src/lib/utils/
git commit -m "feat: implement bill search API

- Search bills from Supabase with filters
- Check analysis status for each bill
- Add pagination support

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 7: 법안 상세 조회 API 구현

**Files:**
- Create: `src/app/api/bills/[billId]/route.ts`

**Step 1: 법안 상세 조회 API 작성**

File: `src/app/api/bills/[billId]/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { supabaseAdmin } from '@/lib/supabase/server'

export async function GET(
  request: NextRequest,
  { params }: { params: { billId: string } }
) {
  try {
    const { billId } = params

    // 법안 기본 정보 조회
    const { data: bill, error: billError } = await supabaseAdmin
      .from('bills')
      .select('*')
      .eq('bill_id', billId)
      .single()

    if (billError || !bill) {
      return NextResponse.json({ error: 'Bill not found' }, { status: 404 })
    }

    // 발의자 정보 조회
    const { data: proposers, error: proposersError } = await supabaseAdmin
      .from('proposers')
      .select('*')
      .eq('bill_id', bill.id)
      .order('is_representative', { ascending: false })

    if (proposersError) {
      throw proposersError
    }

    // 분석 여부 확인
    const { data: analysis } = await supabaseAdmin
      .from('analyses')
      .select('id')
      .eq('bill_id', bill.id)
      .single()

    return NextResponse.json({
      bill,
      proposers: proposers || [],
      hasAnalysis: !!analysis,
    })
  } catch (error) {
    console.error('Get bill detail error:', error)
    return NextResponse.json(
      { error: 'Failed to get bill detail' },
      { status: 500 }
    )
  }
}
```

**Step 2: 테스트 실행**

```bash
npm run dev
# 브라우저에서 http://localhost:3000/api/bills/{테스트법안ID} 접속
# 응답 확인
```

**Step 3: 커밋**

```bash
git add src/app/api/bills/
git commit -m "feat: implement bill detail API

- Get bill basic info, proposers, and analysis status
- Return 404 if bill not found

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 8: 법안 분석 API 구현 (LLM)

**Files:**
- Create: `src/app/api/bills/[billId]/analyze/route.ts`
- Create: `src/lib/utils/lock.ts`

**Step 1: 분석 Lock 유틸리티 작성**

File: `src/lib/utils/lock.ts`
```typescript
import { supabaseAdmin } from '@/lib/supabase/server'

const LOCK_TIMEOUT_SECONDS = 60

export async function acquireLock(billId: string): Promise<boolean> {
  try {
    // 기존 lock 확인
    const { data: existingLock } = await supabaseAdmin
      .from('analysis_locks')
      .select('created_at')
      .eq('bill_id', billId)
      .single()

    if (existingLock) {
      const lockAge =
        (Date.now() - new Date(existingLock.created_at).getTime()) / 1000
      if (lockAge < LOCK_TIMEOUT_SECONDS) {
        return false // Lock이 아직 유효함
      }
      // Timeout된 lock 삭제
      await supabaseAdmin.from('analysis_locks').delete().eq('bill_id', billId)
    }

    // 새 lock 생성
    const { error } = await supabaseAdmin
      .from('analysis_locks')
      .insert({ bill_id: billId })

    return !error
  } catch (error) {
    console.error('Acquire lock error:', error)
    return false
  }
}

export async function releaseLock(billId: string): Promise<void> {
  await supabaseAdmin.from('analysis_locks').delete().eq('bill_id', billId)
}
```

**Step 2: analysis_locks 테이블 마이그레이션 작성**

File: `supabase/migrations/20260201000005_create_analysis_locks_table.sql`
```sql
-- analysis_locks 테이블 생성
CREATE TABLE IF NOT EXISTS analysis_locks (
  bill_id UUID PRIMARY KEY REFERENCES bills(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

COMMENT ON TABLE analysis_locks IS 'LLM 분석 중복 방지 Lock';
```

Supabase Dashboard → SQL Editor에서 실행

**Step 3: 법안 분석 API 작성**

File: `src/app/api/bills/[billId]/analyze/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { supabaseAdmin } from '@/lib/supabase/server'
import { openaiClient } from '@/lib/openai/client'
import { acquireLock, releaseLock } from '@/lib/utils/lock'

export const maxDuration = 60 // Vercel Function timeout

export async function POST(
  request: NextRequest,
  { params }: { params: { billId: string } }
) {
  const { billId } = params

  try {
    // 1. 법안 정보 조회
    const { data: bill, error: billError } = await supabaseAdmin
      .from('bills')
      .select('*')
      .eq('bill_id', billId)
      .single()

    if (billError || !bill) {
      return NextResponse.json({ error: 'Bill not found' }, { status: 404 })
    }

    // 2. 캐시 확인
    const { data: existingAnalysis } = await supabaseAdmin
      .from('analyses')
      .select('*')
      .eq('bill_id', bill.id)
      .single()

    if (existingAnalysis) {
      return NextResponse.json(existingAnalysis)
    }

    // 3. Lock 획득
    const lockAcquired = await acquireLock(bill.id)
    if (!lockAcquired) {
      return NextResponse.json(
        { error: 'Analysis in progress by another user' },
        { status: 409 }
      )
    }

    try {
      // 4. LLM 분석 실행
      // 임시로 법안 요지를 조문으로 사용 (실제로는 국회 API에서 조문 파싱 필요)
      const articles = [
        { article: '제1조 (목적)', text: bill.summary || '' },
      ]

      const analysis = await openaiClient.analyzeBill(
        articles,
        bill.summary || ''
      )

      // 5. Supabase에 저장
      const { data: savedAnalysis, error: saveError } = await supabaseAdmin
        .from('analyses')
        .insert({
          bill_id: bill.id,
          easy_interpretation: analysis.easy_interpretation as any,
          social_impact: analysis.social_impact,
          pros_cons: analysis.pros_cons as any,
          llm_model: 'gpt-4-turbo-preview',
        })
        .select()
        .single()

      if (saveError) {
        throw saveError
      }

      return NextResponse.json(savedAnalysis)
    } finally {
      // 6. Lock 해제
      await releaseLock(bill.id)
    }
  } catch (error) {
    console.error('Analyze bill error:', error)
    return NextResponse.json(
      { error: 'Failed to analyze bill' },
      { status: 500 }
    )
  }
}

export async function GET(
  request: NextRequest,
  { params }: { params: { billId: string } }
) {
  const { billId } = params

  try {
    // 법안 정보 조회
    const { data: bill, error: billError } = await supabaseAdmin
      .from('bills')
      .select('id')
      .eq('bill_id', billId)
      .single()

    if (billError || !bill) {
      return NextResponse.json({ error: 'Bill not found' }, { status: 404 })
    }

    // 분석 결과 조회
    const { data: analysis, error: analysisError } = await supabaseAdmin
      .from('analyses')
      .select('*')
      .eq('bill_id', bill.id)
      .single()

    if (analysisError || !analysis) {
      return NextResponse.json(
        { error: 'Analysis not found' },
        { status: 404 }
      )
    }

    return NextResponse.json(analysis)
  } catch (error) {
    console.error('Get analysis error:', error)
    return NextResponse.json(
      { error: 'Failed to get analysis' },
      { status: 500 }
    )
  }
}
```

**Step 4: 커밋**

```bash
git add src/app/api/bills/ src/lib/utils/lock.ts supabase/migrations/
git commit -m "feat: implement bill analysis API with LLM

- Add analysis lock mechanism
- Integrate OpenAI GPT-4 for bill analysis
- Cache analysis results in Supabase

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 9: 최근 법안 목록 API 구현

**Files:**
- Create: `src/app/api/bills/recent/route.ts`

**Step 1: 최근 법안 목록 API 작성**

File: `src/app/api/bills/recent/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { supabaseAdmin } from '@/lib/supabase/server'

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams
    const limit = parseInt(searchParams.get('limit') || '20')

    const { data: bills, error } = await supabaseAdmin
      .from('bills')
      .select('*, proposers(*)')
      .order('propose_date', { ascending: false })
      .limit(limit)

    if (error) {
      throw error
    }

    // 각 법안의 분석 여부 확인
    const billsWithAnalysis = await Promise.all(
      (bills || []).map(async (bill) => {
        const { data: analysis } = await supabaseAdmin
          .from('analyses')
          .select('id')
          .eq('bill_id', bill.id)
          .single()

        return {
          ...bill,
          hasAnalysis: !!analysis,
        }
      })
    )

    return NextResponse.json({
      bills: billsWithAnalysis,
      total: bills?.length || 0,
    })
  } catch (error) {
    console.error('Get recent bills error:', error)
    return NextResponse.json(
      { error: 'Failed to get recent bills' },
      { status: 500 }
    )
  }
}
```

**Step 2: 커밋**

```bash
git add src/app/api/bills/recent/
git commit -m "feat: implement recent bills API

- Get most recent bills ordered by propose_date
- Include analysis status

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 10: shadcn/ui 설정 및 기본 컴포넌트 설치

**Files:**
- Create: `components.json`
- Modify: `tailwind.config.ts`
- Create: `src/components/ui/*`

**Step 1: shadcn/ui 초기화**

Run:
```bash
npx shadcn@latest init
```

옵션 선택:
- Style: New York
- Base color: Slate
- CSS variables: Yes

**Step 2: 필요한 컴포넌트 설치**

Run:
```bash
npx shadcn@latest add button card input tabs accordion badge skeleton toast
```

**Step 3: 커밋**

```bash
git add components.json tailwind.config.ts src/components/ui/
git commit -m "feat: set up shadcn/ui and install base components

- Initialize shadcn/ui with New York style
- Add button, card, input, tabs, accordion, badge, skeleton, toast

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 11: 홈 페이지 UI 구현

**Files:**
- Create: `src/app/page.tsx`
- Create: `src/components/BillCard.tsx`
- Create: `src/components/SearchBar.tsx`
- Create: `src/app/globals.css`

**Step 1: SearchBar 컴포넌트 작성**

File: `src/components/SearchBar.tsx`
```typescript
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'

export function SearchBar() {
  const [query, setQuery] = useState('')
  const router = useRouter()

  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault()
    if (query.trim()) {
      router.push(`/search?q=${encodeURIComponent(query)}`)
    }
  }

  return (
    <form onSubmit={handleSearch} className="flex gap-2 max-w-2xl mx-auto">
      <Input
        type="text"
        placeholder="법안명 또는 의원명으로 검색..."
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        className="flex-1"
      />
      <Button type="submit">검색</Button>
    </form>
  )
}
```

**Step 2: BillCard 컴포넌트 작성**

File: `src/components/BillCard.tsx`
```typescript
import Link from 'next/link'
import { Card, CardHeader, CardTitle, CardDescription } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'

interface BillCardProps {
  billId: string
  billName: string
  proposerName: string
  proposerParty: string
  proposeDate: string
  status: string
  hasAnalysis: boolean
}

export function BillCard({
  billId,
  billName,
  proposerName,
  proposerParty,
  proposeDate,
  status,
  hasAnalysis,
}: BillCardProps) {
  const statusColor = {
    발의: 'bg-blue-500',
    계류중: 'bg-yellow-500',
    통과: 'bg-green-500',
    폐기: 'bg-red-500',
  }[status] || 'bg-gray-500'

  return (
    <Link href={`/bills/${billId}`}>
      <Card className="hover:shadow-lg transition-shadow cursor-pointer">
        <CardHeader>
          <div className="flex justify-between items-start mb-2">
            <CardTitle className="text-lg">{billName}</CardTitle>
            <Badge className={statusColor}>{status}</Badge>
          </div>
          <CardDescription>
            <div className="flex flex-col gap-1">
              <span>
                대표발의: {proposerName} ({proposerParty})
              </span>
              <span>발의일: {proposeDate}</span>
              {hasAnalysis && (
                <Badge variant="outline" className="w-fit mt-2">
                  AI 분석 완료
                </Badge>
              )}
            </div>
          </CardDescription>
        </CardHeader>
      </Card>
    </Link>
  )
}
```

**Step 3: 홈 페이지 작성**

File: `src/app/page.tsx`
```typescript
import { SearchBar } from '@/components/SearchBar'
import { BillCard } from '@/components/BillCard'
import { supabase } from '@/lib/supabase/client'

export const dynamic = 'force-dynamic'

async function getRecentBills() {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_APP_URL || 'http://localhost:3000'}/api/bills/recent?limit=20`,
    { cache: 'no-store' }
  )

  if (!response.ok) {
    return []
  }

  const data = await response.json()
  return data.bills || []
}

export default async function HomePage() {
  const bills = await getRecentBills()

  return (
    <main className="min-h-screen bg-gradient-to-b from-slate-50 to-slate-100 dark:from-slate-900 dark:to-slate-800">
      <div className="container mx-auto px-4 py-16">
        {/* Hero Section */}
        <div className="text-center mb-12">
          <h1 className="text-5xl font-bold mb-4 bg-gradient-to-r from-blue-600 to-purple-600 bg-clip-text text-transparent">
            법안해석기
          </h1>
          <p className="text-xl text-slate-600 dark:text-slate-300 mb-8">
            새로 발의되는 법, AI가 쉽게 설명해드립니다
          </p>
          <SearchBar />
        </div>

        {/* Recent Bills */}
        <div className="mt-16">
          <h2 className="text-3xl font-bold mb-6">최근 발의된 법안</h2>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            {bills.map((bill: any) => (
              <BillCard
                key={bill.id}
                billId={bill.bill_id}
                billName={bill.bill_name}
                proposerName={
                  bill.proposers.find((p: any) => p.is_representative)
                    ?.member_name || '미상'
                }
                proposerParty={
                  bill.proposers.find((p: any) => p.is_representative)?.party ||
                  ''
                }
                proposeDate={bill.propose_date}
                status={bill.status || '미상'}
                hasAnalysis={bill.hasAnalysis}
              />
            ))}
          </div>
        </div>
      </div>
    </main>
  )
}
```

**Step 4: 커밋**

```bash
git add src/app/page.tsx src/components/
git commit -m "feat: implement home page UI

- Add SearchBar component
- Add BillCard component
- Display recent bills on home page

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 12: 법안 상세 페이지 UI 구현

**Files:**
- Create: `src/app/bills/[billId]/page.tsx`
- Create: `src/components/BillDetailHeader.tsx`
- Create: `src/components/ProposersCard.tsx`
- Create: `src/components/AnalysisSection.tsx`

**Step 1: BillDetailHeader 컴포넌트 작성**

File: `src/components/BillDetailHeader.tsx`
```typescript
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { ExternalLink } from 'lucide-react'

interface BillDetailHeaderProps {
  billName: string
  billId: string
  proposeDate: string
  status: string
  committee: string
  billUrl?: string
}

export function BillDetailHeader({
  billName,
  billId,
  proposeDate,
  status,
  committee,
  billUrl,
}: BillDetailHeaderProps) {
  return (
    <div className="mb-8">
      <h1 className="text-4xl font-bold mb-4">{billName}</h1>
      <div className="flex flex-wrap gap-4 text-slate-600 dark:text-slate-300">
        <span>의안번호: {billId}</span>
        <span>발의일: {proposeDate}</span>
        <Badge>{status}</Badge>
        <span>소관위원회: {committee}</span>
      </div>
      {billUrl && (
        <Button variant="outline" className="mt-4" asChild>
          <a href={billUrl} target="_blank" rel="noopener noreferrer">
            국회 원문 보기 <ExternalLink className="ml-2 h-4 w-4" />
          </a>
        </Button>
      )}
    </div>
  )
}
```

**Step 2: ProposersCard 컴포넌트 작성**

File: `src/components/ProposersCard.tsx`
```typescript
'use client'

import { useState } from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'

interface Proposer {
  member_name: string
  party: string
  is_representative: boolean
}

interface ProposersCardProps {
  proposers: Proposer[]
}

export function ProposersCard({ proposers }: ProposersCardProps) {
  const [showAll, setShowAll] = useState(false)
  const representative = proposers.find((p) => p.is_representative)
  const coProposers = proposers.filter((p) => !p.is_representative)
  const displayedCoProposers = showAll ? coProposers : coProposers.slice(0, 5)

  return (
    <Card>
      <CardHeader>
        <CardTitle>발의자 정보</CardTitle>
      </CardHeader>
      <CardContent>
        {representative && (
          <div className="mb-4 pb-4 border-b">
            <p className="font-semibold mb-1">대표발의</p>
            <Badge variant="default">
              {representative.member_name} ({representative.party})
            </Badge>
          </div>
        )}
        {coProposers.length > 0 && (
          <div>
            <p className="font-semibold mb-2">공동발의 ({coProposers.length}명)</p>
            <div className="flex flex-wrap gap-2">
              {displayedCoProposers.map((p, i) => (
                <Badge key={i} variant="outline">
                  {p.member_name} ({p.party})
                </Badge>
              ))}
            </div>
            {coProposers.length > 5 && (
              <Button
                variant="ghost"
                size="sm"
                className="mt-2"
                onClick={() => setShowAll(!showAll)}
              >
                {showAll ? '접기' : `${coProposers.length - 5}명 더 보기`}
              </Button>
            )}
          </div>
        )}
      </CardContent>
    </Card>
  )
}
```

**Step 3: AnalysisSection 컴포넌트 작성**

File: `src/components/AnalysisSection.tsx`
```typescript
'use client'

import { useState } from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import {
  Accordion,
  AccordionContent,
  AccordionItem,
  AccordionTrigger,
} from '@/components/ui/accordion'
import { Skeleton } from '@/components/ui/skeleton'
import { useToast } from '@/hooks/use-toast'

interface Analysis {
  easy_interpretation: Array<{
    article: string
    original: string
    simplified: string
  }>
  social_impact: string
  pros_cons: {
    pros: string
    cons: string
  }
}

interface AnalysisSectionProps {
  billId: string
  hasAnalysis: boolean
}

export function AnalysisSection({ billId, hasAnalysis }: AnalysisSectionProps) {
  const [loading, setLoading] = useState(false)
  const [analysis, setAnalysis] = useState<Analysis | null>(null)
  const { toast } = useToast()

  const handleAnalyze = async () => {
    setLoading(true)
    try {
      const response = await fetch(`/api/bills/${billId}/analyze`, {
        method: 'POST',
      })

      if (!response.ok) {
        throw new Error('Analysis failed')
      }

      const data = await response.json()
      setAnalysis(data)
      toast({
        title: '분석 완료',
        description: 'AI 분석이 완료되었습니다.',
      })
    } catch (error) {
      toast({
        title: '분석 실패',
        description: 'AI 분석 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요.',
        variant: 'destructive',
      })
    } finally {
      setLoading(false)
    }
  }

  if (!analysis && !hasAnalysis) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>AI 분석</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="text-center py-8">
            <p className="mb-4 text-slate-600 dark:text-slate-300">
              이 법안은 아직 분석되지 않았습니다.
            </p>
            <p className="mb-6 text-sm text-slate-500">
              약 20초 정도 소요됩니다.
            </p>
            <Button onClick={handleAnalyze} disabled={loading}>
              {loading ? 'AI가 법안을 분석하고 있습니다...' : 'AI로 쉽게 해석하기'}
            </Button>
          </div>
        </CardContent>
      </Card>
    )
  }

  if (loading) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>AI 분석 중...</CardTitle>
        </CardHeader>
        <CardContent>
          <Skeleton className="h-40 w-full" />
        </CardContent>
      </Card>
    )
  }

  if (!analysis) {
    return null
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>AI 분석 결과</CardTitle>
      </CardHeader>
      <CardContent>
        <Tabs defaultValue="interpretation">
          <TabsList className="grid w-full grid-cols-3">
            <TabsTrigger value="interpretation">쉬운 해석</TabsTrigger>
            <TabsTrigger value="impact">사회적 영향</TabsTrigger>
            <TabsTrigger value="prosCons">찬반 의견</TabsTrigger>
          </TabsList>

          <TabsContent value="interpretation">
            <Accordion type="single" collapsible>
              {analysis.easy_interpretation.map((item, index) => (
                <AccordionItem key={index} value={`item-${index}`}>
                  <AccordionTrigger>{item.article}</AccordionTrigger>
                  <AccordionContent>
                    <div className="space-y-4">
                      <div>
                        <p className="text-sm text-slate-500 mb-2">원문</p>
                        <p className="text-sm text-slate-600 dark:text-slate-400">
                          {item.original}
                        </p>
                      </div>
                      <div>
                        <p className="text-sm text-slate-500 mb-2">쉬운 풀이</p>
                        <p className="text-base">{item.simplified}</p>
                      </div>
                    </div>
                  </AccordionContent>
                </AccordionItem>
              ))}
            </Accordion>
          </TabsContent>

          <TabsContent value="impact">
            <div className="prose dark:prose-invert max-w-none">
              <p>{analysis.social_impact}</p>
            </div>
          </TabsContent>

          <TabsContent value="prosCons">
            <div className="grid md:grid-cols-2 gap-6">
              <div className="border-l-4 border-green-500 pl-4">
                <h3 className="text-lg font-semibold mb-2 text-green-700 dark:text-green-400">
                  찬성 입장
                </h3>
                <p>{analysis.pros_cons.pros}</p>
              </div>
              <div className="border-l-4 border-red-500 pl-4">
                <h3 className="text-lg font-semibold mb-2 text-red-700 dark:text-red-400">
                  반대 입장
                </h3>
                <p>{analysis.pros_cons.cons}</p>
              </div>
            </div>
          </TabsContent>
        </Tabs>
      </CardContent>
    </Card>
  )
}
```

**Step 4: 법안 상세 페이지 작성**

File: `src/app/bills/[billId]/page.tsx`
```typescript
import { BillDetailHeader } from '@/components/BillDetailHeader'
import { ProposersCard } from '@/components/ProposersCard'
import { AnalysisSection } from '@/components/AnalysisSection'

async function getBillDetail(billId: string) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_APP_URL || 'http://localhost:3000'}/api/bills/${billId}`,
    { cache: 'no-store' }
  )

  if (!response.ok) {
    return null
  }

  return response.json()
}

export default async function BillDetailPage({
  params,
}: {
  params: { billId: string }
}) {
  const data = await getBillDetail(params.billId)

  if (!data) {
    return (
      <div className="container mx-auto px-4 py-16">
        <h1 className="text-3xl font-bold">법안을 찾을 수 없습니다</h1>
      </div>
    )
  }

  const { bill, proposers, hasAnalysis } = data

  return (
    <div className="container mx-auto px-4 py-16">
      <BillDetailHeader
        billName={bill.bill_name}
        billId={bill.bill_id}
        proposeDate={bill.propose_date}
        status={bill.status}
        committee={bill.committee}
        billUrl={bill.bill_url}
      />

      <div className="grid gap-8">
        <ProposersCard proposers={proposers} />
        <AnalysisSection billId={bill.bill_id} hasAnalysis={hasAnalysis} />
      </div>
    </div>
  )
}
```

**Step 5: 커밋**

```bash
git add src/app/bills/ src/components/
git commit -m "feat: implement bill detail page UI

- Add BillDetailHeader component
- Add ProposersCard component
- Add AnalysisSection with tabs and accordion
- Implement analysis request flow

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 13: 검색 결과 페이지 UI 구현

**Files:**
- Create: `src/app/search/page.tsx`
- Create: `src/components/SearchFilters.tsx`

**Step 1: SearchFilters 컴포넌트 작성**

File: `src/components/SearchFilters.tsx`
```typescript
'use client'

import { useRouter, useSearchParams } from 'next/navigation'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'

export function SearchFilters() {
  const router = useRouter()
  const searchParams = useSearchParams()

  const handleFilterChange = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString())
    if (value) {
      params.set(key, value)
    } else {
      params.delete(key)
    }
    router.push(`/search?${params.toString()}`)
  }

  return (
    <div className="flex flex-wrap gap-4 mb-8">
      <Input
        type="date"
        placeholder="시작일"
        onChange={(e) => handleFilterChange('startDate', e.target.value)}
        className="w-40"
      />
      <Input
        type="date"
        placeholder="종료일"
        onChange={(e) => handleFilterChange('endDate', e.target.value)}
        className="w-40"
      />
      <Select
        onValueChange={(value) => handleFilterChange('status', value)}
      >
        <SelectTrigger className="w-32">
          <SelectValue placeholder="상태" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="발의">발의</SelectItem>
          <SelectItem value="계류중">계류중</SelectItem>
          <SelectItem value="통과">통과</SelectItem>
          <SelectItem value="폐기">폐기</SelectItem>
        </SelectContent>
      </Select>
    </div>
  )
}
```

**Step 2: 검색 결과 페이지 작성**

File: `src/app/search/page.tsx`
```typescript
import { SearchBar } from '@/components/SearchBar'
import { SearchFilters } from '@/components/SearchFilters'
import { BillCard } from '@/components/BillCard'
import { Button } from '@/components/ui/button'

async function searchBills(searchParams: any) {
  const params = new URLSearchParams(searchParams)
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_APP_URL || 'http://localhost:3000'}/api/bills/search?${params}`,
    { cache: 'no-store' }
  )

  if (!response.ok) {
    return { bills: [], total: 0, page: 1, totalPages: 0 }
  }

  return response.json()
}

export default async function SearchPage({
  searchParams,
}: {
  searchParams: any
}) {
  const { bills, total, page, totalPages } = await searchBills(searchParams)

  return (
    <div className="container mx-auto px-4 py-16">
      <h1 className="text-3xl font-bold mb-8">법안 검색</h1>

      <SearchBar />

      <div className="mt-8">
        <SearchFilters />

        {bills.length === 0 ? (
          <div className="text-center py-16">
            <p className="text-xl text-slate-600 dark:text-slate-300">
              검색 결과가 없습니다.
            </p>
          </div>
        ) : (
          <>
            <p className="mb-4 text-slate-600 dark:text-slate-300">
              총 {total}건의 법안이 검색되었습니다.
            </p>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
              {bills.map((bill: any) => (
                <BillCard
                  key={bill.id}
                  billId={bill.bill_id}
                  billName={bill.bill_name}
                  proposerName={
                    bill.proposers.find((p: any) => p.is_representative)
                      ?.member_name || '미상'
                  }
                  proposerParty={
                    bill.proposers.find((p: any) => p.is_representative)?.party ||
                    ''
                  }
                  proposeDate={bill.propose_date}
                  status={bill.status || '미상'}
                  hasAnalysis={bill.hasAnalysis}
                />
              ))}
            </div>

            {/* Pagination */}
            {totalPages > 1 && (
              <div className="flex justify-center gap-2 mt-8">
                {Array.from({ length: totalPages }, (_, i) => i + 1).map((p) => (
                  <Button
                    key={p}
                    variant={p === page ? 'default' : 'outline'}
                    onClick={() => {
                      const params = new URLSearchParams(searchParams)
                      params.set('page', p.toString())
                      window.location.href = `/search?${params}`
                    }}
                  >
                    {p}
                  </Button>
                ))}
              </div>
            )}
          </>
        )}
      </div>
    </div>
  )
}
```

**Step 3: Select 컴포넌트 설치**

Run:
```bash
npx shadcn@latest add select
```

**Step 4: 커밋**

```bash
git add src/app/search/ src/components/SearchFilters.tsx
git commit -m "feat: implement search results page

- Add SearchFilters component with date and status filters
- Display search results with pagination
- Add empty state

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 14: 환경 변수 및 배포 준비

**Files:**
- Create: `.env.example`
- Modify: `next.config.ts`
- Create: `vercel.json`

**Step 1: .env.example 작성**

File: `.env.example`
```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key

# OpenAI
OPENAI_API_KEY=sk-...

# 국회 의안정보시스템 API
ASSEMBLY_API_KEY=your_assembly_api_key

# App URL (for server-side fetches)
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

**Step 2: next.config.ts 업데이트**

File: `next.config.ts`
```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactStrictMode: true,

  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,POST,OPTIONS' },
          { key: 'Access-Control-Allow-Headers', value: 'Content-Type' },
        ],
      },
    ]
  },
}

export default nextConfig
```

**Step 3: vercel.json 작성**

File: `vercel.json`
```json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "regions": ["icn1"],
  "functions": {
    "app/api/bills/[billId]/analyze/route.ts": {
      "maxDuration": 60
    }
  }
}
```

**Step 4: 커밋**

```bash
git add .env.example next.config.ts vercel.json
git commit -m "feat: prepare for deployment

- Add .env.example with all required variables
- Configure CORS headers
- Set Vercel region to Seoul (icn1)
- Configure function timeouts

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 15: README 및 문서화

**Files:**
- Create: `README.md`
- Create: `docs/DEPLOYMENT.md`
- Create: `docs/API.md`

**Step 1: README.md 작성**

File: `README.md`
```markdown
# 법안해석기 (Legal Interpreter)

입법 예고/진행 중인 법률을 일반 국민이 쉽게 이해할 수 있도록 LLM 기반 해석을 제공하는 웹 서비스

## 기술 스택

- **Frontend**: Next.js 15, React, TypeScript, TailwindCSS, shadcn/ui
- **Backend**: Next.js API Routes
- **Database**: Supabase (PostgreSQL)
- **LLM**: OpenAI GPT-4 Turbo
- **Deployment**: Vercel

## 시작하기

### 1. 환경 변수 설정

\`\`\`bash
cp .env.example .env.local
\`\`\`

`.env.local` 파일에 필요한 API 키를 입력하세요:
- Supabase 프로젝트 URL 및 키
- OpenAI API 키
- 국회 의안정보시스템 API 키

### 2. 패키지 설치

\`\`\`bash
npm install
\`\`\`

### 3. 데이터베이스 마이그레이션

Supabase Dashboard → SQL Editor에서 `supabase/migrations/` 폴더의 SQL 파일들을 순서대로 실행

### 4. 개발 서버 실행

\`\`\`bash
npm run dev
\`\`\`

브라우저에서 http://localhost:3000 접속

## 프로젝트 구조

\`\`\`
src/
├── app/                    # Next.js App Router
│   ├── api/                # API Routes
│   ├── bills/              # 법안 상세 페이지
│   ├── search/             # 검색 페이지
│   └── page.tsx            # 홈 페이지
├── components/             # React 컴포넌트
│   └── ui/                 # shadcn/ui 컴포넌트
├── lib/                    # 유틸리티 및 클라이언트
│   ├── assembly/           # 국회 API 클라이언트
│   ├── openai/             # OpenAI 클라이언트
│   ├── supabase/           # Supabase 클라이언트
│   └── utils/              # 유틸리티 함수
└── types/                  # TypeScript 타입 정의
\`\`\`

## 주요 기능

- 법안 검색 및 필터링
- 최근 발의 법안 목록
- 법안 상세 정보 조회
- AI 기반 법안 해석 (조문별 쉬운 해석, 사회적 영향, 찬반 의견)
- 분석 결과 캐싱

## 배포

[DEPLOYMENT.md](docs/DEPLOYMENT.md) 참조

## API 문서

[API.md](docs/API.md) 참조

## 라이선스

MIT
```

**Step 2: DEPLOYMENT.md 작성**

File: `docs/DEPLOYMENT.md`
```markdown
# 배포 가이드

## 사전 준비

### 1. Supabase 프로젝트 설정
1. https://supabase.com 접속
2. 새 프로젝트 생성 (Region: Seoul)
3. SQL Editor에서 마이그레이션 실행
4. API Keys 복사

### 2. 외부 API 키 발급
- 국회 의안정보시스템 API: https://www.data.go.kr
- OpenAI API: https://platform.openai.com

## Vercel 배포

### 1. GitHub 연동
\`\`\`bash
git remote add origin <your-repo-url>
git push -u origin main
\`\`\`

### 2. Vercel 프로젝트 생성
1. https://vercel.com 접속
2. Import Git Repository
3. Framework Preset: Next.js 선택

### 3. 환경 변수 설정
Vercel Dashboard → Settings → Environment Variables에서 설정:
- \`NEXT_PUBLIC_SUPABASE_URL\`
- \`NEXT_PUBLIC_SUPABASE_ANON_KEY\`
- \`SUPABASE_SERVICE_ROLE_KEY\`
- \`OPENAI_API_KEY\`
- \`ASSEMBLY_API_KEY\`
- \`NEXT_PUBLIC_APP_URL\`

### 4. 배포
\`\`\`bash
git push origin main
\`\`\`

자동 배포 시작

## 배포 후 확인 사항
- [ ] 홈 페이지 로딩 확인
- [ ] 법안 검색 기능 확인
- [ ] AI 분석 기능 확인
- [ ] 환경 변수 정상 작동 확인
```

**Step 3: API.md 작성**

File: `docs/API.md`
```markdown
# API 문서

## 엔드포인트 목록

### GET /api/bills/search
법안 검색

**Query Parameters:**
- \`q\`: 검색어
- \`startDate\`: 시작일 (YYYY-MM-DD)
- \`endDate\`: 종료일 (YYYY-MM-DD)
- \`status\`: 상태 필터
- \`page\`: 페이지 번호
- \`limit\`: 페이지당 항목 수

### GET /api/bills/[billId]
법안 상세 정보

### GET /api/bills/recent
최근 발의 법안

**Query Parameters:**
- \`limit\`: 항목 수 (default: 20)

### POST /api/bills/[billId]/analyze
법안 AI 분석

### GET /api/bills/[billId]/analyze
분석 결과 조회

### GET /api/assembly/bills
국회 API 프록시
```

**Step 4: 커밋**

```bash
git add README.md docs/
git commit -m "docs: add comprehensive documentation

- Add README with setup instructions
- Add deployment guide
- Add API documentation

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## 실행 안내

계획이 완료되었습니다. 두 가지 실행 옵션이 있습니다:

**1. Subagent-Driven (현재 세션)** - 각 Task마다 새로운 하위 에이전트를 실행하고, 작업 완료 후 코드 리뷰를 진행합니다. 빠른 반복 개발이 가능합니다.

**2. Parallel Session (별도 세션)** - 새 세션을 열어 executing-plans 스킬을 사용하여 일괄 실행합니다. 체크포인트별로 검토가 이루어집니다.

어떤 방식으로 진행하시겠습니까?
