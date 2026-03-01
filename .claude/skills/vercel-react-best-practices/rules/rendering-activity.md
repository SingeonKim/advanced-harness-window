---
title: Use Activity Component for Show/Hide
impact: MEDIUM
impactDescription: preserves state/DOM
tags: rendering, activity, visibility, state-preservation
---

## show/hide에 Activity 컴포넌트 사용

자주 가시성을 토글하는 비용이 많이 드는 컴포넌트의 state/DOM을 보존하기 위해 React의 `<Activity>`를 사용합니다.

**사용법:**

```tsx
import { Activity } from 'react'

function Dropdown({ isOpen }: Props) {
  return (
    <Activity mode={isOpen ? 'visible' : 'hidden'}>
      <ExpensiveMenu />
    </Activity>
  )
}
```

비용이 많이 드는 리렌더와 state 손실을 방지합니다.
