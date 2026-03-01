---
title: Subscribe to Derived State
impact: MEDIUM
impactDescription: reduces re-render frequency
tags: rerender, derived-state, media-query, optimization
---

## 파생 상태 구독

리렌더 빈도를 줄이기 위해 연속 값 대신 파생된 불리언 상태를 구독합니다.

**잘못된 방법 (픽셀 변화마다 리렌더):**

```tsx
function Sidebar() {
  const width = useWindowWidth()  // 지속적으로 업데이트
  const isMobile = width < 768
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```

**올바른 방법 (불리언이 변경될 때만 리렌더):**

```tsx
function Sidebar() {
  const isMobile = useMediaQuery('(max-width: 767px)')
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```
