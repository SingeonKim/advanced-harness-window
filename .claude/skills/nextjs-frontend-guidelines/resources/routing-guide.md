# 라우팅 가이드 - Next.js 15 App Router

## 파일 기반 라우팅

Next.js 15는 `app/` 디렉토리에서 파일 기반 라우팅을 사용합니다.

### 기본 라우트

```
app/
  page.tsx              → /
  about/
    page.tsx            → /about
  artists/
    page.tsx            → /artists
    [id]/
      page.tsx          → /artists/:id
```

### 동적 라우트

```typescript
// app/artists/[id]/page.tsx
interface PageProps {
  params: { id: string };
}

export default async function ArtistPage({ params }: PageProps) {
  const artist = await api.artists.getById(params.id);
  return <ArtistProfile artist={artist} />;
}
```

### Catch-All 라우트

```typescript
// app/docs/[...slug]/page.tsx
interface PageProps {
  params: { slug: string[] };
}

export default function DocsPage({ params }: PageProps) {
  // /docs/a/b/c → params.slug = ['a', 'b', 'c']
  return <div>Docs: {params.slug.join('/')}</div>;
}
```

## 네비게이션

### 클라이언트 사이드 (useRouter)

```typescript
'use client';

import { useRouter } from 'next/navigation';

export function Component() {
  const router = useRouter();

  return (
    <button onClick={() => router.push('/artists')}>
      아티스트 목록으로
    </button>
  );
}
```

### 서버 사이드 (redirect)

```typescript
import { redirect } from 'next/navigation';

export default async function Page() {
  const auth = await getAuth();

  if (!auth.isAuthenticated) {
    redirect('/login');
  }

  return <div>보호된 콘텐츠</div>;
}
```

### Link 컴포넌트

```typescript
import Link from 'next/link';

export function Nav() {
  return (
    <nav>
      <Link href="/">홈</Link>
      <Link href="/artists">아티스트</Link>
      <Link href="/about">소개</Link>
    </nav>
  );
}
```

## 레이아웃

```typescript
// app/layout.tsx (루트 레이아웃)
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx (중첩 레이아웃)
export default function DashboardLayout({ children }) {
  return (
    <div>
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}
```

## 라우트 그룹

```
app/
  (marketing)/
    page.tsx            → /
    about/
      page.tsx          → /about
  (app)/
    dashboard/
      page.tsx          → /dashboard
```

라우트 그룹은 URL 구조에 영향을 주지 않으면서 파일을 구성할 때 사용합니다.
