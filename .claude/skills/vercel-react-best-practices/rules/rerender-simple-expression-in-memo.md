---
title: Do not wrap a simple expression with a primitive result type in useMemo
impact: LOW-MEDIUM
impactDescription: wasted computation on every render
tags: rerender, useMemo, optimization
---

## 원시형 결과를 갖는 단순 표현식을 useMemo로 감싸지 않기

표현식이 단순하고(논리 또는 산술 연산자가 적음) 원시형 결과 타입(boolean, number, string)을 가질 때 `useMemo`로 감싸지 않습니다.
`useMemo`를 호출하고 훅 의존성을 비교하는 것이 표현식 자체보다 더 많은 리소스를 소비할 수 있습니다.

**잘못된 방법:**

```tsx
function Header({ user, notifications }: Props) {
  const isLoading = useMemo(() => {
    return user.isLoading || notifications.isLoading
  }, [user.isLoading, notifications.isLoading])

  if (isLoading) return <Skeleton />
  // 마크업 반환
}
```

**올바른 방법:**

```tsx
function Header({ user, notifications }: Props) {
  const isLoading = user.isLoading || notifications.isLoading

  if (isLoading) return <Skeleton />
  // 마크업 반환
}
```
