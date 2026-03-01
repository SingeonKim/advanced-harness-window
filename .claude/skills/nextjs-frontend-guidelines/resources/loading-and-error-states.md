# 로딩 및 에러 상태 - Next.js 15

## 로딩 상태

### 라우트 레벨 로딩

```typescript
// app/artists/loading.tsx
export default function Loading() {
  return (
    <div className="flex items-center justify-center p-8">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500" />
    </div>
  );
}
```

### Suspense 경계

```typescript
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <h1>아티스트</h1>
      <Suspense fallback={<LoadingSpinner />}>
        <ArtistList />
      </Suspense>
    </div>
  );
}
```

### 클라이언트 사이드 로딩

```typescript
'use client';

import { useState } from 'react';
import { CircularProgress, Button } from '@mui/material';

export function Component() {
  const [loading, setLoading] = useState(false);

  const handleClick = async () => {
    setLoading(true);
    try {
      await api.doSomething();
    } finally {
      setLoading(false);
    }
  };

  return (
    <Button onClick={handleClick} disabled={loading}>
      {loading ? <CircularProgress size={20} /> : '클릭'}
    </Button>
  );
}
```

## 에러 처리

### 라우트 레벨 에러 경계

```typescript
// app/artists/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="p-8 text-center">
      <h2 className="text-xl font-bold text-red-600 mb-4">
        오류가 발생했습니다!
      </h2>
      <p className="text-gray-600 mb-4">{error.message}</p>
      <button
        onClick={reset}
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        다시 시도
      </button>
    </div>
  );
}
```

### 클라이언트 사이드 에러 처리

```typescript
'use client';

import { useState } from 'react';

export function Component() {
  const [error, setError] = useState<string | null>(null);

  const handleAction = async () => {
    try {
      await api.doSomething();
    } catch (err) {
      setError('오류가 발생했습니다');
      console.error(err);
    }
  };

  return (
    <div>
      {error && <div className="text-red-600">{error}</div>}
      <button onClick={handleAction}>액션</button>
    </div>
  );
}
```

### Not Found (찾을 수 없음)

```typescript
// app/artists/[id]/not-found.tsx
export default function NotFound() {
  return (
    <div className="text-center p-8">
      <h2 className="text-2xl font-bold">아티스트를 찾을 수 없습니다</h2>
      <p className="text-gray-600 mt-2">
        찾으시는 아티스트가 존재하지 않습니다.
      </p>
    </div>
  );
}

// 페이지 컴포넌트에서 사용
import { notFound } from 'next/navigation';

export default async function ArtistPage({ params }) {
  const artist = await api.artists.getById(params.id);

  if (!artist) {
    notFound();
  }

  return <ArtistProfile artist={artist} />;
}
```
