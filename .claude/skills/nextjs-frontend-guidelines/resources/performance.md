# 성능 최적화 - Next.js 15

## Server Components (JavaScript 제로)

더 나은 성능을 위해 기본적으로 Server Components를 사용하세요:

```typescript
// 클라이언트에 JavaScript 전송 없음
export default async function Page() {
  const data = await fetchData();
  return <StaticContent data={data} />;
}
```

## Dynamic Imports

```typescript
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <div>로딩 중...</div>,
  ssr: false, // 필요한 경우 SSR 비활성화
});

export function Page() {
  return <HeavyComponent />;
}
```

## 이미지 최적화

```typescript
import Image from 'next/image';

export function Component() {
  return (
    <Image
      src="/path/to/image.jpg"
      alt="설명"
      width={500}
      height={300}
      priority // 뷰포트 상단 이미지에 사용
      quality={90}
    />
  );
}
```

## React 최적화 Hooks

### useMemo

```typescript
'use client';

import { useMemo } from 'react';

export function Component({ items }) {
  const sortedItems = useMemo(() => {
    return items.sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);

  return <List items={sortedItems} />;
}
```

### useCallback

```typescript
'use client';

import { useCallback } from 'react';

export function Component() {
  const handleClick = useCallback(() => {
    // 핸들러 로직
  }, []);

  return <ChildComponent onClick={handleClick} />;
}
```

### React.memo

```typescript
'use client';

import { memo } from 'react';

export const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  // 비용이 큰 렌더링
  return <div>{data}</div>;
});
```

## 모범 사례

1. **Server Components 우선**: 클라이언트에 전송되는 JavaScript 최소화
2. **Dynamic Imports**: 무거운 컴포넌트에 사용
3. **이미지 최적화**: 항상 next/image 사용
4. **메모이제이션**: 적절하게 useMemo/useCallback 사용
5. **코드 분할**: Next.js 라우트로 자동 처리
6. **캐싱**: 데이터 패칭에 revalidate 사용
7. **클라이언트 JS 최소화**: 필요한 경우에만 'use client' 사용
