---
title: Avoid Duplicate Serialization in RSC Props
impact: LOW
impactDescription: reduces network payload by avoiding duplicate serialization
tags: server, rsc, serialization, props, client-components
---

## RSC Props에서 중복 직렬화 지양

**영향도: LOW (중복 직렬화를 방지하여 네트워크 페이로드 감소)**

RSC→클라이언트 직렬화는 값이 아닌 객체 참조로 중복을 제거합니다. 동일한 참조 = 한 번 직렬화; 새 참조 = 다시 직렬화. 변환(`.toSorted()`, `.filter()`, `.map()`)은 서버가 아닌 클라이언트에서 수행합니다.

**잘못된 방법 (배열 중복):**

```tsx
// RSC: 6개 문자열 전송 (2개 배열 × 3개 항목)
<ClientList usernames={usernames} usernamesOrdered={usernames.toSorted()} />
```

**올바른 방법 (3개 문자열 전송):**

```tsx
// RSC: 한 번만 전송
<ClientList usernames={usernames} />

// 클라이언트: 거기서 변환
'use client'
const sorted = useMemo(() => [...usernames].sort(), [usernames])
```

**중첩된 중복 제거 동작:**

중복 제거는 재귀적으로 작동합니다. 영향은 데이터 유형에 따라 다릅니다:

- `string[]`, `number[]`, `boolean[]`: **HIGH 영향** - 배열 + 모든 원시형이 완전히 중복됨
- `object[]`: **LOW 영향** - 배열은 중복되지만, 중첩된 객체는 참조로 중복 제거됨

```tsx
// string[] - 모든 것이 중복됨
usernames={['a','b']} sorted={usernames.toSorted()} // 4개 문자열 전송

// object[] - 배열 구조만 중복됨
users={[{id:1},{id:2}]} sorted={users.toSorted()} // 2개 배열 + 2개 고유 객체 (4개가 아님)
```

**중복 제거를 깨뜨리는 작업 (새 참조 생성):**

- 배열: `.toSorted()`, `.filter()`, `.map()`, `.slice()`, `[...arr]`
- 객체: `{...obj}`, `Object.assign()`, `structuredClone()`, `JSON.parse(JSON.stringify())`

**더 많은 예시:**

```tsx
// 잘못된 방법
<C users={users} active={users.filter(u => u.active)} />
<C product={product} productName={product.name} />

// 올바른 방법
<C users={users} />
<C product={product} />
// 필터링/구조 분해는 클라이언트에서 수행
```

**예외:** 변환이 비용이 많이 들거나 클라이언트가 원본을 필요로 하지 않을 때는 파생 데이터를 전달합니다.
