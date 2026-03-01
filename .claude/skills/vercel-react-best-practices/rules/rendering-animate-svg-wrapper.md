---
title: Animate SVG Wrapper Instead of SVG Element
impact: LOW
impactDescription: enables hardware acceleration
tags: rendering, svg, css, animation, performance
---

## SVG 요소 대신 SVG 래퍼 애니메이션

많은 브라우저가 SVG 요소의 CSS3 애니메이션에 대한 하드웨어 가속을 지원하지 않습니다. SVG를 `<div>`로 감싸고 래퍼를 애니메이션합니다.

**잘못된 방법 (SVG를 직접 애니메이션 - 하드웨어 가속 없음):**

```tsx
function LoadingSpinner() {
  return (
    <svg
      className="animate-spin"
      width="24"
      height="24"
      viewBox="0 0 24 24"
    >
      <circle cx="12" cy="12" r="10" stroke="currentColor" />
    </svg>
  )
}
```

**올바른 방법 (래퍼 div 애니메이션 - 하드웨어 가속됨):**

```tsx
function LoadingSpinner() {
  return (
    <div className="animate-spin">
      <svg
        width="24"
        height="24"
        viewBox="0 0 24 24"
      >
        <circle cx="12" cy="12" r="10" stroke="currentColor" />
      </svg>
    </div>
  )
}
```

이는 모든 CSS 변환 및 트랜지션(`transform`, `opacity`, `translate`, `scale`, `rotate`)에 적용됩니다. 래퍼 div를 사용하면 브라우저가 더 부드러운 애니메이션을 위해 GPU 가속을 사용할 수 있습니다.
