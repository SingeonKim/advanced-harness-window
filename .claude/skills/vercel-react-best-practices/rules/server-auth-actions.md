---
title: Authenticate Server Actions Like API Routes
impact: CRITICAL
impactDescription: prevents unauthorized access to server mutations
tags: server, server-actions, authentication, security, authorization
---

## API 라우트처럼 Server Actions 인증하기

**영향도: CRITICAL (서버 뮤테이션에 대한 무단 접근 방지)**

`"use server"`가 있는 함수인 Server Actions는 API 라우트와 마찬가지로 공개 엔드포인트로 노출됩니다. Server Actions는 직접 호출될 수 있으므로 미들웨어, 레이아웃 가드, 페이지 수준 검사에만 의존하지 말고 각 Server Action **내부**에서 항상 인증과 권한 부여를 확인합니다.

Next.js 문서에 명시되어 있습니다: "Server Actions을 공개 API 엔드포인트와 동일한 보안 고려 사항으로 처리하고, 사용자가 뮤테이션을 수행할 수 있는지 확인하세요."

**잘못된 방법 (인증 검사 없음):**

```typescript
'use server'

export async function deleteUser(userId: string) {
  // 누구든 호출 가능! 인증 검사 없음
  await db.user.delete({ where: { id: userId } })
  return { success: true }
}
```

**올바른 방법 (액션 내부에서 인증):**

```typescript
'use server'

import { verifySession } from '@/lib/auth'
import { unauthorized } from '@/lib/errors'

export async function deleteUser(userId: string) {
  // 항상 액션 내부에서 인증 확인
  const session = await verifySession()

  if (!session) {
    throw unauthorized('Must be logged in')
  }

  // 권한 부여도 확인
  if (session.user.role !== 'admin' && session.user.id !== userId) {
    throw unauthorized('Cannot delete other users')
  }

  await db.user.delete({ where: { id: userId } })
  return { success: true }
}
```

**입력 검증 포함:**

```typescript
'use server'

import { verifySession } from '@/lib/auth'
import { z } from 'zod'

const updateProfileSchema = z.object({
  userId: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email()
})

export async function updateProfile(data: unknown) {
  // 먼저 입력 검증
  const validated = updateProfileSchema.parse(data)

  // 그 다음 인증
  const session = await verifySession()
  if (!session) {
    throw new Error('Unauthorized')
  }

  // 그 다음 권한 부여
  if (session.user.id !== validated.userId) {
    throw new Error('Can only update own profile')
  }

  // 마지막으로 뮤테이션 수행
  await db.user.update({
    where: { id: validated.userId },
    data: {
      name: validated.name,
      email: validated.email
    }
  })

  return { success: true }
}
```

참조: [https://nextjs.org/docs/app/guides/authentication](https://nextjs.org/docs/app/guides/authentication)
