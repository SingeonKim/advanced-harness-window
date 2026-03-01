# TypeScript 표준 - Next.js 15

## Strict 모드

프로젝트에 TypeScript strict 모드를 활성화하세요:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

## 컴포넌트 Props

```typescript
interface ComponentProps {
  /** 사용자 이름 */
  name: string;
  /** 선택적 콜백 함수 */
  onUpdate?: (value: string) => void;
  /** 로딩 상태 */
  loading?: boolean;
}

export function Component({ name, onUpdate, loading = false }: ComponentProps) {
  return <div>{name}</div>;
}
```

## 타입 Imports

```typescript
// 타입 imports 사용 (권장)
import type { User } from '@/types/user';
import type { SxProps, Theme } from '@mui/material';

// 값과 혼용 금지
import { User } from '@/types/user';
```

## Next.js 타입

```typescript
import type { Metadata } from 'next';
import type { NextRequest } from 'next/server';

export const metadata: Metadata = {
  title: '페이지 제목',
};

export async function GET(request: NextRequest) {
  // ...
}
```

## 페이지 Props

```typescript
interface PageProps {
  params: { id: string };
  searchParams: { [key: string]: string | string[] | undefined };
}

export default async function Page({ params, searchParams }: PageProps) {
  return <div>{params.id}</div>;
}
```

## API 응답 타입

```typescript
// types/api.ts
export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

// 사용 예시
const response: ApiResponse<User> = await api.users.get(id);
```

## 이벤트 핸들러

```typescript
'use client';

import type { FormEvent, ChangeEvent } from 'react';

export function Form() {
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
  };

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

## 모범 사례

1. **명시적 타입**: 항상 함수 매개변수와 반환 값에 타입 지정
2. **타입 Imports**: 타입에는 `import type` 사용
3. **Interfaces**: 객체 형태에는 인터페이스 사용
4. **any 금지**: 타입을 알 수 없는 경우 `unknown` 사용
5. **JSDoc**: 복잡한 타입과 props 문서화
6. **유틸리티 타입**: 적절히 Partial, Pick, Omit 활용
