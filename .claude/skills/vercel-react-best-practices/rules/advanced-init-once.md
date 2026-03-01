---
title: Initialize App Once, Not Per Mount
impact: LOW-MEDIUM
impactDescription: avoids duplicate init in development
tags: initialization, useEffect, app-startup, side-effects
---

## 마운트마다가 아닌 앱 시작 시 한 번 초기화

앱 로드 시 한 번만 실행되어야 하는 앱 전체 초기화를 컴포넌트의 `useEffect([])`에 넣지 마세요. 컴포넌트는 재마운트될 수 있고 effect도 다시 실행됩니다. 대신 모듈 레벨 가드를 사용하거나 엔트리 모듈 최상위에서 초기화하세요.

**잘못된 방법 (개발 환경에서 두 번 실행, 재마운트 시 재실행):**

```tsx
function Comp() {
  useEffect(() => {
    loadFromStorage()
    checkAuthToken()
  }, [])

  // ...
}
```

**올바른 방법 (앱 로드 시 한 번만 실행):**

```tsx
let didInit = false

function Comp() {
  useEffect(() => {
    if (didInit) return
    didInit = true
    loadFromStorage()
    checkAuthToken()
  }, [])

  // ...
}
```

참조: [애플리케이션 초기화](https://react.dev/learn/you-might-not-need-an-effect#initializing-the-application)
