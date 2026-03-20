# Next.js 종합 강의 & 문제집
## 초급자 → 고급 Full-Stack Developer 과정 | 글로벌 IT 기업 인터뷰 대비

> **기준 버전**: Next.js 15 (App Router 중심, 최신 안정 버전 기반)
> **대상**: 프론트엔드 입문자부터 시니어 Full-Stack 엔지니어까지
> **구성**: 개념 강의 → 샘플 코드 → 연습 문제 → 인터뷰 문제

---

# PART 1: 강의 (기본 → 고급)

---

## Chapter 1. Next.js 개요 및 프로젝트 설정

### 1.1 Next.js란?

Next.js는 Vercel이 개발한 React 기반 풀스택 웹 프레임워크다. 서버 사이드 렌더링(SSR), 정적 사이트 생성(SSG), 증분 정적 재생성(ISR), 서버 컴포넌트(RSC), 서버 액션(Server Actions) 등을 기본 제공하며, 파일 기반 라우팅 시스템을 통해 직관적인 개발 경험을 제공한다.

**React와의 차이점:**

React는 UI 라이브러리로, 라우팅·서버 렌더링·빌드 최적화 등을 직접 구성해야 한다. Next.js는 이 모든 것을 프레임워크 수준에서 통합 제공한다. React 19와 긴밀하게 통합되어 Server Components, Suspense, Transitions 등의 React 최신 기능을 네이티브로 지원한다.

**핵심 특징:**

- App Router: React Server Components 기반의 새로운 라우팅 시스템
- 하이브리드 렌더링: SSR, SSG, ISR, CSR을 페이지·컴포넌트 단위로 혼합 사용
- Server Actions: 서버 측 로직을 클라이언트에서 함수처럼 호출
- Streaming & Suspense: 점진적 HTML 스트리밍으로 TTFB 개선
- 내장 최적화: Image, Font, Script, Metadata 자동 최적화
- Middleware: Edge Runtime에서 요청 전처리
- Turbopack: Webpack 대비 대폭 빠른 번들러 (기본 활성화)

### 1.2 프로젝트 생성

```bash
# 최신 Next.js 프로젝트 생성
npx create-next-app@latest my-app

# 생성 시 옵션 선택
# ✔ TypeScript? Yes
# ✔ ESLint? Yes
# ✔ Tailwind CSS? Yes
# ✔ src/ directory? Yes
# ✔ App Router? Yes
# ✔ Turbopack? Yes
# ✔ Import alias? @/*
```

### 1.3 프로젝트 구조

```
my-app/
├── src/
│   ├── app/                    # App Router (핵심)
│   │   ├── layout.tsx          # 루트 레이아웃
│   │   ├── page.tsx            # 홈 페이지
│   │   ├── loading.tsx         # 로딩 UI
│   │   ├── error.tsx           # 에러 UI
│   │   ├── not-found.tsx       # 404 페이지
│   │   ├── global-error.tsx    # 전역 에러 UI
│   │   ├── template.tsx        # 템플릿 (리마운트)
│   │   ├── default.tsx         # 병렬 라우트 기본값
│   │   ├── favicon.ico
│   │   ├── globals.css
│   │   ├── dashboard/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   └── settings/
│   │   │       └── page.tsx
│   │   ├── blog/
│   │   │   ├── [slug]/
│   │   │   │   └── page.tsx
│   │   │   └── page.tsx
│   │   └── api/                # Route Handlers
│   │       └── users/
│   │           └── route.ts
│   ├── components/             # 공유 컴포넌트
│   ├── lib/                    # 유틸리티, DB 클라이언트 등
│   └── middleware.ts           # 미들웨어
├── public/                     # 정적 파일
├── next.config.ts              # Next.js 설정
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### 1.4 next.config.ts 주요 설정

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // 실험적 기능
  experimental: {
    ppr: true,                    // Partial Prerendering
    reactCompiler: true,          // React Compiler (자동 메모이제이션)
    serverActions: {
      bodySizeLimit: '2mb',       // Server Action 요청 크기 제한
      allowedOrigins: ['my-domain.com'],
    },
    typedRoutes: true,            // 타입 안전 라우팅
  },

  // 이미지 최적화
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        port: '',
        pathname: '/images/**',
      },
    ],
  },

  // 환경 변수
  env: {
    CUSTOM_KEY: 'value',
  },

  // 리다이렉트
  async redirects() {
    return [
      {
        source: '/old-blog/:slug',
        destination: '/blog/:slug',
        permanent: true, // 308 status
      },
    ];
  },

  // 리라이트
  async rewrites() {
    return [
      {
        source: '/api/proxy/:path*',
        destination: 'https://external-api.com/:path*',
      },
    ];
  },

  // 헤더
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
        ],
      },
    ];
  },
};

export default nextConfig;
```

---

## Chapter 2. App Router와 라우팅 시스템

### 2.1 파일 기반 라우팅

App Router에서는 `app/` 디렉토리 내의 폴더 구조가 곧 URL 경로가 된다. `page.tsx` 파일이 있는 폴더만 접근 가능한 라우트로 등록된다.

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
├── blog/
│   ├── page.tsx          → /blog
│   └── [slug]/
│       └── page.tsx      → /blog/:slug
├── shop/
│   └── [...categories]/
│       └── page.tsx      → /shop/a, /shop/a/b, /shop/a/b/c ...
└── docs/
    └── [[...slug]]/
        └── page.tsx      → /docs, /docs/a, /docs/a/b ...
```

### 2.2 동적 라우팅

```typescript
// app/blog/[slug]/page.tsx
// 단일 동적 세그먼트

interface PageProps {
  params: Promise<{ slug: string }>;
}

export default async function BlogPost({ params }: PageProps) {
  const { slug } = await params;
  const post = await getPost(slug);

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}

// 정적 생성을 위한 경로 사전 정의
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({
    slug: post.slug,
  }));
}
```

```typescript
// app/shop/[...categories]/page.tsx
// Catch-all 라우트

interface PageProps {
  params: Promise<{ categories: string[] }>;
}

export default async function ShopCategory({ params }: PageProps) {
  const { categories } = await params;
  // /shop/electronics/phones → categories = ['electronics', 'phones']

  return (
    <div>
      <h1>카테고리: {categories.join(' > ')}</h1>
    </div>
  );
}
```

### 2.3 라우트 그룹 (Route Groups)

URL 경로에 영향을 주지 않으면서 라우트를 논리적으로 그룹화한다.

```
app/
├── (marketing)/         # URL에 포함되지 않음
│   ├── layout.tsx       # 마케팅 전용 레이아웃
│   ├── about/
│   │   └── page.tsx     → /about
│   └── contact/
│       └── page.tsx     → /contact
├── (shop)/
│   ├── layout.tsx       # 쇼핑 전용 레이아웃
│   ├── products/
│   │   └── page.tsx     → /products
│   └── cart/
│       └── page.tsx     → /cart
└── (auth)/
    ├── layout.tsx       # 인증 전용 레이아웃 (최소 UI)
    ├── login/
    │   └── page.tsx     → /login
    └── register/
        └── page.tsx     → /register
```

### 2.4 병렬 라우트 (Parallel Routes)

같은 레이아웃 안에서 여러 페이지를 동시에 렌더링한다. `@folder` 컨벤션을 사용한다.

```
app/
├── layout.tsx
├── page.tsx
├── @analytics/
│   ├── page.tsx
│   └── default.tsx
├── @team/
│   ├── page.tsx
│   └── default.tsx
└── @revenue/
    ├── page.tsx
    └── default.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
  revenue,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
  revenue: React.ReactNode;
}) {
  return (
    <div>
      <main>{children}</main>
      <div className="grid grid-cols-3 gap-4">
        {analytics}
        {team}
        {revenue}
      </div>
    </div>
  );
}
```

### 2.5 인터셉팅 라우트 (Intercepting Routes)

현재 레이아웃을 유지하면서 다른 라우트를 가로채서 모달 등으로 표시한다.

```
app/
├── layout.tsx
├── page.tsx              # 피드 목록
├── @modal/
│   ├── default.tsx
│   └── (.)photo/[id]/    # (.) = 같은 레벨 인터셉트
│       └── page.tsx       # 모달로 표시
└── photo/[id]/
    └── page.tsx           # 직접 접근 시 전체 페이지
```

```
인터셉트 컨벤션:
(.)  → 같은 레벨
(..) → 한 레벨 위
(..)(..) → 두 레벨 위
(...) → 루트에서
```

```tsx
// app/@modal/(.)photo/[id]/page.tsx
import { Modal } from '@/components/modal';

export default async function PhotoModal({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const photo = await getPhoto(id);

  return (
    <Modal>
      <img src={photo.url} alt={photo.title} />
      <p>{photo.title}</p>
    </Modal>
  );
}
```

### 2.6 레이아웃과 템플릿

```tsx
// app/layout.tsx — 루트 레이아웃 (필수)
// 상태가 유지되며 리렌더링 시 리마운트되지 않음
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: { default: 'My App', template: '%s | My App' },
  description: 'Next.js 풀스택 애플리케이션',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body className={inter.className}>
        <nav>네비게이션</nav>
        {children}
        <footer>푸터</footer>
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/template.tsx
// 네비게이션마다 새로 마운트됨 (상태 초기화)
// useEffect, useState가 매번 재실행됨

export default function DashboardTemplate({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <h2>Dashboard</h2>
      {children}
    </div>
  );
}
```

**layout vs template 차이:**
- `layout.tsx`: 네비게이션 간 상태 유지, 리렌더링만 발생
- `template.tsx`: 네비게이션마다 새 인스턴스 생성, 상태 초기화

---

## Chapter 3. 서버 컴포넌트와 클라이언트 컴포넌트

### 3.1 React Server Components (RSC)

App Router에서 모든 컴포넌트는 기본적으로 서버 컴포넌트다. 서버에서만 실행되며 브라우저로 JavaScript가 전송되지 않는다.

```tsx
// app/users/page.tsx — 서버 컴포넌트 (기본)
// 'use client' 없으면 자동으로 서버 컴포넌트

import { db } from '@/lib/db';

export default async function UsersPage() {
  // 서버에서 직접 DB 접근 가능!
  const users = await db.user.findMany({
    orderBy: { createdAt: 'desc' },
    take: 20,
  });

  return (
    <section>
      <h1>사용자 목록</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            {user.name} — {user.email}
          </li>
        ))}
      </ul>
    </section>
  );
}
```

**서버 컴포넌트에서 할 수 있는 것:**
- `async/await`으로 직접 데이터 페칭
- DB, 파일 시스템 등 서버 리소스 직접 접근
- API 키 등 민감 정보 안전하게 사용
- 대용량 의존성을 서버에서만 사용 (클라이언트 번들 감소)

**서버 컴포넌트에서 할 수 없는 것:**
- `useState`, `useEffect` 등 React 훅 사용
- 브라우저 API 접근 (`window`, `document`)
- 이벤트 핸들러 (`onClick`, `onChange`)
- `createContext` 사용

### 3.2 클라이언트 컴포넌트

```tsx
// components/counter.tsx — 클라이언트 컴포넌트
'use client'; // 파일 최상단에 반드시 선언

import { useState, useTransition } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  const handleIncrement = () => {
    startTransition(() => {
      setCount((prev) => prev + 1);
    });
  };

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={handleIncrement} disabled={isPending}>
        {isPending ? '업데이트 중...' : '+1'}
      </button>
    </div>
  );
}
```

### 3.3 서버/클라이언트 컴포넌트 조합 패턴

```tsx
// app/dashboard/page.tsx — 서버 컴포넌트
import { db } from '@/lib/db';
import DashboardClient from './dashboard-client';
import AnalyticsChart from './analytics-chart';

export default async function DashboardPage() {
  // 서버에서 데이터 페칭
  const stats = await db.stats.findFirst();
  const recentOrders = await db.order.findMany({ take: 10 });

  return (
    <div>
      <h1>대시보드</h1>
      {/* 서버 컴포넌트 → 클라이언트 컴포넌트로 데이터 전달 */}
      <DashboardClient initialStats={stats} />
      {/* 서버 컴포넌트를 클라이언트 컴포넌트의 children으로 전달 */}
      <AnalyticsChart>
        <RecentOrdersTable orders={recentOrders} />
      </AnalyticsChart>
    </div>
  );
}
```

```tsx
// app/dashboard/dashboard-client.tsx
'use client';

import { useState } from 'react';

interface Stats {
  totalUsers: number;
  totalRevenue: number;
  activeUsers: number;
}

export default function DashboardClient({
  initialStats,
}: {
  initialStats: Stats;
}) {
  const [stats, setStats] = useState(initialStats);
  const [period, setPeriod] = useState<'day' | 'week' | 'month'>('day');

  return (
    <div>
      <select
        value={period}
        onChange={(e) => setPeriod(e.target.value as typeof period)}
      >
        <option value="day">일간</option>
        <option value="week">주간</option>
        <option value="month">월간</option>
      </select>
      <div className="grid grid-cols-3 gap-4">
        <StatCard label="총 사용자" value={stats.totalUsers} />
        <StatCard label="총 매출" value={`₩${stats.totalRevenue.toLocaleString()}`} />
        <StatCard label="활성 사용자" value={stats.activeUsers} />
      </div>
    </div>
  );
}
```

### 3.4 컴포지션 패턴: 서버 컴포넌트를 children으로 전달

```tsx
// components/client-wrapper.tsx
'use client';

import { useState } from 'react';

export default function ClientWrapper({
  children,
}: {
  children: React.ReactNode;
}) {
  const [isOpen, setIsOpen] = useState(true);

  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>
        {isOpen ? '접기' : '펼치기'}
      </button>
      {isOpen && children}
      {/* children으로 전달된 서버 컴포넌트는 서버에서 렌더링됨 */}
    </div>
  );
}
```

```tsx
// app/page.tsx — 서버 컴포넌트
import ClientWrapper from '@/components/client-wrapper';
import ServerHeavyContent from '@/components/server-heavy-content';

export default function Page() {
  return (
    <ClientWrapper>
      {/* ServerHeavyContent는 서버에서 렌더링 → HTML로 전달 */}
      <ServerHeavyContent />
    </ClientWrapper>
  );
}
```

---

## Chapter 4. 데이터 페칭

### 4.1 서버 컴포넌트에서의 데이터 페칭

```tsx
// app/posts/page.tsx
// Next.js 15에서 fetch는 기본적으로 캐시되지 않음 (no-store)

interface Post {
  id: number;
  title: string;
  body: string;
}

export default async function PostsPage() {
  // 기본: 캐시 없음 (매 요청마다 새로 페칭)
  const res = await fetch('https://jsonplaceholder.typicode.com/posts');
  const posts: Post[] = await res.json();

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 4.2 캐싱 전략

```tsx
// 1. 캐시 없음 (기본값, Next.js 15+)
const data = await fetch('https://api.example.com/data');

// 2. 강제 캐시 (빌드 시 한 번만 페칭, SSG와 유사)
const data = await fetch('https://api.example.com/data', {
  cache: 'force-cache',
});

// 3. 시간 기반 재검증 (ISR)
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }, // 1시간마다 재검증
});

// 4. 태그 기반 재검증
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});
```

### 4.3 세그먼트 레벨 캐싱 설정

```tsx
// app/blog/page.tsx

// 이 세그먼트의 모든 fetch에 대한 기본 재검증 시간
export const revalidate = 3600; // 1시간

// 동적/정적 강제 설정
export const dynamic = 'force-dynamic';   // 항상 SSR
// export const dynamic = 'force-static'; // 항상 SSG
// export const dynamic = 'auto';         // 자동 (기본)
// export const dynamic = 'error';        // 동적 사용 시 빌드 에러

// Runtime 선택
export const runtime = 'nodejs'; // 또는 'edge'
```

### 4.4 데이터 페칭 함수 분리와 `cache()`

```tsx
// lib/data.ts
import { cache } from 'react';
import { db } from './db';

// React cache()로 동일 렌더 트리 내 중복 요청 제거
export const getUser = cache(async (id: string) => {
  const user = await db.user.findUnique({
    where: { id },
    include: { posts: true, profile: true },
  });
  return user;
});

export const getPostsByUser = cache(async (userId: string) => {
  return db.post.findMany({
    where: { authorId: userId },
    orderBy: { createdAt: 'desc' },
  });
});
```

```tsx
// app/user/[id]/page.tsx
import { getUser } from '@/lib/data';

export default async function UserPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const user = await getUser(id);

  if (!user) return <div>사용자를 찾을 수 없습니다.</div>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.profile?.bio}</p>
    </div>
  );
}

// 같은 렌더 트리 내 다른 컴포넌트에서 getUser(id)를 호출해도
// DB 쿼리는 한 번만 실행됨 (React cache)
```

### 4.5 `unstable_cache` (서버 사이드 데이터 캐시)

```tsx
import { unstable_cache } from 'next/cache';
import { db } from '@/lib/db';

// fetch가 아닌 DB 쿼리 등에 대한 캐싱
const getCachedPosts = unstable_cache(
  async (category: string) => {
    return db.post.findMany({
      where: { category },
      orderBy: { createdAt: 'desc' },
    });
  },
  ['posts-by-category'], // 캐시 키
  {
    revalidate: 3600,     // 1시간 재검증
    tags: ['posts'],      // 태그 기반 무효화에 사용
  }
);
```

### 4.6 재검증 (Revalidation)

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { secret, path, tag } = await request.json();

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  // 경로 기반 재검증
  if (path) {
    revalidatePath(path);        // 특정 경로
    revalidatePath(path, 'layout'); // 레이아웃 포함
    revalidatePath(path, 'page');   // 페이지만
  }

  // 태그 기반 재검증
  if (tag) {
    revalidateTag(tag);
  }

  return NextResponse.json({ revalidated: true, now: Date.now() });
}
```

---

## Chapter 5. Server Actions

### 5.1 기본 Server Actions

Server Actions는 서버에서 실행되는 비동기 함수로, 폼 제출 및 데이터 변경에 사용된다.

```tsx
// app/actions/post-actions.ts
'use server'; // 이 파일의 모든 export가 Server Action이 됨

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

// 입력 검증 스키마
const CreatePostSchema = z.object({
  title: z.string().min(1, '제목을 입력하세요').max(100),
  content: z.string().min(10, '내용은 10자 이상이어야 합니다'),
  category: z.enum(['tech', 'life', 'news']),
});

export type PostFormState = {
  errors?: {
    title?: string[];
    content?: string[];
    category?: string[];
  };
  message?: string;
  success?: boolean;
};

export async function createPost(
  prevState: PostFormState,
  formData: FormData
): Promise<PostFormState> {
  // 1. 검증
  const validatedFields = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    category: formData.get('category'),
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
      message: '입력값을 확인해주세요.',
    };
  }

  // 2. DB 저장
  try {
    await db.post.create({
      data: validatedFields.data,
    });
  } catch (error) {
    return { message: '게시글 생성에 실패했습니다.' };
  }

  // 3. 캐시 무효화 & 리다이렉트
  revalidatePath('/blog');
  redirect('/blog');
}

export async function deletePost(postId: string) {
  await db.post.delete({ where: { id: postId } });
  revalidatePath('/blog');
}
```

### 5.2 `useActionState`로 폼 상태 관리

```tsx
// app/blog/new/page.tsx
'use client';

import { useActionState } from 'react';
import { createPost, type PostFormState } from '@/app/actions/post-actions';
import { SubmitButton } from './submit-button';

const initialState: PostFormState = {
  errors: {},
  message: '',
};

export default function NewPostPage() {
  const [state, formAction] = useActionState(createPost, initialState);

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="title">제목</label>
        <input
          id="title"
          name="title"
          type="text"
          placeholder="게시글 제목"
        />
        {state.errors?.title && (
          <p className="text-red-500">{state.errors.title[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="content">내용</label>
        <textarea
          id="content"
          name="content"
          rows={10}
          placeholder="내용을 입력하세요"
        />
        {state.errors?.content && (
          <p className="text-red-500">{state.errors.content[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="category">카테고리</label>
        <select id="category" name="category">
          <option value="tech">기술</option>
          <option value="life">생활</option>
          <option value="news">뉴스</option>
        </select>
      </div>

      {state.message && (
        <p className="text-red-500">{state.message}</p>
      )}

      <SubmitButton />
    </form>
  );
}
```

```tsx
// app/blog/new/submit-button.tsx
'use client';

import { useFormStatus } from 'react-dom';

export function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? '저장 중...' : '게시글 작성'}
    </button>
  );
}
```

### 5.3 낙관적 업데이트 (Optimistic Updates)

```tsx
// components/like-button.tsx
'use client';

import { useOptimistic, useTransition } from 'react';
import { toggleLike } from '@/app/actions/like-actions';

interface LikeButtonProps {
  postId: string;
  initialLikes: number;
  isLiked: boolean;
}

export function LikeButton({ postId, initialLikes, isLiked }: LikeButtonProps) {
  const [isPending, startTransition] = useTransition();
  const [optimisticState, addOptimistic] = useOptimistic(
    { likes: initialLikes, isLiked },
    (current, _newAction: 'toggle') => ({
      likes: current.isLiked ? current.likes - 1 : current.likes + 1,
      isLiked: !current.isLiked,
    })
  );

  const handleLike = () => {
    startTransition(async () => {
      addOptimistic('toggle');
      await toggleLike(postId);
    });
  };

  return (
    <button onClick={handleLike} disabled={isPending}>
      {optimisticState.isLiked ? '❤️' : '🤍'} {optimisticState.likes}
    </button>
  );
}
```

---

## Chapter 6. 로딩, 에러 처리, Streaming

### 6.1 `loading.tsx`

```tsx
// app/dashboard/loading.tsx
// 이 파일은 Suspense 경계를 자동으로 생성
export default function DashboardLoading() {
  return (
    <div className="animate-pulse space-y-4">
      <div className="h-8 w-48 bg-gray-200 rounded" />
      <div className="grid grid-cols-3 gap-4">
        {[1, 2, 3].map((i) => (
          <div key={i} className="h-32 bg-gray-200 rounded" />
        ))}
      </div>
      <div className="h-64 bg-gray-200 rounded" />
    </div>
  );
}
```

### 6.2 `error.tsx`

```tsx
// app/dashboard/error.tsx
'use client'; // Error 컴포넌트는 반드시 클라이언트 컴포넌트

import { useEffect } from 'react';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // 에러 리포팅 서비스로 전송
    console.error(error);
  }, [error]);

  return (
    <div className="p-8 text-center">
      <h2>문제가 발생했습니다</h2>
      <p>{error.message}</p>
      {error.digest && (
        <p className="text-sm text-gray-500">오류 코드: {error.digest}</p>
      )}
      <button
        onClick={() => reset()} // 세그먼트 리렌더링 시도
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        다시 시도
      </button>
    </div>
  );
}
```

### 6.3 Streaming과 Suspense

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import RevenueChart from './revenue-chart';
import LatestOrders from './latest-orders';
import TopProducts from './top-products';

export default function DashboardPage() {
  return (
    <div>
      <h1>대시보드</h1>

      {/* 각 섹션이 독립적으로 스트리밍됨 */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart /> {/* 느린 쿼리 — 나중에 도착 */}
      </Suspense>

      <div className="grid grid-cols-2 gap-4">
        <Suspense fallback={<TableSkeleton />}>
          <LatestOrders /> {/* 별도 스트리밍 */}
        </Suspense>

        <Suspense fallback={<ListSkeleton />}>
          <TopProducts /> {/* 별도 스트리밍 */}
        </Suspense>
      </div>
    </div>
  );
}

// 각 컴포넌트는 서버 컴포넌트로, 독립적으로 데이터 페칭
async function RevenueChart() {
  const data = await getRevenueData(); // 2초 소요
  return <Chart data={data} />;
}

async function LatestOrders() {
  const orders = await getLatestOrders(); // 1초 소요
  return <OrderTable orders={orders} />;
}

async function TopProducts() {
  const products = await getTopProducts(); // 0.5초 소요
  return <ProductList products={products} />;
}
```

### 6.4 `not-found.tsx`

```tsx
// app/not-found.tsx — 전역 404 페이지
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="text-center py-20">
      <h2 className="text-4xl font-bold">404</h2>
      <p className="mt-4">요청하신 페이지를 찾을 수 없습니다.</p>
      <Link href="/" className="text-blue-500 underline mt-4 inline-block">
        홈으로 돌아가기
      </Link>
    </div>
  );
}
```

```tsx
// app/blog/[slug]/page.tsx 에서 수동으로 404 트리거
import { notFound } from 'next/navigation';

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await getPost(slug);

  if (!post) {
    notFound(); // 가장 가까운 not-found.tsx 렌더링
  }

  return <article>{/* ... */}</article>;
}
```

---

## Chapter 7. Route Handlers (API Routes)

### 7.1 기본 Route Handler

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';

// GET /api/users
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '10');
  const skip = (page - 1) * limit;

  const [users, total] = await Promise.all([
    db.user.findMany({ skip, take: limit, orderBy: { createdAt: 'desc' } }),
    db.user.count(),
  ]);

  return NextResponse.json({
    data: users,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  });
}

// POST /api/users
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    const user = await db.user.create({
      data: {
        name: body.name,
        email: body.email,
      },
    });

    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create user' },
      { status: 500 }
    );
  }
}
```

### 7.2 동적 Route Handler

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/users/:id
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await db.user.findUnique({ where: { id } });

  if (!user) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }

  return NextResponse.json(user);
}

// PATCH /api/users/:id
export async function PATCH(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const body = await request.json();

  const user = await db.user.update({
    where: { id },
    data: body,
  });

  return NextResponse.json(user);
}

// DELETE /api/users/:id
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  await db.user.delete({ where: { id } });
  return new NextResponse(null, { status: 204 });
}
```

### 7.3 Streaming Response

```typescript
// app/api/chat/route.ts
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { message } = await request.json();

  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      // AI API 등으로부터 스트리밍 응답 시뮬레이션
      const words = message.split(' ');
      for (const word of words) {
        controller.enqueue(encoder.encode(`data: ${word}\n\n`));
        await new Promise((r) => setTimeout(r, 100));
      }
      controller.enqueue(encoder.encode('data: [DONE]\n\n'));
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  });
}
```

### 7.4 Route Handler에서 쿠키, 헤더 접근

```typescript
// app/api/auth/route.ts
import { cookies, headers } from 'next/headers';
import { NextResponse } from 'next/server';

export async function GET() {
  const cookieStore = await cookies();
  const headersList = await headers();

  const token = cookieStore.get('session-token');
  const userAgent = headersList.get('user-agent');
  const ip = headersList.get('x-forwarded-for');

  return NextResponse.json({ token: token?.value, userAgent, ip });
}

export async function POST() {
  const cookieStore = await cookies();

  // 쿠키 설정
  cookieStore.set('session-token', 'abc123', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7, // 7일
    path: '/',
  });

  return NextResponse.json({ success: true });
}
```

---

## Chapter 8. Middleware

### 8.1 미들웨어 기본

```typescript
// src/middleware.ts (프로젝트 루트 또는 src/ 아래)
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 1. 인증 체크
  const token = request.cookies.get('session-token')?.value;

  if (pathname.startsWith('/dashboard') && !token) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }

  // 2. 국제화 (i18n) 리다이렉트
  const locale = request.headers.get('accept-language')?.split(',')[0]?.split('-')[0];
  if (pathname === '/' && locale === 'ko') {
    return NextResponse.redirect(new URL('/ko', request.url));
  }

  // 3. 응답 헤더 추가
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'my-value');
  response.headers.set('x-request-id', crypto.randomUUID());

  return response;
}

// 미들웨어가 실행될 경로 지정
export const config = {
  matcher: [
    // 정적 파일과 API 제외한 모든 경로
    '/((?!_next/static|_next/image|favicon.ico|api).*)',
    // 특정 경로만 지정할 수도 있음
    '/dashboard/:path*',
    '/admin/:path*',
  ],
};
```

### 8.2 고급 미들웨어: Rate Limiting

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

// 간단한 인메모리 rate limiter (프로덕션에서는 Redis 사용)
const rateLimit = new Map<string, { count: number; resetTime: number }>();

function checkRateLimit(ip: string, limit: number, windowMs: number): boolean {
  const now = Date.now();
  const record = rateLimit.get(ip);

  if (!record || now > record.resetTime) {
    rateLimit.set(ip, { count: 1, resetTime: now + windowMs });
    return true;
  }

  if (record.count >= limit) return false;

  record.count++;
  return true;
}

export function middleware(request: NextRequest) {
  // API 라우트에 rate limiting 적용
  if (request.nextUrl.pathname.startsWith('/api')) {
    const ip = request.headers.get('x-forwarded-for') || 'unknown';
    const allowed = checkRateLimit(ip, 100, 60_000); // 분당 100회

    if (!allowed) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429, headers: { 'Retry-After': '60' } }
      );
    }
  }

  return NextResponse.next();
}
```

---

## Chapter 9. 메타데이터와 SEO

### 9.1 정적 메타데이터

```tsx
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://mysite.com'),
  title: {
    default: 'My Site',
    template: '%s | My Site', // 하위 페이지: "About | My Site"
  },
  description: 'Next.js 풀스택 웹 애플리케이션',
  keywords: ['Next.js', 'React', 'TypeScript'],
  authors: [{ name: 'Your Name', url: 'https://yoursite.com' }],
  creator: 'Your Name',
  openGraph: {
    type: 'website',
    locale: 'ko_KR',
    url: 'https://mysite.com',
    siteName: 'My Site',
    title: 'My Site',
    description: 'Next.js 풀스택 웹 애플리케이션',
    images: [
      {
        url: '/og-image.png',
        width: 1200,
        height: 630,
        alt: 'My Site',
      },
    ],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'My Site',
    description: 'Next.js 풀스택 웹 애플리케이션',
    images: ['/og-image.png'],
    creator: '@yourhandle',
  },
  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      'max-video-preview': -1,
      'max-image-preview': 'large',
      'max-snippet': -1,
    },
  },
  verification: {
    google: 'google-verification-code',
    yandex: 'yandex-verification-code',
  },
};
```

### 9.2 동적 메타데이터

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next';

type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

export async function generateMetadata(
  { params, searchParams }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  const parentMetadata = await parent;
  const previousImages = parentMetadata.openGraph?.images || [];

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage, ...previousImages],
      type: 'article',
      publishedTime: post.publishedAt,
      authors: [post.author.name],
    },
  };
}
```

### 9.3 sitemap.xml & robots.txt

```typescript
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getAllPosts();
  const postEntries = posts.map((post) => ({
    url: `https://mysite.com/blog/${post.slug}`,
    lastModified: post.updatedAt,
    changeFrequency: 'weekly' as const,
    priority: 0.8,
  }));

  return [
    {
      url: 'https://mysite.com',
      lastModified: new Date(),
      changeFrequency: 'daily',
      priority: 1,
    },
    {
      url: 'https://mysite.com/about',
      lastModified: new Date(),
      changeFrequency: 'monthly',
      priority: 0.5,
    },
    ...postEntries,
  ];
}
```

```typescript
// app/robots.ts
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: ['/admin/', '/api/', '/private/'],
      },
      {
        userAgent: 'Googlebot',
        allow: '/',
      },
    ],
    sitemap: 'https://mysite.com/sitemap.xml',
  };
}
```

---

## Chapter 10. 이미지, 폰트, 스크립트 최적화

### 10.1 `next/image`

```tsx
import Image from 'next/image';

// 로컬 이미지 (자동 width/height 추론)
import profilePic from '@/public/profile.jpg';

export default function Profile() {
  return (
    <div>
      {/* 로컬 이미지 */}
      <Image
        src={profilePic}
        alt="프로필 사진"
        placeholder="blur" // 자동 블러 플레이스홀더
        priority           // LCP 이미지에 사용 (프리로드)
        quality={85}
      />

      {/* 리모트 이미지 */}
      <Image
        src="https://cdn.example.com/photo.jpg"
        alt="외부 사진"
        width={800}
        height={600}
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
        loading="lazy"     // 기본값
      />

      {/* fill 모드 (부모 요소 기준) */}
      <div className="relative w-full h-64">
        <Image
          src="/hero.jpg"
          alt="히어로 이미지"
          fill
          style={{ objectFit: 'cover' }}
          sizes="100vw"
        />
      </div>
    </div>
  );
}
```

### 10.2 `next/font`

```tsx
// app/layout.tsx
import { Inter, Noto_Sans_KR } from 'next/font/google';
import localFont from 'next/font/local';

// Google Fonts (자동 최적화, self-hosting)
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

const notoSansKR = Noto_Sans_KR({
  subsets: ['latin'],
  weight: ['400', '700'],
  variable: '--font-noto',
});

// 로컬 폰트
const pretendard = localFont({
  src: [
    { path: '../fonts/Pretendard-Regular.woff2', weight: '400' },
    { path: '../fonts/Pretendard-Bold.woff2', weight: '700' },
  ],
  variable: '--font-pretendard',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html
      lang="ko"
      className={`${inter.variable} ${notoSansKR.variable} ${pretendard.variable}`}
    >
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

### 10.3 `next/script`

```tsx
import Script from 'next/script';

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      {/* beforeInteractive: _document에 삽입 (최고 우선순위) */}
      <Script
        src="https://polyfill.io/v3/polyfill.min.js"
        strategy="beforeInteractive"
      />

      {/* afterInteractive: 페이지 하이드레이션 후 (기본값, GA 등) */}
      <Script
        src="https://www.googletagmanager.com/gtag/js?id=G-XXXXX"
        strategy="afterInteractive"
      />
      <Script id="gtag-init" strategy="afterInteractive">
        {`
          window.dataLayer = window.dataLayer || [];
          function gtag(){dataLayer.push(arguments);}
          gtag('js', new Date());
          gtag('config', 'G-XXXXX');
        `}
      </Script>

      {/* lazyOnload: 브라우저 idle 시 로드 */}
      <Script
        src="https://connect.facebook.net/en_US/sdk.js"
        strategy="lazyOnload"
        onLoad={() => console.log('Facebook SDK loaded')}
      />

      {children}
    </>
  );
}
```

---

## Chapter 11. 인증 (Authentication)

### 11.1 NextAuth.js (Auth.js) 통합

```typescript
// lib/auth.ts
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { db } from './db';
import bcrypt from 'bcryptjs';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  session: { strategy: 'jwt' },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    Credentials({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;

        const user = await db.user.findUnique({
          where: { email: credentials.email as string },
        });

        if (!user || !user.hashedPassword) return null;

        const isValid = await bcrypt.compare(
          credentials.password as string,
          user.hashedPassword
        );

        if (!isValid) return null;
        return user;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
        token.id = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.role = token.role as string;
        session.user.id = token.id as string;
      }
      return session;
    },
    async authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user;
      const isOnDashboard = nextUrl.pathname.startsWith('/dashboard');
      if (isOnDashboard) return isLoggedIn;
      return true;
    },
  },
});
```

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth';
export const { GET, POST } = handlers;
```

```tsx
// 서버 컴포넌트에서 세션 접근
import { auth } from '@/lib/auth';

export default async function DashboardPage() {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  return <div>환영합니다, {session.user.name}님!</div>;
}
```

```tsx
// 클라이언트 컴포넌트에서 세션 접근
'use client';

import { useSession, signIn, signOut } from 'next-auth/react';

export function AuthButton() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <div>로딩 중...</div>;

  if (session) {
    return (
      <div>
        <p>{session.user.name}</p>
        <button onClick={() => signOut()}>로그아웃</button>
      </div>
    );
  }

  return <button onClick={() => signIn()}>로그인</button>;
}
```

---

## Chapter 12. 데이터베이스 통합

### 12.1 Prisma ORM 설정

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id             String   @id @default(cuid())
  name           String?
  email          String   @unique
  hashedPassword String?
  role           Role     @default(USER)
  posts          Post[]
  profile        Profile?
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
}

model Post {
  id          String     @id @default(cuid())
  title       String
  content     String?
  published   Boolean    @default(false)
  author      User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId    String
  categories  Category[]
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([authorId])
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}

model Profile {
  id     String  @id @default(cuid())
  bio    String?
  avatar String?
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId String  @unique
}

enum Role {
  USER
  ADMIN
}
```

### 12.2 Prisma 클라이언트 싱글턴

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = db;
}
```

### 12.3 Drizzle ORM 대안

```typescript
// lib/db-drizzle.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
```

```typescript
// lib/schema.ts
import { pgTable, text, boolean, timestamp, pgEnum } from 'drizzle-orm/pg-core';

export const roleEnum = pgEnum('role', ['USER', 'ADMIN']);

export const users = pgTable('users', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  name: text('name'),
  email: text('email').notNull().unique(),
  role: roleEnum('role').default('USER'),
  createdAt: timestamp('created_at').defaultNow(),
});

export const posts = pgTable('posts', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false),
  authorId: text('author_id').references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow(),
});
```

---

## Chapter 13. Partial Prerendering (PPR)

PPR은 Next.js의 핵심 혁신 기능으로, 하나의 페이지 안에서 정적 부분과 동적 부분을 분리하여, 정적 셸은 즉시 전달하고 동적 부분만 스트리밍하는 방식이다.

```tsx
// next.config.ts
const nextConfig = {
  experimental: {
    ppr: true,
  },
};
```

```tsx
// app/product/[id]/page.tsx
import { Suspense } from 'react';
import { ProductInfo, ProductReviews, RecommendedProducts } from './components';

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return (
    <div>
      {/* 정적 셸: 빌드 시 사전 렌더링 */}
      <header>
        <h1>제품 상세</h1>
        <nav>홈 &gt; 제품</nav>
      </header>

      {/* 동적 부분: Suspense 경계 안의 동적 데이터 */}
      <Suspense fallback={<ProductInfoSkeleton />}>
        <ProductInfo id={id} />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews id={id} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <RecommendedProducts id={id} />
      </Suspense>

      {/* 정적 부분 */}
      <footer>© 2025 My Store</footer>
    </div>
  );
}
```

작동 원리: 빌드 시 정적 HTML 셸이 생성되고, Suspense 경계 내부의 동적 콘텐츠는 "구멍(hole)"으로 남겨진다. 요청 시 정적 셸은 즉시 전달(CDN 캐시 가능)되고, 동적 콘텐츠는 서버에서 스트리밍으로 채워진다. 이를 통해 TTFB(Time to First Byte)는 정적 사이트 수준, 데이터 신선도는 SSR 수준을 동시에 달성한다.

---

## Chapter 14. 국제화 (i18n)

### 14.1 서브패스 기반 i18n

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { match } from '@formatjs/intl-localematcher';
import Negotiator from 'negotiator';

const locales = ['en', 'ko', 'ja'];
const defaultLocale = 'ko';

function getLocale(request: NextRequest): string {
  const headers = { 'accept-language': request.headers.get('accept-language') || '' };
  const languages = new Negotiator({ headers }).languages();
  return match(languages, locales, defaultLocale);
}

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 이미 로케일이 있는지 확인
  const pathnameHasLocale = locales.some(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (pathnameHasLocale) return;

  // 로케일 감지 후 리다이렉트
  const locale = getLocale(request);
  request.nextUrl.pathname = `/${locale}${pathname}`;
  return NextResponse.redirect(request.nextUrl);
}

export const config = {
  matcher: ['/((?!_next|api|favicon.ico).*)'],
};
```

```
app/
└── [lang]/
    ├── layout.tsx
    ├── page.tsx
    └── dictionaries.ts
```

```typescript
// app/[lang]/dictionaries.ts
const dictionaries = {
  en: () => import('@/dictionaries/en.json').then((m) => m.default),
  ko: () => import('@/dictionaries/ko.json').then((m) => m.default),
  ja: () => import('@/dictionaries/ja.json').then((m) => m.default),
};

export const getDictionary = async (locale: string) => {
  return dictionaries[locale as keyof typeof dictionaries]();
};
```

```json
// dictionaries/ko.json
{
  "navigation": {
    "home": "홈",
    "about": "소개",
    "blog": "블로그"
  },
  "home": {
    "title": "환영합니다",
    "description": "Next.js로 만든 웹 애플리케이션입니다."
  }
}
```

```tsx
// app/[lang]/page.tsx
import { getDictionary } from './dictionaries';

export default async function HomePage({
  params,
}: {
  params: Promise<{ lang: string }>;
}) {
  const { lang } = await params;
  const dict = await getDictionary(lang);

  return (
    <div>
      <h1>{dict.home.title}</h1>
      <p>{dict.home.description}</p>
    </div>
  );
}

export async function generateStaticParams() {
  return [{ lang: 'en' }, { lang: 'ko' }, { lang: 'ja' }];
}
```

---

## Chapter 15. 테스팅

### 15.1 Jest + React Testing Library

```typescript
// __tests__/components/counter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from '@/components/counter';

describe('Counter', () => {
  it('초기 카운트가 0이다', () => {
    render(<Counter />);
    expect(screen.getByText('카운트: 0')).toBeInTheDocument();
  });

  it('버튼 클릭 시 카운트가 증가한다', async () => {
    render(<Counter />);
    const button = screen.getByRole('button', { name: /\+1/ });
    fireEvent.click(button);
    // useTransition 사용 시 비동기 대기
    expect(await screen.findByText('카운트: 1')).toBeInTheDocument();
  });
});
```

### 15.2 E2E 테스트 (Playwright)

```typescript
// e2e/blog.spec.ts
import { test, expect } from '@playwright/test';

test.describe('블로그', () => {
  test('게시글 목록이 표시된다', async ({ page }) => {
    await page.goto('/blog');
    await expect(page.getByRole('heading', { name: '블로그' })).toBeVisible();
    const posts = page.getByRole('listitem');
    await expect(posts).toHaveCount(10);
  });

  test('게시글 작성 후 목록에 표시된다', async ({ page }) => {
    await page.goto('/blog/new');
    await page.getByLabel('제목').fill('테스트 게시글');
    await page.getByLabel('내용').fill('이것은 테스트 게시글의 내용입니다.');
    await page.getByRole('button', { name: '게시글 작성' }).click();

    // 리다이렉트 후 목록에서 확인
    await expect(page).toHaveURL('/blog');
    await expect(page.getByText('테스트 게시글')).toBeVisible();
  });

  test('모바일 반응형 레이아웃', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/blog');
    await expect(page.getByRole('navigation')).toBeVisible();
  });
});
```

### 15.3 Server Actions 테스트

```typescript
// __tests__/actions/post-actions.test.ts
import { createPost } from '@/app/actions/post-actions';

// 모킹
jest.mock('@/lib/db', () => ({
  db: {
    post: {
      create: jest.fn(),
    },
  },
}));

jest.mock('next/cache', () => ({
  revalidatePath: jest.fn(),
}));

jest.mock('next/navigation', () => ({
  redirect: jest.fn(),
}));

describe('createPost', () => {
  it('유효하지 않은 입력에 에러를 반환한다', async () => {
    const formData = new FormData();
    formData.set('title', '');
    formData.set('content', 'short');
    formData.set('category', 'tech');

    const result = await createPost({}, formData);
    expect(result.errors?.title).toBeDefined();
  });
});
```

---

## Chapter 16. 배포와 성능 최적화

### 16.1 Vercel 배포

```bash
# Vercel CLI
npm i -g vercel
vercel          # 프리뷰 배포
vercel --prod   # 프로덕션 배포
```

### 16.2 Docker 배포

```dockerfile
# Dockerfile (standalone output)
FROM node:20-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN corepack enable pnpm && pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "server.js"]
```

```typescript
// next.config.ts (standalone 빌드 활성화)
const nextConfig = {
  output: 'standalone',
};
```

### 16.3 성능 최적화 체크리스트

**번들 사이즈 최적화:**
- `@next/bundle-analyzer`로 번들 분석
- 동적 임포트(`dynamic()`) 활용
- Tree shaking을 위한 barrel export 제거
- `'use client'` 경계를 가능한 한 리프 컴포넌트에 배치

```tsx
// 동적 임포트
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/heavy-chart'), {
  loading: () => <p>차트 로딩 중...</p>,
  ssr: false, // 클라이언트에서만 로드
});
```

**데이터 페칭 최적화:**
- 병렬 데이터 페칭 (`Promise.all`)
- React `cache()`로 중복 제거
- 적절한 재검증 전략 설정
- Streaming으로 점진적 렌더링

```tsx
// 병렬 데이터 페칭
async function Dashboard() {
  // BAD: 순차 실행 (워터폴)
  // const users = await getUsers();
  // const posts = await getPosts();

  // GOOD: 병렬 실행
  const [users, posts, analytics] = await Promise.all([
    getUsers(),
    getPosts(),
    getAnalytics(),
  ]);

  return <DashboardView users={users} posts={posts} analytics={analytics} />;
}
```

---

## Chapter 17. 고급 패턴

### 17.1 React Compiler (자동 메모이제이션)

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    reactCompiler: true,
  },
};
```

React Compiler가 활성화되면 `useMemo`, `useCallback`, `React.memo`를 수동으로 작성하지 않아도 컴파일러가 자동으로 최적화를 수행한다.

### 17.2 Server-Sent Events (SSE)와 실시간 기능

```typescript
// app/api/events/route.ts
export const dynamic = 'force-dynamic';

export async function GET() {
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    start(controller) {
      const interval = setInterval(() => {
        const data = JSON.stringify({
          time: new Date().toISOString(),
          value: Math.random(),
        });
        controller.enqueue(encoder.encode(`data: ${data}\n\n`));
      }, 1000);

      // 클라이언트 연결 해제 시 정리
      return () => clearInterval(interval);
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  });
}
```

```tsx
// components/live-data.tsx
'use client';

import { useEffect, useState } from 'react';

export function LiveData() {
  const [data, setData] = useState<{ time: string; value: number } | null>(null);

  useEffect(() => {
    const eventSource = new EventSource('/api/events');

    eventSource.onmessage = (event) => {
      setData(JSON.parse(event.data));
    };

    eventSource.onerror = () => {
      eventSource.close();
    };

    return () => eventSource.close();
  }, []);

  return (
    <div>
      <p>시간: {data?.time}</p>
      <p>값: {data?.value?.toFixed(4)}</p>
    </div>
  );
}
```

### 17.3 타입 안전 환경 변수

```typescript
// env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(1),
  NEXTAUTH_URL: z.string().url(),
  GITHUB_CLIENT_ID: z.string().min(1),
  GITHUB_CLIENT_SECRET: z.string().min(1),
  NEXT_PUBLIC_APP_URL: z.string().url(),
});

// 빌드 타임에 검증
export const env = envSchema.parse(process.env);

// 타입 추론
export type Env = z.infer<typeof envSchema>;
```

---

# PART 2: 연습 문제 (초급 → 고급)

---

## 초급 문제

### 문제 1. 파일 기반 라우팅

**문제:** 다음 URL 경로에 해당하는 폴더/파일 구조를 작성하세요.

- `/` (홈 페이지)
- `/about` (소개 페이지)
- `/products` (제품 목록)
- `/products/123` (개별 제품 — 동적)
- `/blog/2024/01/hello-world` (연/월/슬러그 — catch-all)

**답안:**

```
app/
├── page.tsx                          → /
├── about/
│   └── page.tsx                      → /about
├── products/
│   ├── page.tsx                      → /products
│   └── [id]/
│       └── page.tsx                  → /products/:id
└── blog/
    └── [...segments]/
        └── page.tsx                  → /blog/2024/01/hello-world
```

**해설:** `[id]`는 단일 동적 세그먼트, `[...segments]`는 catch-all 세그먼트다. catch-all 라우트에서 `segments`는 `['2024', '01', 'hello-world']` 배열로 전달된다. 선택적 catch-all `[[...segments]]`는 `/blog` 경로도 매칭하지만, `[...segments]`는 최소 하나의 세그먼트가 필요하다.

---

### 문제 2. 서버 vs 클라이언트 컴포넌트

**문제:** 아래 각 시나리오에서 서버 컴포넌트와 클라이언트 컴포넌트 중 어떤 것을 사용해야 하는지 고르고 이유를 설명하세요.

1. DB에서 사용자 목록을 가져와 표시하는 컴포넌트
2. 검색 입력창과 실시간 필터링 기능
3. 마크다운 파일을 읽어 HTML로 변환하여 표시하는 블로그 포스트
4. 드래그 앤 드롭으로 순서를 변경하는 카드 보드

**답안:**

1. **서버 컴포넌트** — DB 직접 접근이 필요하고, 인터랙션이 없으므로 서버에서 렌더링하면 JS 번들을 줄일 수 있다.
2. **클라이언트 컴포넌트** — `useState`로 입력 상태를 관리하고 `onChange` 이벤트를 처리해야 한다.
3. **서버 컴포넌트** — 파일 시스템 접근(`fs`)이 필요하고, 마크다운 파싱 라이브러리(remark 등)의 번들을 클라이언트에 보낼 필요가 없다.
4. **클라이언트 컴포넌트** — 드래그 앤 드롭은 브라우저 이벤트(drag/drop)와 상태 관리가 필수적이다.

---

### 문제 3. loading.tsx의 역할

**문제:** `loading.tsx`는 내부적으로 어떤 React 기능을 사용하는지 설명하고, 이것이 없을 때와 있을 때의 차이를 서술하세요.

**답안:**

`loading.tsx`는 내부적으로 React `<Suspense>` 경계를 자동 생성한다. Next.js가 해당 세그먼트의 `page.tsx`를 `<Suspense fallback={<Loading />}>`로 감싸는 것과 동일하다.

- **없을 때:** 서버 컴포넌트의 데이터 페칭이 완료될 때까지 사용자에게 아무것도 표시되지 않는다(빈 화면 또는 상위 레이아웃만 보임).
- **있을 때:** 데이터 페칭 중에도 즉시 loading UI가 표시되고, 데이터가 준비되면 실제 콘텐츠로 교체된다. Streaming을 통해 점진적으로 페이지가 채워진다.

---

## 중급 문제

### 문제 4. 캐싱 전략 설계

**문제:** 전자상거래 사이트에서 다음 데이터에 대한 최적의 캐싱 전략을 각각 제시하세요.

1. 제품 카탈로그 (하루에 몇 번 업데이트)
2. 사용자 장바구니
3. 제품 리뷰 (분당 수십 건 추가)
4. 메인 페이지 배너

**답안:**

```tsx
// 1. 제품 카탈로그: ISR (시간 기반 재검증)
const products = await fetch('https://api.store.com/products', {
  next: { revalidate: 3600, tags: ['products'] },
});
// 1시간 캐시 + 제품 업데이트 시 태그 기반 수동 재검증 병행

// 2. 장바구니: 캐시 없음 (항상 동적)
// 사용자별 데이터이므로 캐시하면 안 됨
export const dynamic = 'force-dynamic';
const cart = await getCart(userId); // 매 요청마다 새로 조회

// 3. 제품 리뷰: 짧은 ISR + 태그 재검증
const reviews = await fetch(`/api/reviews/${productId}`, {
  next: { revalidate: 60, tags: [`reviews-${productId}`] },
});
// 새 리뷰 작성 시 revalidateTag(`reviews-${productId}`)

// 4. 메인 배너: 긴 ISR
const banner = await fetch('/api/banner', {
  next: { revalidate: 86400, tags: ['banner'] }, // 24시간
});
// CMS 업데이트 시 webhook으로 revalidateTag('banner')
```

**해설:** 핵심은 "데이터의 변경 빈도"와 "사용자별 데이터 여부"로 결정한다. 사용자별 데이터는 절대 공유 캐시에 저장하면 안 되고, 공용 데이터는 변경 빈도에 따라 revalidate 시간을 조절한다. 태그 기반 재검증을 병행하면 긴급 업데이트에도 대응할 수 있다.

---

### 문제 5. Server Action과 유효성 검증

**문제:** 다음 요구사항을 만족하는 Server Action을 작성하세요.

- 사용자 프로필 업데이트 (이름, 이메일, 자기소개)
- Zod로 입력 검증
- 이메일 중복 체크
- 성공 시 프로필 페이지로 리다이렉트
- 에러 시 필드별 에러 메시지 반환

**답안:**

```typescript
'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { auth } from '@/lib/auth';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

const ProfileSchema = z.object({
  name: z.string().min(2, '이름은 2자 이상이어야 합니다').max(50),
  email: z.string().email('유효한 이메일을 입력하세요'),
  bio: z.string().max(500, '자기소개는 500자 이내여야 합니다').optional(),
});

export type ProfileState = {
  errors?: { name?: string[]; email?: string[]; bio?: string[] };
  message?: string;
};

export async function updateProfile(
  prevState: ProfileState,
  formData: FormData
): Promise<ProfileState> {
  const session = await auth();
  if (!session?.user?.id) {
    return { message: '인증이 필요합니다.' };
  }

  const validated = ProfileSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    bio: formData.get('bio'),
  });

  if (!validated.success) {
    return { errors: validated.error.flatten().fieldErrors };
  }

  // 이메일 중복 체크
  const existingUser = await db.user.findFirst({
    where: {
      email: validated.data.email,
      NOT: { id: session.user.id },
    },
  });

  if (existingUser) {
    return { errors: { email: ['이미 사용 중인 이메일입니다.'] } };
  }

  try {
    await db.user.update({
      where: { id: session.user.id },
      data: {
        name: validated.data.name,
        email: validated.data.email,
        profile: {
          upsert: {
            create: { bio: validated.data.bio },
            update: { bio: validated.data.bio },
          },
        },
      },
    });
  } catch {
    return { message: '프로필 업데이트에 실패했습니다.' };
  }

  revalidatePath('/profile');
  redirect('/profile');
}
```

---

### 문제 6. 병렬 라우트와 조건부 렌더링

**문제:** 관리자와 일반 사용자에 따라 같은 `/dashboard` URL에서 다른 콘텐츠를 보여주는 병렬 라우트를 설계하세요.

**답안:**

```
app/dashboard/
├── layout.tsx
├── page.tsx
├── @admin/
│   ├── page.tsx
│   └── default.tsx
└── @user/
    ├── page.tsx
    └── default.tsx
```

```tsx
// app/dashboard/layout.tsx
import { auth } from '@/lib/auth';

export default async function DashboardLayout({
  children,
  admin,
  user,
}: {
  children: React.ReactNode;
  admin: React.ReactNode;
  user: React.ReactNode;
}) {
  const session = await auth();
  const isAdmin = session?.user?.role === 'ADMIN';

  return (
    <div>
      <h1>대시보드</h1>
      {children}
      {isAdmin ? admin : user}
    </div>
  );
}
```

**해설:** 병렬 라우트(`@admin`, `@user`)는 레이아웃에서 props로 받아 조건부 렌더링할 수 있다. `default.tsx`는 초기 로드나 네비게이션 시 매칭되는 하위 라우트가 없을 때 표시되는 폴백이다. 이 패턴은 역할 기반 대시보드, A/B 테스트, 기능 플래그 등에 활용된다.

---

## 고급 문제

### 문제 7. PPR(Partial Prerendering)의 작동 원리

**문제:** PPR이 기존의 SSR, SSG, ISR과 어떻게 다른지 기술하고, PPR을 활용한 뉴스 사이트의 메인 페이지를 설계하세요. 정적/동적 부분을 명확히 구분하세요.

**답안:**

기존 방식은 페이지 단위로 렌더링 전략을 선택했다: SSG(전체 정적), SSR(전체 동적), ISR(주기적 재생성). PPR은 한 페이지 내에서 컴포넌트 단위로 정적/동적을 혼합한다.

PPR 작동 방식: 빌드 시 Suspense 경계 바깥은 정적 HTML로 사전 렌더링하고, 경계 안쪽은 "구멍"으로 남긴다. 요청 시 정적 셸은 CDN에서 즉시 전달하고, 동적 부분은 서버에서 스트리밍으로 채운다.

```tsx
// app/page.tsx — 뉴스 사이트 메인
import { Suspense } from 'react';

export default function HomePage() {
  return (
    <div>
      {/* ── 정적 영역 (빌드 시 사전 렌더링) ── */}
      <header>
        <Logo />
        <Navigation />
      </header>
      <CategoryTabs />

      {/* ── 동적 영역 (요청 시 스트리밍) ── */}
      <Suspense fallback={<BreakingNewsSkeleton />}>
        <BreakingNews /> {/* 실시간 속보 */}
      </Suspense>

      <Suspense fallback={<PersonalizedFeedSkeleton />}>
        <PersonalizedFeed /> {/* 사용자 맞춤 피드 (쿠키 기반) */}
      </Suspense>

      <div className="grid grid-cols-3">
        <div className="col-span-2">
          <Suspense fallback={<ArticleListSkeleton />}>
            <TrendingArticles /> {/* 인기 기사 */}
          </Suspense>
        </div>
        <aside>
          <Suspense fallback={<WeatherSkeleton />}>
            <WeatherWidget /> {/* 위치 기반 날씨 */}
          </Suspense>
          <Suspense fallback={<AdSkeleton />}>
            <TargetedAds /> {/* 타겟 광고 */}
          </Suspense>
        </aside>
      </div>

      {/* ── 정적 영역 ── */}
      <footer>
        <FooterLinks />
        <Copyright />
      </footer>
    </div>
  );
}
```

결과: TTFB는 정적 CDN 수준(50ms 이하), 동적 콘텐츠는 0.5~2초 내 스트리밍 완료. 사용자는 즉시 페이지 골격을 보고, 실시간 데이터가 점진적으로 채워지는 것을 경험한다.

---

### 문제 8. 마이크로프론트엔드 아키텍처

**문제:** Next.js의 Module Federation 또는 Multi-Zones를 활용한 마이크로프론트엔드 아키텍처를 설계하세요. 메인 앱, 쇼핑 앱, 블로그 앱 3개로 구성됩니다.

**답안:**

```typescript
// apps/main/next.config.ts — Multi-Zones 방식
const nextConfig = {
  async rewrites() {
    return [
      // /shop/* 요청을 쇼핑 앱으로 프록시
      {
        source: '/shop/:path*',
        destination: 'https://shop.mysite.com/shop/:path*',
      },
      // /blog/* 요청을 블로그 앱으로 프록시
      {
        source: '/blog/:path*',
        destination: 'https://blog.mysite.com/blog/:path*',
      },
    ];
  },
};
```

```typescript
// apps/shop/next.config.ts
const nextConfig = {
  basePath: '/shop',
  // 독립 배포 가능
};
```

```typescript
// apps/blog/next.config.ts
const nextConfig = {
  basePath: '/blog',
};
```

**아키텍처 설명:**

사용자는 `mysite.com`에 접속하면 메인 앱이 서빙된다. `/shop/products`로 이동하면 메인 앱의 rewrite 규칙에 의해 쇼핑 앱의 `/shop/products` 페이지가 프록시된다. 각 앱은 독립 저장소, 독립 배포, 독립 팀 운영이 가능하다.

공유 요소(헤더, 푸터, 디자인 시스템)는 npm 패키지로 배포하여 각 앱에서 사용한다. 인증 상태는 공유 쿠키 도메인(`.mysite.com`)으로 관리한다.

---

# PART 3: 글로벌 IT 기업 인터뷰 예상 문제

---

## Q1. [Google/Meta] React Server Components의 동작 원리

**문제:** RSC가 클라이언트에 전달되는 과정을 직렬화(serialization) 수준에서 설명하세요. RSC Payload란 무엇이며, 기존 SSR과 어떻게 다른가요?

**모범 답안:**

RSC에서 서버 컴포넌트는 서버에서 실행된 후 "RSC Payload"라는 특수한 바이너리 스트리밍 형식으로 직렬화된다. 이 페이로드에는 렌더링된 서버 컴포넌트의 결과(React 엘리먼트 트리), 클라이언트 컴포넌트에 대한 참조(모듈 ID + props), 그리고 서버에서 클라이언트로 전달되는 데이터가 포함된다.

기존 SSR과의 핵심 차이는 다음과 같다. 기존 SSR은 컴포넌트를 서버에서 HTML 문자열로 렌더링하고, 클라이언트에서 동일한 컴포넌트 코드를 다시 실행하여 하이드레이션(hydration)한다. 즉, 모든 컴포넌트 코드가 클라이언트로 전송된다.

RSC에서 서버 컴포넌트의 JavaScript 코드는 클라이언트로 전송되지 않는다. 서버에서 실행된 결과만 RSC Payload로 전달되며, 클라이언트는 이 payload를 React 트리에 삽입한다. 클라이언트 컴포넌트만 하이드레이션이 필요하다.

이를 통해 서버 전용 라이브러리(DB 클라이언트, 마크다운 파서 등)의 코드가 클라이언트 번들에 포함되지 않아 번들 사이즈가 크게 감소하고, 서버에서만 안전하게 민감 정보를 다룰 수 있다.

스트리밍 관점에서, RSC Payload는 청크 단위로 스트리밍되어 Suspense 경계와 결합하면 페이지의 일부가 준비되는 즉시 클라이언트에 점진적으로 전달된다.

---

## Q2. [Amazon/Netflix] 데이터 페칭 워터폴을 감지하고 해결하는 방법

**문제:** Next.js에서 데이터 페칭 워터폴(waterfall)이 발생하는 시나리오를 설명하고, 이를 감지·해결하는 전략을 코드와 함께 제시하세요.

**모범 답안:**

워터폴은 순차적 의존 관계가 없는 데이터 페칭이 직렬로 실행될 때 발생한다.

```tsx
// BAD: 워터폴 — 3초 + 2초 + 1초 = 총 6초
async function Dashboard() {
  const user = await getUser();        // 3초
  const posts = await getPosts();      // 2초 (user에 의존하지 않음!)
  const analytics = await getAnalytics(); // 1초
  return <View user={user} posts={posts} analytics={analytics} />;
}
```

**해결 전략 1: `Promise.all`로 병렬화**

```tsx
// GOOD: 병렬 — max(3, 2, 1) = 총 3초
async function Dashboard() {
  const [user, posts, analytics] = await Promise.all([
    getUser(),
    getPosts(),
    getAnalytics(),
  ]);
  return <View user={user} posts={posts} analytics={analytics} />;
}
```

**해결 전략 2: Suspense로 독립 스트리밍**

```tsx
// BETTER: 각 영역이 독립적으로 스트리밍 — 가장 빠른 것부터 표시
async function Dashboard() {
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />   {/* 독립적으로 데이터 페칭 & 렌더 */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection />
      </Suspense>
      <Suspense fallback={<AnalyticsSkeleton />}>
        <AnalyticsSection />
      </Suspense>
    </div>
  );
}
```

**감지 방법:** Next.js의 React DevTools Profiler, 서버 로그의 타이밍, 또는 OpenTelemetry 트레이싱으로 각 fetch의 시작/종료 시점을 추적한다. Vercel의 Speed Insights나 브라우저의 Server-Timing 헤더를 활용할 수도 있다.

---

## Q3. [Microsoft/Stripe] Next.js 미들웨어와 Edge Runtime의 제약

**문제:** Next.js 미들웨어가 Edge Runtime에서 실행되는 이유와 그로 인한 제약 사항을 설명하세요. Node.js API를 사용해야 하는 인증 로직은 어떻게 처리해야 하나요?

**모범 답안:**

미들웨어는 Edge Runtime에서 실행되는데, 그 이유는 CDN 에지 노드에서 요청을 가로채 지연 시간(latency)을 최소화하기 위해서다. Edge Runtime은 V8 isolates 기반으로, 콜드 스타트가 거의 없고 전 세계 에지 노드에 분산 배포된다.

제약 사항으로는 Node.js 네이티브 모듈(`fs`, `crypto`의 일부, `child_process` 등) 사용 불가, 실행 시간 제한(보통 25초), 번들 사이즈 제한, 일부 npm 패키지 비호환 등이 있다.

Node.js API가 필요한 인증 처리 전략:

```typescript
// middleware.ts — 경량 토큰 검증만 수행
import { jwtVerify } from 'jose'; // Edge 호환 JWT 라이브러리

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('session')?.value;

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    // jose 라이브러리는 Web Crypto API 기반 — Edge 호환
    const secret = new TextEncoder().encode(process.env.JWT_SECRET);
    await jwtVerify(token, secret);
    return NextResponse.next();
  } catch {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

무거운 인증 로직(bcrypt 비밀번호 검증, DB 세션 조회 등)은 미들웨어가 아닌 Route Handler나 Server Action에서 Node.js Runtime으로 처리한다. 미들웨어는 가드(guard) 역할만 하고 실제 인증 처리는 서버로 위임하는 것이 올바른 패턴이다.

---

## Q4. [Apple/Vercel] 캐싱 아키텍처 심층 분석

**문제:** Next.js의 4가지 캐싱 레이어를 모두 설명하고, 프로덕션에서 "캐시된 오래된 데이터가 표시되는" 버그를 디버깅하는 과정을 서술하세요.

**모범 답안:**

Next.js 15 기준 캐싱 레이어:

1. **Request Memoization** (React `cache`): 동일 렌더 트리 내에서 같은 fetch 요청 중복 제거. 렌더링 단위로 유효하며, 요청 완료 후 사라진다.

2. **Data Cache**: `fetch`의 `cache`/`revalidate` 옵션으로 제어. 서버에서 영구적으로 저장되며, 시간 기반(`revalidate`) 또는 태그 기반(`revalidateTag`) 무효화가 가능하다. Next.js 15에서 기본값이 `no-store`로 변경되어 명시적으로 opt-in해야 한다.

3. **Full Route Cache**: 빌드 시 생성된 정적 라우트의 HTML + RSC Payload 캐시. `dynamic = 'force-static'`이거나 동적 함수를 사용하지 않는 경우 활성화된다.

4. **Router Cache** (클라이언트): 브라우저에서 방문한 RSC Payload를 메모리 캐시. Next.js 15에서 기본 TTL이 0으로 변경되어 페이지 간 이동 시 항상 새 데이터를 가져온다(동적 페이지 기준).

디버깅 과정:

첫째, 개발자 도구에서 응답 헤더(`x-nextjs-cache`, `cache-control`)를 확인하여 어느 레이어에서 캐시되었는지 파악한다. 둘째, Route Segment Config(`export const dynamic`, `export const revalidate`)를 확인한다. 셋째, `fetch` 호출의 캐시 옵션을 검토한다. 넷째, `revalidatePath`/`revalidateTag` 호출이 올바른 경로/태그를 대상으로 하는지 확인한다. 다섯째, CDN 캐시(Vercel Edge Network 등)에서의 캐시도 고려한다.

---

## Q5. [Netflix/Spotify] Streaming SSR과 Selective Hydration

**문제:** Next.js에서 Streaming SSR이 사용자 경험(TTFB, FCP, TTI)에 미치는 영향을 설명하고, Selective Hydration이 이를 어떻게 보완하는지 서술하세요.

**모범 답안:**

Streaming SSR은 서버가 전체 HTML을 한 번에 보내는 대신, 준비된 부분부터 청크 단위로 전송하는 방식이다(HTTP chunked transfer encoding).

성능 지표에 미치는 영향: TTFB(Time to First Byte)는 전체 데이터 페칭을 기다리지 않으므로 크게 감소한다. FCP(First Contentful Paint)는 정적 셸과 빠른 컴포넌트가 먼저 도착하므로 개선된다. TTI(Time to Interactive)는 Selective Hydration과 결합하여 개선된다.

Selective Hydration은 React 18/19의 기능으로, 모든 컴포넌트의 JavaScript가 로드되기 전에 사용자가 상호작용하려는 컴포넌트를 우선적으로 하이드레이션한다. 예를 들어 사용자가 댓글 섹션을 클릭하면, 아직 로드되지 않은 사이드바보다 댓글 컴포넌트의 하이드레이션이 우선된다.

```tsx
// Streaming + Selective Hydration 활용 예시
export default function Page() {
  return (
    <>
      {/* 즉시 전송: 정적 HTML */}
      <Header />

      {/* 1단계 스트리밍: 핵심 콘텐츠 */}
      <Suspense fallback={<ArticleSkeleton />}>
        <ArticleContent />
      </Suspense>

      {/* 2단계 스트리밍: 부가 콘텐츠 (느린 API) */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments />  {/* 사용자가 클릭하면 우선 하이드레이션 */}
      </Suspense>

      {/* 3단계 스트리밍: 비핵심 */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations />
      </Suspense>
    </>
  );
}
```

이 조합으로 사용자는 즉시 페이지 골격을 보고(FCP↑), 핵심 콘텐츠를 빠르게 읽을 수 있으며(LCP↑), 상호작용이 필요한 부분이 우선 활성화된다(TTI↑).

---

## Q6. [Uber/Airbnb] Server Actions의 보안

**문제:** Server Actions 사용 시 발생할 수 있는 보안 취약점과 대응 방법을 설명하세요.

**모범 답안:**

Server Actions는 HTTP POST 엔드포인트로 노출되므로 다음 보안 고려 사항이 있다.

**1. 입력 검증 (필수):** Server Action은 공개 API 엔드포인트와 같으므로, 클라이언트에서 어떤 데이터든 보낼 수 있다. 모든 입력을 Zod 등으로 서버 측에서 반드시 검증해야 한다.

**2. 인증/인가 확인:** 매 Server Action 호출마다 세션을 확인해야 한다. `'use server'`라고 해서 자동으로 보호되지 않는다.

```tsx
'use server';
import { auth } from '@/lib/auth';

export async function deletePost(postId: string) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  const post = await db.post.findUnique({ where: { id: postId } });
  if (post?.authorId !== session.user.id && session.user.role !== 'ADMIN') {
    throw new Error('Forbidden');
  }

  await db.post.delete({ where: { id: postId } });
}
```

**3. CSRF 보호:** Next.js는 Server Actions에 대해 자동으로 CSRF 토큰을 생성·검증한다. `serverActions.allowedOrigins` 설정으로 허용된 도메인만 호출할 수 있도록 제한한다.

**4. Rate Limiting:** Server Action도 API 엔드포인트이므로 rate limiting이 필요하다. 미들웨어나 Server Action 내부에서 구현한다.

**5. 민감 데이터 노출 방지:** Server Action의 인자와 반환값은 직렬화되어 네트워크를 통해 전달되므로, 민감 정보(비밀번호 해시, 내부 ID 등)가 반환값에 포함되지 않도록 주의한다.

**6. 클로저 캡처 주의:** 서버 컴포넌트에서 Server Action을 인라인으로 정의할 때 캡처된 변수에 민감 정보가 있으면 클라이언트로 직렬화될 수 있다. 민감 데이터는 Action 내부에서 직접 조회해야 한다.

---

## Q7. [Google/Meta] 대규모 Next.js 애플리케이션의 모노레포 설계

**문제:** 100명 이상의 개발자가 참여하는 대규모 Next.js 프로젝트의 모노레포 아키텍처를 설계하세요. 빌드 성능, 코드 공유, 팀 간 독립성을 고려해야 합니다.

**모범 답안:**

```
monorepo/
├── apps/
│   ├── web/                    # 메인 웹앱 (Next.js)
│   ├── admin/                  # 관리자 대시보드 (Next.js)
│   ├── mobile-web/             # 모바일 최적화 웹 (Next.js)
│   └── docs/                   # 문서 사이트
├── packages/
│   ├── ui/                     # 공유 디자인 시스템
│   │   ├── src/components/
│   │   ├── src/hooks/
│   │   └── package.json
│   ├── config-eslint/          # ESLint 설정
│   ├── config-typescript/      # TypeScript 설정
│   ├── database/               # Prisma 스키마 & 클라이언트
│   ├── auth/                   # 인증 라이브러리
│   ├── analytics/              # 분석 유틸리티
│   └── api-client/             # 타입 안전 API 클라이언트
├── tooling/
│   ├── scripts/                # 빌드/배포 스크립트
│   └── generators/             # 코드 생성기 (plop 등)
├── turbo.json                  # Turborepo 설정
├── pnpm-workspace.yaml
└── package.json
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

핵심 전략: Turborepo의 원격 캐싱으로 변경된 패키지만 재빌드한다. 팀별로 `apps/`를 독립 배포하며, `packages/`는 공유 라이브러리로 관리한다. CI/CD에서 `turbo --filter=[HEAD^1]`로 영향받는 앱만 빌드·배포한다. CODEOWNERS 파일로 패키지별 담당 팀을 지정한다.

---

## Q8. [Amazon/Cloudflare] Edge vs Node.js Runtime 선택 기준

**문제:** Route Handler에서 Edge Runtime과 Node.js Runtime을 각각 언제 선택해야 하는지, 구체적인 사용 사례와 함께 설명하세요.

**모범 답안:**

```tsx
// Edge Runtime 적합 사례
export const runtime = 'edge';
```

Edge Runtime이 적합한 경우: 지리적 위치 기반 응답(가장 가까운 에지 노드에서 처리), A/B 테스트 분기, 간단한 인증 토큰 검증, 리다이렉트/리라이트 로직, 경량 API(KV 스토어 조회 등). 콜드 스타트가 거의 없어 지연 시간에 민감한 API에 적합하다.

```tsx
// Node.js Runtime 적합 사례
export const runtime = 'nodejs';
```

Node.js Runtime이 적합한 경우: DB 연결(커넥션 풀링), 파일 시스템 접근, 무거운 연산(이미지 처리, PDF 생성), 네이티브 Node.js 모듈이 필요한 경우(bcrypt, sharp 등), 긴 실행 시간이 필요한 작업, WebSocket 연결.

판단 기준: 실행 시간 10ms 이하이고 Node.js API 불필요하면 Edge, 그 외에는 Node.js가 안전하다. Edge에서의 최대 실행 시간(보통 25초)과 번들 크기 제한(보통 4MB)도 고려해야 한다.

---

## Q9. [Meta/Vercel] React Compiler와 Next.js 통합

**문제:** React Compiler가 Next.js 개발에 미치는 영향을 설명하세요. 수동 메모이제이션(`useMemo`, `useCallback`, `React.memo`)은 더 이상 필요 없나요?

**모범 답안:**

React Compiler(이전 React Forget)는 빌드 타임에 컴포넌트를 분석하여 자동으로 메모이제이션 코드를 삽입하는 컴파일러다. `useMemo`, `useCallback`, `React.memo`가 수행하던 작업을 컴파일러가 자동으로 처리한다.

Next.js에서 `experimental.reactCompiler: true`로 활성화하면 모든 컴포넌트에 자동 적용된다.

수동 메모이제이션이 여전히 필요한 경우도 있다. 컴파일러가 분석할 수 없는 패턴(동적 객체 키, eval, 외부 변경 가능 참조 등), 컴파일러 옵트아웃(`"use no memo"` 지시어), 라이브러리 코드(컴파일러가 처리하지 않는 외부 모듈) 등에서는 여전히 수동 최적화가 필요할 수 있다.

실무적으로는 새 코드에서 수동 메모이제이션을 작성하지 않아도 되지만, 기존 코드의 수동 메모이제이션을 제거할 필요는 없다(컴파일러가 중복을 감지하고 무시한다). 성능 프로파일링으로 병목을 확인한 후에만 추가적인 수동 최적화를 고려해야 한다.

---

## Q10. [Uber/Stripe] 프로덕션 장애 대응 시나리오

**문제:** Next.js 프로덕션 환경에서 특정 페이지의 응답 시간이 갑자기 10초 이상으로 증가했습니다. 원인 분석과 해결 과정을 단계별로 서술하세요.

**모범 답안:**

**1단계 — 즉시 대응:**
영향 범위를 파악한다(특정 페이지만? 전체 사이트? 특정 지역?). 모니터링 대시보드(Datadog, Vercel Analytics 등)에서 에러율, 응답 시간 분포, 서버 리소스를 확인한다.

**2단계 — 범위 좁히기:**
정적/동적 여부를 확인한다. 캐시 히트율이 급감했으면 캐시 무효화 문제, 외부 API 응답 시간이 증가했으면 외부 의존성 문제다. 서버 로그에서 해당 페이지의 데이터 페칭 시간을 확인한다.

**3단계 — 일반적 원인들:**
외부 API/DB 장애: 해당 서비스의 상태 페이지를 확인하고, 타임아웃을 적절히 설정했는지 검토한다. 워터폴 데이터 페칭: 새로 배포된 코드에서 순차 fetch가 도입되었는지 확인한다. 메모리 누수: `globalThis`에 저장된 캐시나 커넥션 풀이 한도를 초과했는지 확인한다. 캐시 무효화 폭풍: 대량의 `revalidatePath('/')`가 호출되어 전체 캐시가 무효화되었는지 확인한다.

**4단계 — 긴급 완화:**

```tsx
// 캐시 폴백 추가로 임시 대응
async function getProduct(id: string) {
  try {
    const data = await fetchWithTimeout(
      `https://api.example.com/products/${id}`,
      { timeout: 3000 } // 3초 타임아웃
    );
    return data;
  } catch {
    // 캐시된 이전 데이터 반환 (stale-while-error)
    const cached = await getCachedProduct(id);
    if (cached) return { ...cached, _stale: true };
    throw new Error('Service unavailable');
  }
}
```

**5단계 — 근본 해결:**
원인에 따라 데이터 페칭 병렬화, 타임아웃 조정, 캐싱 전략 변경, 서킷 브레이커 도입, 인프라 스케일링 등을 적용하고, 재발 방지를 위한 모니터링 알림을 설정한다.

---

# 부록: 핵심 명령어 & 도구

```bash
# 개발 서버
pnpm dev           # Turbopack 사용 (기본)
pnpm dev --port 4000

# 빌드 & 배포
pnpm build         # 프로덕션 빌드
pnpm start         # 프로덕션 서버 시작
pnpm lint          # ESLint 실행

# Prisma
npx prisma migrate dev      # 마이그레이션 생성 & 적용
npx prisma generate         # 클라이언트 생성
npx prisma studio           # DB GUI

# 번들 분석
ANALYZE=true pnpm build     # @next/bundle-analyzer

# 테스트
pnpm test                   # Jest
pnpm test:e2e               # Playwright
```

---

*이 문서는 Next.js 15 (App Router 기반)의 공식 문서, Vercel 블로그, React RFC를 참고하여 작성되었습니다. 실제 Next.js 16 릴리스 시 변경 사항은 공식 릴리스 노트(nextjs.org/blog)에서 확인하세요.*
