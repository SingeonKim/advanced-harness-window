---
title: Use Loop for Min/Max Instead of Sort
impact: LOW
impactDescription: O(n) instead of O(n log n)
tags: javascript, arrays, performance, sorting, algorithms
---

## 정렬 대신 루프로 최솟값/최댓값 구하기

가장 작거나 큰 요소를 찾는 데는 배열을 한 번만 순회하면 됩니다. 정렬은 낭비적이고 느립니다.

**잘못된 방법 (O(n log n) - 정렬로 최신 항목 찾기):**

```typescript
interface Project {
  id: string
  name: string
  updatedAt: number
}

function getLatestProject(projects: Project[]) {
  const sorted = [...projects].sort((a, b) => b.updatedAt - a.updatedAt)
  return sorted[0]
}
```

최댓값을 찾기 위해 전체 배열을 정렬합니다.

**잘못된 방법 (O(n log n) - 가장 오래된 것과 최신 것 모두 정렬):**

```typescript
function getOldestAndNewest(projects: Project[]) {
  const sorted = [...projects].sort((a, b) => a.updatedAt - b.updatedAt)
  return { oldest: sorted[0], newest: sorted[sorted.length - 1] }
}
```

최솟값/최댓값만 필요할 때도 불필요하게 정렬합니다.

**올바른 방법 (O(n) - 단일 루프):**

```typescript
function getLatestProject(projects: Project[]) {
  if (projects.length === 0) return null

  let latest = projects[0]

  for (let i = 1; i < projects.length; i++) {
    if (projects[i].updatedAt > latest.updatedAt) {
      latest = projects[i]
    }
  }

  return latest
}

function getOldestAndNewest(projects: Project[]) {
  if (projects.length === 0) return { oldest: null, newest: null }

  let oldest = projects[0]
  let newest = projects[0]

  for (let i = 1; i < projects.length; i++) {
    if (projects[i].updatedAt < oldest.updatedAt) oldest = projects[i]
    if (projects[i].updatedAt > newest.updatedAt) newest = projects[i]
  }

  return { oldest, newest }
}
```

배열을 한 번만 순회하고, 복사도 없고, 정렬도 없습니다.

**대안 (소규모 배열에 Math.min/Math.max):**

```typescript
const numbers = [5, 2, 8, 1, 9]
const min = Math.min(...numbers)
const max = Math.max(...numbers)
```

소규모 배열에는 동작하지만, 스프레드 연산자 제한으로 인해 매우 큰 배열에서는 느리거나 에러가 발생할 수 있습니다. Chrome 143에서 최대 배열 길이는 약 124000, Safari 18에서는 638000입니다. 정확한 수치는 달라질 수 있으므로 안정성을 위해 루프 방식을 사용하세요.
