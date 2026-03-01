---
title: Cache Property Access in Loops
impact: LOW-MEDIUM
impactDescription: reduces lookups
tags: javascript, loops, optimization, caching
---

## 루프에서 속성 접근 캐시

핫 패스에서 객체 속성 조회를 캐시합니다.

**잘못된 방법 (3번 조회 × N번 반복):**

```typescript
for (let i = 0; i < arr.length; i++) {
  process(obj.config.settings.value)
}
```

**올바른 방법 (총 1번 조회):**

```typescript
const value = obj.config.settings.value
const len = arr.length
for (let i = 0; i < len; i++) {
  process(value)
}
```
