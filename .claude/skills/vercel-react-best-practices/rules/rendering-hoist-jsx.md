---
title: Hoist Static JSX Elements
impact: LOW
impactDescription: avoids re-creation
tags: rendering, jsx, static, optimization
---

## 정적 JSX 요소 호이스팅

재생성을 방지하기 위해 컴포넌트 외부로 정적 JSX를 추출합니다.

**잘못된 방법 (매 렌더마다 요소 재생성):**

```tsx
function LoadingSkeleton() {
  return <div className="animate-pulse h-20 bg-gray-200" />
}

function Container() {
  return (
    <div>
      {loading && <LoadingSkeleton />}
    </div>
  )
}
```

**올바른 방법 (동일한 요소 재사용):**

```tsx
const loadingSkeleton = (
  <div className="animate-pulse h-20 bg-gray-200" />
)

function Container() {
  return (
    <div>
      {loading && loadingSkeleton}
    </div>
  )
}
```

이는 매 렌더마다 재생성하기 비용이 많이 드는 크고 정적인 SVG 노드에 특히 유용합니다.

**참고:** 프로젝트에 [React Compiler](https://react.dev/learn/react-compiler)가 활성화된 경우 컴파일러가 정적 JSX 요소를 자동으로 호이스팅하고 컴포넌트 리렌더를 최적화하므로 수동 호이스팅이 불필요합니다.
