---
title: Suppress Expected Hydration Mismatches
impact: LOW-MEDIUM
impactDescription: avoids noisy hydration warnings for known differences
tags: rendering, hydration, ssr, nextjs
---

## 예상되는 하이드레이션 불일치 억제

SSR 프레임워크(예: Next.js)에서 일부 값은 의도적으로 서버와 클라이언트에서 다릅니다(랜덤 ID, 날짜, 로케일/시간대 포맷). 이러한 *예상되는* 불일치의 경우 `suppressHydrationWarning`이 있는 요소로 동적 텍스트를 감싸 노이즈가 많은 경고를 방지합니다. 실제 버그를 숨기기 위해 사용하지 마세요. 과도하게 사용하지 마세요.

**잘못된 방법 (알려진 불일치 경고):**

```tsx
function Timestamp() {
  return <span>{new Date().toLocaleString()}</span>
}
```

**올바른 방법 (예상되는 불일치만 억제):**

```tsx
function Timestamp() {
  return (
    <span suppressHydrationWarning>
      {new Date().toLocaleString()}
    </span>
  )
}
```
