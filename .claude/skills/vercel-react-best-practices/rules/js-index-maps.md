---
title: Build Index Maps for Repeated Lookups
impact: LOW-MEDIUM
impactDescription: 1M ops to 2K ops
tags: javascript, map, indexing, optimization, performance
---

## 반복 조회를 위해 인덱스 Map 구축

같은 키로 여러 번 `.find()`를 호출할 경우 Map을 사용합니다.

**잘못된 방법 (조회당 O(n)):**

```typescript
function processOrders(orders: Order[], users: User[]) {
  return orders.map(order => ({
    ...order,
    user: users.find(u => u.id === order.userId)
  }))
}
```

**올바른 방법 (조회당 O(1)):**

```typescript
function processOrders(orders: Order[], users: User[]) {
  const userById = new Map(users.map(u => [u.id, u]))

  return orders.map(order => ({
    ...order,
    user: userById.get(order.userId)
  }))
}
```

Map을 한 번 구축(O(n))하면 이후 모든 조회가 O(1)입니다.
1000개의 주문 × 1000명의 사용자: 1M 작업 → 2K 작업.
