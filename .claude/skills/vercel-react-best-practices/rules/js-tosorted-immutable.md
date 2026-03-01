---
title: Use toSorted() Instead of sort() for Immutability
impact: MEDIUM-HIGH
impactDescription: prevents mutation bugs in React state
tags: javascript, arrays, immutability, react, state, mutation
---

## 불변성을 위해 sort() 대신 toSorted() 사용

`.sort()`는 배열을 직접 변이시키므로 React state와 props에서 버그를 유발할 수 있습니다. `.toSorted()`를 사용하면 변이 없이 새 정렬 배열을 생성합니다.

**잘못된 방법 (원본 배열 변이):**

```typescript
function UserList({ users }: { users: User[] }) {
  // users prop 배열을 변이시킴!
  const sorted = useMemo(
    () => users.sort((a, b) => a.name.localeCompare(b.name)),
    [users]
  )
  return <div>{sorted.map(renderUser)}</div>
}
```

**올바른 방법 (새 배열 생성):**

```typescript
function UserList({ users }: { users: User[] }) {
  // 새 정렬 배열 생성, 원본은 변경 없음
  const sorted = useMemo(
    () => users.toSorted((a, b) => a.name.localeCompare(b.name)),
    [users]
  )
  return <div>{sorted.map(renderUser)}</div>
}
```

**React에서 중요한 이유:**

1. Props/state 변이는 React의 불변성 모델을 위반함 - React는 props와 state를 읽기 전용으로 처리할 것을 기대함
2. 클로저 캐싱 버그 유발 - 클로저(콜백, effect) 내부에서 배열을 변이시키면 예상치 못한 동작이 발생할 수 있음

**브라우저 지원 (구형 브라우저 폴백):**

`.toSorted()`는 모든 모던 브라우저에서 지원됩니다 (Chrome 110+, Safari 16+, Firefox 115+, Node.js 20+). 구형 환경에서는 스프레드 연산자를 사용하세요:

```typescript
// 구형 브라우저를 위한 폴백
const sorted = [...items].sort((a, b) => a.value - b.value)
```

**다른 불변 배열 메서드:**

- `.toSorted()` - 불변 정렬
- `.toReversed()` - 불변 역순
- `.toSpliced()` - 불변 splice
- `.with()` - 불변 요소 교체
