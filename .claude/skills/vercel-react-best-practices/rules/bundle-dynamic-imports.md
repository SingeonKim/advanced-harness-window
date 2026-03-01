---
title: Dynamic Imports for Heavy Components
impact: CRITICAL
impactDescription: directly affects TTI and LCP
tags: bundle, dynamic-import, code-splitting, next-dynamic
---

## 무거운 컴포넌트에 Dynamic Import 사용

초기 렌더링에 필요하지 않은 대용량 컴포넌트를 지연 로드하기 위해 `next/dynamic`을 사용합니다.

**잘못된 방법 (Monaco가 메인 청크와 번들됨 ~300KB):**

```tsx
import { MonacoEditor } from './monaco-editor'

function CodePanel({ code }: { code: string }) {
  return <MonacoEditor value={code} />
}
```

**올바른 방법 (Monaco가 필요할 때 로드됨):**

```tsx
import dynamic from 'next/dynamic'

const MonacoEditor = dynamic(
  () => import('./monaco-editor').then(m => m.MonacoEditor),
  { ssr: false }
)

function CodePanel({ code }: { code: string }) {
  return <MonacoEditor value={code} />
}
```
