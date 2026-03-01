---
title: Use useTransition Over Manual Loading States
impact: LOW
impactDescription: reduces re-renders and improves code clarity
tags: rendering, transitions, useTransition, loading, state
---

## 수동 로딩 상태 대신 useTransition 사용

로딩 상태에 수동 `useState` 대신 `useTransition`을 사용합니다. 내장된 `isPending` 상태를 제공하고 자동으로 트랜지션을 관리합니다.

**잘못된 방법 (수동 로딩 상태):**

```tsx
function SearchResults() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [isLoading, setIsLoading] = useState(false)

  const handleSearch = async (value: string) => {
    setIsLoading(true)
    setQuery(value)
    const data = await fetchResults(value)
    setResults(data)
    setIsLoading(false)
  }

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isLoading && <Spinner />}
      <ResultsList results={results} />
    </>
  )
}
```

**올바른 방법 (내장 pending 상태가 있는 useTransition):**

```tsx
import { useTransition, useState } from 'react'

function SearchResults() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [isPending, startTransition] = useTransition()

  const handleSearch = (value: string) => {
    setQuery(value) // 입력을 즉시 업데이트

    startTransition(async () => {
      // 결과를 페치하고 업데이트
      const data = await fetchResults(value)
      setResults(data)
    })
  }

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  )
}
```

**장점:**

- **자동 pending 상태**: `setIsLoading(true/false)` 수동 관리 불필요
- **오류 복원력**: 트랜지션이 예외를 발생시켜도 pending 상태가 올바르게 재설정됨
- **더 나은 응답성**: 업데이트 중에도 UI 응답성 유지
- **인터럽트 처리**: 새 트랜지션이 pending 중인 트랜지션을 자동으로 취소

참조: [useTransition](https://react.dev/reference/react/useTransition)
