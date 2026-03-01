---
title: Extract to Memoized Components
impact: MEDIUM
impactDescription: enables early returns
tags: rerender, memo, useMemo, optimization
---

## 메모이즈된 컴포넌트로 추출

비용이 많이 드는 작업을 메모이즈된 컴포넌트로 추출하여 계산 전에 조기 반환을 가능하게 합니다.

**잘못된 방법 (로딩 중에도 avatar를 계산):**

```tsx
function Profile({ user, loading }: Props) {
  const avatar = useMemo(() => {
    const id = computeAvatarId(user)
    return <Avatar id={id} />
  }, [user])

  if (loading) return <Skeleton />
  return <div>{avatar}</div>
}
```

**올바른 방법 (로딩 중일 때 계산 건너뜀):**

```tsx
const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  const id = useMemo(() => computeAvatarId(user), [user])
  return <Avatar id={id} />
})

function Profile({ user, loading }: Props) {
  if (loading) return <Skeleton />
  return (
    <div>
      <UserAvatar user={user} />
    </div>
  )
}
```

**참고:** 프로젝트에 [React Compiler](https://react.dev/learn/react-compiler)가 활성화된 경우, `memo()`와 `useMemo()`를 사용한 수동 메모이제이션이 필요하지 않습니다. 컴파일러가 자동으로 리렌더를 최적화합니다.
