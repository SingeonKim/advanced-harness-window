---

title: Extract Default Non-primitive Parameter Value from Memoized Component to Constant
impact: MEDIUM
impactDescription: restores memoization by using a constant for default value
tags: rerender, memo, optimization

---

## 메모이즈된 컴포넌트의 기본 비원시형 매개변수 값을 상수로 추출

메모이즈된 컴포넌트가 배열, 함수, 객체 같은 비원시형 선택적 매개변수에 기본값을 가질 때, 해당 매개변수 없이 컴포넌트를 호출하면 메모이제이션이 깨집니다. 이는 리렌더마다 새 값 인스턴스가 생성되어 `memo()`의 엄격한 동등성 비교를 통과하지 못하기 때문입니다.

이 문제를 해결하려면 기본값을 상수로 추출합니다.

**잘못된 방법 (`onClick`이 리렌더마다 다른 값을 가짐):**

```tsx
const UserAvatar = memo(function UserAvatar({ onClick = () => {} }: { onClick?: () => void }) {
  // ...
})

// 선택적 onClick 없이 사용
<UserAvatar />
```

**올바른 방법 (안정적인 기본값):**

```tsx
const NOOP = () => {};

const UserAvatar = memo(function UserAvatar({ onClick = NOOP }: { onClick?: () => void }) {
  // ...
})

// 선택적 onClick 없이 사용
<UserAvatar />
```
