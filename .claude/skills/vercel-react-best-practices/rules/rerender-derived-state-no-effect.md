---
title: Calculate Derived State During Rendering
impact: MEDIUM
impactDescription: avoids redundant renders and state drift
tags: rerender, derived-state, useEffect, state
---

## 렌더링 중 파생 상태 계산

값이 현재 props/state에서 계산될 수 있다면 state에 저장하거나 effect에서 업데이트하지 않습니다. 렌더 중에 파생하여 추가적인 렌더와 상태 불일치를 방지합니다. prop 변경에만 응답하는 effect에서 상태를 설정하지 않고 파생 값이나 키 기반 리셋을 선호합니다.

**잘못된 방법 (중복 state와 effect):**

```tsx
function Form() {
  const [firstName, setFirstName] = useState('First')
  const [lastName, setLastName] = useState('Last')
  const [fullName, setFullName] = useState('')

  useEffect(() => {
    setFullName(firstName + ' ' + lastName)
  }, [firstName, lastName])

  return <p>{fullName}</p>
}
```

**올바른 방법 (렌더 중에 파생):**

```tsx
function Form() {
  const [firstName, setFirstName] = useState('First')
  const [lastName, setLastName] = useState('Last')
  const fullName = firstName + ' ' + lastName

  return <p>{fullName}</p>
}
```

참조: [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
