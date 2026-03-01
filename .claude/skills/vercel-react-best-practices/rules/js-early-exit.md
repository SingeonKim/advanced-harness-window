---
title: Early Return from Functions
impact: LOW-MEDIUM
impactDescription: avoids unnecessary computation
tags: javascript, functions, optimization, early-return
---

## 함수에서 조기 반환

결과가 결정되면 즉시 반환하여 불필요한 처리를 건너뜁니다.

**잘못된 방법 (답을 찾은 후에도 모든 항목 처리):**

```typescript
function validateUsers(users: User[]) {
  let hasError = false
  let errorMessage = ''

  for (const user of users) {
    if (!user.email) {
      hasError = true
      errorMessage = 'Email required'
    }
    if (!user.name) {
      hasError = true
      errorMessage = 'Name required'
    }
    // 에러 발견 후에도 모든 사용자 계속 확인
  }

  return hasError ? { valid: false, error: errorMessage } : { valid: true }
}
```

**올바른 방법 (첫 번째 에러에서 즉시 반환):**

```typescript
function validateUsers(users: User[]) {
  for (const user of users) {
    if (!user.email) {
      return { valid: false, error: 'Email required' }
    }
    if (!user.name) {
      return { valid: false, error: 'Name required' }
    }
  }

  return { valid: true }
}
```
