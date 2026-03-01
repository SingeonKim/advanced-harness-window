# 데이터 패칭 - Next.js 15

## 개요

Next.js 15는 데이터를 패칭하는 다양한 방법을 제공합니다. 용도에 따라 선택하세요:

1. **Server Components** (권장): async 컴포넌트에서 직접 데이터 패칭
2. **클라이언트 사이드 패칭**: 동적 데이터를 위한 Client Components에서 사용
3. **API Routes**: 외부 API 호출 또는 복잡한 서버 로직에 사용
4. **Server Actions**: 변이 및 폼 제출에 사용

---

## Server Component 데이터 패칭 (권장)

### 기본 패턴

```typescript
// app/artists/page.tsx
import { api } from '@/lib/api';
import type { Artist } from '@/types/artist';

export default async function ArtistsPage() {
  // 컴포넌트에서 직접 패칭
  const artists: Artist[] = await api.artists.getAll();

  return (
    <div>
      {artists.map(artist => (
        <div key={artist.id}>{artist.name}</div>
      ))}
    </div>
  );
}
```

### 에러 처리 포함

```typescript
export default async function ArtistsPage() {
  try {
    const artists = await api.artists.getAll();
    return <ArtistList artists={artists} />;
  } catch (error) {
    console.error('Failed to fetch artists:', error);
    return <ErrorDisplay message="아티스트를 불러오는데 실패했습니다" />;
  }
}
```

### 병렬 데이터 패칭

```typescript
export default async function DashboardPage() {
  // 여러 데이터 소스를 병렬로 패칭
  const [artists, artworks, exhibitions] = await Promise.all([
    api.artists.getAll(),
    api.artworks.getAll(),
    api.exhibitions.getAll(),
  ]);

  return (
    <Dashboard
      artists={artists}
      artworks={artworks}
      exhibitions={exhibitions}
    />
  );
}
```

### 캐싱 사용

```typescript
// 60초마다 재검증
export const revalidate = 60;

export default async function ArtistsPage() {
  const artists = await api.artists.getAll();
  return <ArtistList artists={artists} />;
}
```

또는 요청별로:

```typescript
export default async function ArtistsPage() {
  const artists = await fetch('https://api.example.com/artists', {
    next: { revalidate: 60 }, // 60초마다 재검증
  }).then(res => res.json());

  return <ArtistList artists={artists} />;
}
```

### 캐싱 없음 (항상 최신 데이터)

```typescript
export default async function ArtistsPage() {
  const artists = await fetch('https://api.example.com/artists', {
    cache: 'no-store', // 항상 최신 데이터 패칭
  }).then(res => res.json());

  return <ArtistList artists={artists} />;
}
```

---

## 클라이언트 사이드 데이터 패칭

### useState를 사용한 기본 패턴

```typescript
'use client';

import { useState, useEffect } from 'react';
import { api } from '@/lib/api';
import type { Artist } from '@/types/artist';

export function ArtistList() {
  const [artists, setArtists] = useState<Artist[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchArtists() {
      try {
        const data = await api.artists.getAll();
        setArtists(data);
      } catch (err) {
        setError('아티스트를 불러오는데 실패했습니다');
        console.error(err);
      } finally {
        setLoading(false);
      }
    }

    fetchArtists();
  }, []);

  if (loading) return <div>로딩 중...</div>;
  if (error) return <div>오류: {error}</div>;

  return (
    <div>
      {artists.map(artist => (
        <div key={artist.id}>{artist.name}</div>
      ))}
    </div>
  );
}
```

### 커스텀 Hook 사용

```typescript
// hooks/useArtists.ts
'use client';

import { useState, useEffect } from 'react';
import { api } from '@/lib/api';
import type { Artist } from '@/types/artist';

export function useArtists() {
  const [artists, setArtists] = useState<Artist[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    api.artists.getAll()
      .then(setArtists)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  return { artists, loading, error };
}

// 컴포넌트에서 사용
'use client';

import { useArtists } from '@/hooks/useArtists';

export function ArtistList() {
  const { artists, loading, error } = useArtists();

  if (loading) return <div>로딩 중...</div>;
  if (error) return <div>오류: {error.message}</div>;

  return <div>{/* 아티스트 렌더링 */}</div>;
}
```

---

## API Routes

### API Routes 생성

```typescript
// app/api/artists/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getAuth } from '@/lib/serverAuth';

export async function GET(request: NextRequest) {
  try {
    // 선택사항: 인증 확인
    const auth = await getAuth(request);
    if (!auth.isAuthenticated) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    // 데이터 패칭 (데이터베이스, 외부 API 등)
    const artists = await fetchArtistsFromDatabase();

    return NextResponse.json({ artists });
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

export async function POST(request: NextRequest) {
  try {
    const auth = await getAuth(request);
    if (!auth.isAuthenticated) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const body = await request.json();
    const newArtist = await createArtist(body);

    return NextResponse.json({ artist: newArtist }, { status: 201 });
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { error: '아티스트 생성에 실패했습니다' },
      { status: 500 }
    );
  }
}
```

### 동적 API Routes

```typescript
// app/api/artists/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const artistId = params.id;
  const artist = await fetchArtistById(artistId);

  if (!artist) {
    return NextResponse.json({ error: '아티스트를 찾을 수 없습니다' }, { status: 404 });
  }

  return NextResponse.json({ artist });
}

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const artistId = params.id;
  const body = await request.json();

  const updatedArtist = await updateArtist(artistId, body);

  return NextResponse.json({ artist: updatedArtist });
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const artistId = params.id;
  await deleteArtist(artistId);

  return NextResponse.json({ success: true }, { status: 204 });
}
```

---

## Server Actions

### 기본 Server Action

```typescript
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { api } from '@/lib/api';

export async function createArtist(formData: FormData) {
  const name = formData.get('name') as string;
  const bio = formData.get('bio') as string;

  try {
    const artist = await api.artists.create({ name, bio });

    // 아티스트 페이지를 재검증하여 새 데이터 표시
    revalidatePath('/artists');

    return { success: true, artist };
  } catch (error) {
    console.error('아티스트 생성 실패:', error);
    return { success: false, error: '아티스트 생성에 실패했습니다' };
  }
}
```

### 폼에서 Server Actions 사용

```typescript
'use client';

import { useCallback, useState } from 'react';
import { Box, TextField, Button } from '@mui/material';
import { createArtist } from './actions';

export function ArtistForm() {
  const [loading, setLoading] = useState(false);
  const [message, setMessage] = useState('');

  const handleSubmit = useCallback(async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setLoading(true);

    const formData = new FormData(e.currentTarget);
    const result = await createArtist(formData);

    if (result.success) {
      setMessage('아티스트가 성공적으로 생성되었습니다!');
      e.currentTarget.reset();
    } else {
      setMessage(result.error || '아티스트 생성에 실패했습니다');
    }

    setLoading(false);
  }, []);

  return (
    <Box component="form" onSubmit={handleSubmit}>
      <TextField name="name" label="이름" required fullWidth />
      <TextField name="bio" label="소개" multiline rows={4} fullWidth />
      <Button type="submit" disabled={loading}>
        {loading ? '생성 중...' : '아티스트 생성'}
      </Button>
      {message && <div>{message}</div>}
    </Box>
  );
}
```

### redirect가 있는 Server Action

```typescript
'use server';

import { redirect } from 'next/navigation';
import { api } from '@/lib/api';

export async function createArtist(formData: FormData) {
  const artist = await api.artists.create({
    name: formData.get('name') as string,
    bio: formData.get('bio') as string,
  });

  // 새 아티스트 페이지로 리다이렉트
  redirect(`/artists/${artist.id}`);
}
```

---

## 데이터 패칭 패턴

프로젝트는 **두 가지 패턴**을 지원합니다:

### 패턴 1: 중앙화된 API 클라이언트 (복잡한 요청에 권장)

**`@/lib/api` 사용 시:**
- 타입 안전한 API 호출
- 인증이 있는 복잡한 요청
- 중앙화된 에러 처리
- 일관된 요청 포맷

```typescript
// Server Component
import { api } from '@/lib/api';

export default async function Page() {
  const artists = await api.artists.getAll();
  return <ArtistList artists={artists} />;
}

// Client Component
'use client';

import { useEffect, useState } from 'react';
import { api } from '@/lib/api';

export function ClientComponent() {
  const [data, setData] = useState(null);

  useEffect(() => {
    api.artists.getAll().then(setData);
  }, []);

  return <div>{/* 렌더링 */}</div>;
}
```

### 패턴 2: 직접 Fetch (간단한 공개 엔드포인트에 적합)

**`little_boy_server_endpoint`와 함께 직접 `fetch()` 사용 시:**
- 간단한 GET 요청
- 인증 없는 공개 엔드포인트
- 빠른 프로토타이핑

```typescript
import { little_boy_server_endpoint } from '@/const/endpoint';

export default async function Page() {
  const response = await fetch(`${little_boy_server_endpoint}/api/v1/artists/tags/`);
  const tags = await response.json();
  return <TagList tags={tags} />;
}
```

**언제 무엇을 사용할지:**
- API 클라이언트: 인증된 요청, 복잡한 작업, 타입 안전성 필요 시
- 직접 Fetch: 공개 엔드포인트, 간단한 GET 요청, 빠른 프로토타이핑

---

## Server 인증과 함께 데이터 패칭

서버에서 인증된 요청을 위해:

```typescript
// app/protected/page.tsx
import { cookies } from 'next/headers';
import { getAuth } from '@/lib/serverAuth';
import { redirect } from 'next/navigation';

export default async function ProtectedPage() {
  const cookieStore = await cookies();
  const auth = await getAuth({ cookies: cookieStore });

  if (!auth.isAuthenticated) {
    redirect('/login');
  }

  // 사용자 인증됨, 보호된 데이터 패칭
  const userData = await api.users.getProfile(auth.userId);

  return <UserProfile user={userData} />;
}
```

---

## 모범 사례

1. **Server Components 우선**: 가능하면 Server Components에서 데이터 패칭
2. **API 클라이언트 사용**: 항상 중앙화된 API 클라이언트(`@/lib/api`) 사용
3. **에러 처리**: 항상 데이터 패칭을 try/catch로 감싸기
4. **로딩 상태**: 데이터를 패칭하는 동안 로딩 UI 표시
5. **병렬 패칭**: 독립적인 데이터 소스에 `Promise.all()` 사용
6. **캐싱**: 캐시 가능한 데이터에 `revalidate` 사용
7. **타입 안전성**: 항상 TypeScript interfaces로 데이터 타입 지정
8. **Server Actions**: 변이 및 폼 제출에 사용
9. **인증**: 보호된 라우트에 `src/lib/serverAuth`의 `getAuth()` 사용
10. **재검증**: 변이 후 데이터 갱신에 `revalidatePath()` 사용
