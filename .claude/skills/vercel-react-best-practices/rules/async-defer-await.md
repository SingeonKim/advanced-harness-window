---
title: Defer Await Until Needed
impact: HIGH
impactDescription: avoids blocking unused code paths
tags: async, await, conditional, optimization
---

## 필요할 때까지 Await 지연

실제로 사용되는 분기로 `await` 작업을 이동하여 필요하지 않은 코드 경로가 차단되는 것을 방지합니다.

**잘못된 방법 (두 분기 모두 차단):**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  const userData = await fetchUserData(userId)

  if (skipProcessing) {
    // 즉시 반환하지만 이미 userData를 기다림
    return { skipped: true }
  }

  // 이 분기만 userData 사용
  return processUserData(userData)
}
```

**올바른 방법 (필요할 때만 차단):**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  if (skipProcessing) {
    // 기다리지 않고 즉시 반환
    return { skipped: true }
  }

  // 필요할 때만 페치
  const userData = await fetchUserData(userId)
  return processUserData(userData)
}
```

**다른 예시 (조기 반환 최적화):**

```typescript
// 잘못된 방법: 항상 권한을 페치함
async function updateResource(resourceId: string, userId: string) {
  const permissions = await fetchPermissions(userId)
  const resource = await getResource(resourceId)

  if (!resource) {
    return { error: 'Not found' }
  }

  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }

  return await updateResourceData(resource, permissions)
}

// 올바른 방법: 필요할 때만 페치
async function updateResource(resourceId: string, userId: string) {
  const resource = await getResource(resourceId)

  if (!resource) {
    return { error: 'Not found' }
  }

  const permissions = await fetchPermissions(userId)

  if (!permissions.canEdit) {
    return { error: 'Forbidden' }
  }

  return await updateResourceData(resource, permissions)
}
```

건너뛰는 분기가 자주 실행되거나 지연된 작업이 비용이 많이 드는 경우에 특히 유용한 최적화입니다.
