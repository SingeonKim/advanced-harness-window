---
title: Put Interaction Logic in Event Handlers
impact: MEDIUM
impactDescription: avoids effect re-runs and duplicate side effects
tags: rerender, useEffect, events, side-effects, dependencies
---

## 상호작용 로직을 이벤트 핸들러에 배치

사이드 이펙트가 특정 사용자 액션(submit, click, drag)에 의해 트리거되는 경우 해당 이벤트 핸들러에서 실행합니다. 액션을 state + effect로 모델링하면 관련 없는 변경에서도 effect가 재실행되고 액션이 중복될 수 있습니다.

**잘못된 방법 (이벤트를 state + effect로 모델링):**

```tsx
function Form() {
  const [submitted, setSubmitted] = useState(false)
  const theme = useContext(ThemeContext)

  useEffect(() => {
    if (submitted) {
      post('/api/register')
      showToast('Registered', theme)
    }
  }, [submitted, theme])

  return <button onClick={() => setSubmitted(true)}>Submit</button>
}
```

**올바른 방법 (핸들러에서 직접 처리):**

```tsx
function Form() {
  const theme = useContext(ThemeContext)

  function handleSubmit() {
    post('/api/register')
    showToast('Registered', theme)
  }

  return <button onClick={handleSubmit}>Submit</button>
}
```

참조: [Should this code move to an event handler?](https://react.dev/learn/removing-effect-dependencies#should-this-code-move-to-an-event-handler)
