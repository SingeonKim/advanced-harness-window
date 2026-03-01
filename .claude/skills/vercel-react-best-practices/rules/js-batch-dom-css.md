---
title: Avoid Layout Thrashing
impact: MEDIUM
impactDescription: prevents forced synchronous layouts and reduces performance bottlenecks
tags: javascript, dom, css, performance, reflow, layout-thrashing
---

## 레이아웃 스래싱 방지

스타일 쓰기와 레이아웃 읽기를 섞어서 사용하지 않습니다. 스타일 변경 사이에 레이아웃 속성(`offsetWidth`, `getBoundingClientRect()`, `getComputedStyle()`)을 읽으면 브라우저가 강제로 동기 리플로우를 실행합니다.

**괜찮은 방법 (브라우저가 스타일 변경을 배치 처리):**
```typescript
function updateElementStyles(element: HTMLElement) {
  // 각 줄이 스타일을 무효화하지만 브라우저가 재계산을 배치 처리
  element.style.width = '100px'
  element.style.height = '200px'
  element.style.backgroundColor = 'blue'
  element.style.border = '1px solid black'
}
```

**잘못된 방법 (읽기와 쓰기가 섞여 리플로우 강제):**
```typescript
function layoutThrashing(element: HTMLElement) {
  element.style.width = '100px'
  const width = element.offsetWidth  // 리플로우 강제
  element.style.height = '200px'
  const height = element.offsetHeight  // 다시 리플로우 강제
}
```

**올바른 방법 (쓰기 배치 처리 후 한 번 읽기):**
```typescript
function updateElementStyles(element: HTMLElement) {
  // 모든 쓰기를 함께 배치
  element.style.width = '100px'
  element.style.height = '200px'
  element.style.backgroundColor = 'blue'
  element.style.border = '1px solid black'

  // 모든 쓰기가 완료된 후 읽기 (단일 리플로우)
  const { width, height } = element.getBoundingClientRect()
}
```

**올바른 방법 (읽기 배치 후 쓰기):**
```typescript
function avoidThrashing(element: HTMLElement) {
  // 읽기 단계 - 모든 레이아웃 쿼리를 먼저
  const rect1 = element.getBoundingClientRect()
  const offsetWidth = element.offsetWidth
  const offsetHeight = element.offsetHeight

  // 쓰기 단계 - 모든 스타일 변경을 나중에
  element.style.width = '100px'
  element.style.height = '200px'
}
```

**더 나은 방법: CSS 클래스 사용**
```css
.highlighted-box {
  width: 100px;
  height: 200px;
  background-color: blue;
  border: 1px solid black;
}
```
```typescript
function updateElementStyles(element: HTMLElement) {
  element.classList.add('highlighted-box')

  const { width, height } = element.getBoundingClientRect()
}
```

**React 예시:**
```tsx
// 잘못된 방법: 스타일 변경과 레이아웃 쿼리 섞기
function Box({ isHighlighted }: { isHighlighted: boolean }) {
  const ref = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (ref.current && isHighlighted) {
      ref.current.style.width = '100px'
      const width = ref.current.offsetWidth // 레이아웃 강제
      ref.current.style.height = '200px'
    }
  }, [isHighlighted])

  return <div ref={ref}>Content</div>
}

// 올바른 방법: 클래스 토글
function Box({ isHighlighted }: { isHighlighted: boolean }) {
  return (
    <div className={isHighlighted ? 'highlighted-box' : ''}>
      Content
    </div>
  )
}
```

가능한 경우 인라인 스타일 대신 CSS 클래스를 선호합니다. CSS 파일은 브라우저에 캐시되며 관심사 분리가 더 잘 이루어지고 유지 관리가 더 쉽습니다.

레이아웃을 강제하는 작업에 대한 자세한 내용은 [this gist](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)와 [CSS Triggers](https://csstriggers.com/)를 참조하세요.
