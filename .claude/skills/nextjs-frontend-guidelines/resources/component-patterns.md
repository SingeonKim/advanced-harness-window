# Component 패턴 - Next.js 15

## Server Components vs Client Components

### Server Components (기본값)

**사용 시점:**
- 변경되지 않는 정적 콘텐츠
- API 또는 데이터베이스에서 데이터 패칭
- 인터랙티비티 불필요
- SEO에 중요한 콘텐츠

**특징:**
- `'use client'` 지시문 없음
- async 함수 가능
- 직접 데이터 패칭
- 클라이언트에 JavaScript 전송 없음
- hooks 사용 불가 (useState, useEffect 등)
- 브라우저 API 사용 불가
- 이벤트 핸들러 사용 불가

```typescript
// Server Component ('use client' 없음)
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { api } from '@/lib/api';
import type { Artist } from '@/types/artist';

interface ArtistProfileProps {
  artistId: string;
}

export default async function ArtistProfile({ artistId }: ArtistProfileProps) {
  // 서버에서 직접 데이터 패칭
  const artist: Artist = await api.artists.getById(artistId);

  return (
    <Card>
      <CardHeader>
        <CardTitle>{artist.name}</CardTitle>
      </CardHeader>
      <CardContent>
        <p className="text-muted-foreground">{artist.bio}</p>
      </CardContent>
    </Card>
  );
}
```

### Client Components

**사용 시점:**
- 상태 관리 (useState)
- 사이드 이펙트 (useEffect)
- 이벤트 핸들러 (onClick, onChange 등)
- 브라우저 API (localStorage, window 등)
- 커스텀 hooks
- Context providers

**특징:**
- 파일 상단에 반드시 `'use client'` 필요
- 모든 React hooks 사용 가능
- 브라우저 API 사용 가능
- 클라이언트에 JavaScript 전송
- 인터랙티브 요소

```typescript
'use client';

import { useState, useCallback } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { api } from '@/lib/api';

interface CommentFormProps {
  artworkId: string;
  onSubmit?: () => void;
}

export function CommentForm({ artworkId, onSubmit }: CommentFormProps) {
  const [comment, setComment] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = useCallback(async () => {
    setLoading(true);
    try {
      await api.comments.create({ artworkId, text: comment });
      setComment('');
      onSubmit?.();
    } catch (error) {
      console.error('Failed to submit comment:', error);
    } finally {
      setLoading(false);
    }
  }, [artworkId, comment, onSubmit]);

  return (
    <div className="flex flex-col gap-4">
      <Textarea
        value={comment}
        onChange={(e) => setComment(e.target.value)}
        placeholder="댓글을 입력하세요..."
        className="min-h-[100px]"
      />
      <Button onClick={handleSubmit} disabled={loading}>
        {loading ? '제출 중...' : '제출'}
      </Button>
    </div>
  );
}
```

## Component 구조 패턴

### 권장 순서

```typescript
'use client'; // 필요한 경우에만

// 1. Imports
import { useState, useCallback } from 'react';
import { Button } from '@/components/ui/button';
import { Card, CardContent } from '@/components/ui/card';
import { api } from '@/lib/api';
import { cn } from '@/lib/utils';
import type { User } from '@/types/user';

// 2. 타입/인터페이스
interface MyComponentProps {
  userId: string;
  className?: string;
  onUpdate?: (user: User) => void;
}

// 3. 컴포넌트 함수
export function MyComponent({ userId, className, onUpdate }: MyComponentProps) {
  // 4. 상태
  const [data, setData] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);

  // 5. Hooks (useCallback, useMemo, useEffect, 커스텀 hooks)
  const fetchUser = useCallback(async () => {
    setLoading(true);
    try {
      const user = await api.users.getById(userId);
      setData(user);
      onUpdate?.(user);
    } finally {
      setLoading(false);
    }
  }, [userId, onUpdate]);

  // 6. 렌더링
  return (
    <Card className={cn("p-4", className)}>
      <CardContent>
        <Button onClick={fetchUser} disabled={loading}>
          {loading ? '로딩 중...' : '사용자 불러오기'}
        </Button>
        {data && <p className="mt-4 text-muted-foreground">{data.name}</p>}
      </CardContent>
    </Card>
  );
}
```

## Server와 Client Components 혼합

### 패턴: Client 자식이 있는 Server Component

```typescript
// app/artists/page.tsx (Server Component)
import { ArtistList } from '@/components/artist/ArtistList'; // Server
import { ArtistFilters } from '@/components/artist/ArtistFilters'; // Client
import { api } from '@/lib/api';

export default async function ArtistsPage() {
  // 서버에서 데이터 패칭
  const artists = await api.artists.getAll();

  return (
    <div className="container py-8">
      {/* 인터랙티브 필터를 위한 Client Component */}
      <ArtistFilters />

      {/* 정적 목록을 위한 Server Component */}
      <ArtistList artists={artists} />
    </div>
  );
}
```

### 패턴: Server에서 Client로 데이터 전달

```typescript
// Server Component
import { ClientComponent } from './ClientComponent';

export default async function ServerPage() {
  const data = await fetch('...');

  // props를 통해 Client Component에 데이터 전달
  return <ClientComponent initialData={data} />;
}

// ClientComponent.tsx
'use client';

interface ClientComponentProps {
  initialData: DataType;
}

export function ClientComponent({ initialData }: ClientComponentProps) {
  const [data, setData] = useState(initialData);
  // ... 클라이언트 사이드 로직
}
```

## 성능 패턴

### Dynamic Imports

무거운 Client Components에는 dynamic imports 사용:

```typescript
import dynamic from 'next/dynamic';
import { Skeleton } from '@/components/ui/skeleton';

// 로딩 상태와 함께 지연 로딩
const HeavyChart = dynamic(
  () => import('@/components/charts/HeavyChart'),
  {
    loading: () => <Skeleton className="h-[400px] w-full" />,
    ssr: false, // 컴포넌트가 브라우저 API를 사용하면 SSR 비활성화
  }
);

export function Dashboard() {
  return (
    <div className="space-y-4">
      <HeavyChart data={...} />
    </div>
  );
}
```

### 비용이 큰 컴포넌트에 React.memo 사용

```typescript
'use client';

import { memo } from 'react';
import { Card, CardContent } from '@/components/ui/card';

interface ExpensiveListProps {
  items: Item[];
}

export const ExpensiveList = memo(function ExpensiveList({ items }: ExpensiveListProps) {
  // 비용이 큰 렌더링 로직
  return (
    <div className="grid gap-4">
      {items.map(item => (
        <Card key={item.id}>
          <CardContent className="pt-6">
            {item.name}
          </CardContent>
        </Card>
      ))}
    </div>
  );
});
```

## 공통 패턴

### 이벤트 핸들러에 useCallback

자식 컴포넌트에 전달되는 이벤트 핸들러는 항상 `useCallback` 사용:

```typescript
'use client';

import { useCallback, useState } from 'react';
import { Button } from '@/components/ui/button';

export function Parent() {
  const [count, setCount] = useState(0);

  // useCallback으로 감쌈
  const handleClick = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  return <Child onClick={handleClick} />;
}
```

### shadcn/ui로 폼 처리

```typescript
'use client';

import { useState, useCallback, FormEvent } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = useCallback(async (e: FormEvent) => {
    e.preventDefault();
    setLoading(true);
    try {
      // 제출 처리
    } finally {
      setLoading(false);
    }
  }, [email, password]);

  return (
    <Card className="w-full max-w-md">
      <CardHeader>
        <CardTitle>로그인</CardTitle>
      </CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          <div className="space-y-2">
            <Label htmlFor="email">이메일</Label>
            <Input
              id="email"
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              placeholder="이메일을 입력하세요"
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="password">비밀번호</Label>
            <Input
              id="password"
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              placeholder="비밀번호를 입력하세요"
            />
          </div>
          <Button type="submit" className="w-full" disabled={loading}>
            {loading ? '로그인 중...' : '로그인'}
          </Button>
        </form>
      </CardContent>
    </Card>
  );
}
```

## TypeScript 패턴

### JSDoc이 있는 컴포넌트 Props

```typescript
interface ButtonProps {
  /** 버튼에 표시할 텍스트 */
  label: string;
  /** 선택적 클릭 핸들러 */
  onClick?: () => void;
  /** 버튼이 로딩 상태인지 여부 */
  loading?: boolean;
  /** 추가 CSS 클래스 */
  className?: string;
}

export function CustomButton({ label, onClick, loading = false, className }: ButtonProps) {
  return (
    <Button onClick={onClick} disabled={loading} className={className}>
      {loading ? '로딩 중...' : label}
    </Button>
  );
}
```

### 타입 추출

```typescript
// types/artist.ts
export interface Artist {
  id: string;
  name: string;
  bio: string;
  artworks: Artwork[];
}

// 컴포넌트
import type { Artist } from '@/types/artist';

interface ArtistCardProps {
  artist: Artist;
  className?: string;
}
```

### className Props에 cn() 사용

```typescript
import { cn } from '@/lib/utils';

interface ComponentProps {
  className?: string;
  children: React.ReactNode;
}

export function Component({ className, children }: ComponentProps) {
  return (
    <div className={cn(
      "flex items-center gap-4 p-4 rounded-lg border",
      className
    )}>
      {children}
    </div>
  );
}
```

## 모범 사례

1. **Server Components 기본값**: 인터랙티비티가 필요할 때만 Client Components 사용
2. **Client Components 작게 유지**: 비인터랙티브 부분은 Server Components로 추출
3. **데이터를 아래로 전달**: Server Components에서 패칭하여 Client Components에 props로 전달
4. **Callbacks 사용**: 리렌더링 방지를 위해 `useCallback`으로 이벤트 핸들러 감싸기
5. **모든 것 타입 지정**: props, 상태, 반환 값에 명시적 타입 사용
6. **Named Exports**: 컴포넌트에 named exports 사용 (검색과 리팩토링에 용이)
7. **Async Server Components**: Server Components에서 async/await 활용
8. **에러 경계**: 우아한 실패를 위해 error.tsx로 Client Components 감싸기
9. **항상 className 허용**: 커스터마이즈를 위해 컴포넌트가 `className` prop 허용
10. **클래스 병합에 cn() 사용**: 클래스 조합 시 항상 `cn()` 사용
