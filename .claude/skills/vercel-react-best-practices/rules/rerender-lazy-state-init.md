---
title: Use Lazy State Initialization
impact: MEDIUM
impactDescription: wasted computation on every render
tags: react, hooks, useState, performance, initialization
---

## 지연 상태 초기화 사용

비용이 많이 드는 초기 값에 대해 `useState`에 함수를 전달합니다. 함수 형식 없이는 초기화 함수가 값이 한 번만 사용되더라도 모든 렌더마다 실행됩니다.

**잘못된 방법 (모든 렌더마다 실행):**

```tsx
function FilteredList({ items }: { items: Item[] }) {
  // buildSearchIndex()가 초기화 후에도 모든 렌더마다 실행됨
  const [searchIndex, setSearchIndex] = useState(buildSearchIndex(items))
  const [query, setQuery] = useState('')

  // query가 변경될 때 buildSearchIndex가 불필요하게 다시 실행됨
  return <SearchResults index={searchIndex} query={query} />
}

function UserProfile() {
  // JSON.parse가 모든 렌더마다 실행됨
  const [settings, setSettings] = useState(
    JSON.parse(localStorage.getItem('settings') || '{}')
  )

  return <SettingsForm settings={settings} onChange={setSettings} />
}
```

**올바른 방법 (한 번만 실행):**

```tsx
function FilteredList({ items }: { items: Item[] }) {
  // buildSearchIndex()가 초기 렌더에서만 실행됨
  const [searchIndex, setSearchIndex] = useState(() => buildSearchIndex(items))
  const [query, setQuery] = useState('')

  return <SearchResults index={searchIndex} query={query} />
}

function UserProfile() {
  // JSON.parse가 초기 렌더에서만 실행됨
  const [settings, setSettings] = useState(() => {
    const stored = localStorage.getItem('settings')
    return stored ? JSON.parse(stored) : {}
  })

  return <SettingsForm settings={settings} onChange={setSettings} />
}
```

localStorage/sessionStorage에서 초기 값 계산, 데이터 구조(인덱스, 맵) 구축, DOM 읽기, 또는 무거운 변환을 수행할 때 지연 초기화를 사용합니다.

단순 원시형(`useState(0)`), 직접 참조(`useState(props.value)`), 또는 저렴한 리터럴(`useState({})`)의 경우 함수 형식이 불필요합니다.
