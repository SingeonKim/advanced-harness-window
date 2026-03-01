---
title: Preload Based on User Intent
impact: MEDIUM
impactDescription: reduces perceived latency
tags: bundle, preload, user-intent, hover
---

## 사용자 의도에 따른 미리 로드

인식되는 지연 시간을 줄이기 위해 필요하기 전에 무거운 번들을 미리 로드합니다.

**예시 (hover/focus 시 미리 로드):**

```tsx
function EditorButton({ onClick }: { onClick: () => void }) {
  const preload = () => {
    if (typeof window !== 'undefined') {
      void import('./monaco-editor')
    }
  }

  return (
    <button
      onMouseEnter={preload}
      onFocus={preload}
      onClick={onClick}
    >
      Open Editor
    </button>
  )
}
```

**예시 (기능 플래그가 활성화될 때 미리 로드):**

```tsx
function FlagsProvider({ children, flags }: Props) {
  useEffect(() => {
    if (flags.editorEnabled && typeof window !== 'undefined') {
      void import('./monaco-editor').then(mod => mod.init())
    }
  }, [flags.editorEnabled])

  return <FlagsContext.Provider value={flags}>
    {children}
  </FlagsContext.Provider>
}
```

`typeof window !== 'undefined'` 검사는 SSR을 위해 미리 로드된 모듈이 번들링되는 것을 방지하여 서버 번들 크기와 빌드 속도를 최적화합니다.
