---
name: vercel-react-best-practices
description: React and Next.js performance optimization guidelines from Vercel Engineering. This skill should be used when writing, reviewing, or refactoring React/Next.js code to ensure optimal performance patterns. Triggers on tasks involving React components, Next.js pages, data fetching, bundle optimization, or performance improvements.
license: MIT
metadata:
  author: vercel
  version: "1.0.0"
---

# Vercel React 모범 사례

Vercel이 관리하는 React 및 Next.js 애플리케이션을 위한 종합 성능 최적화 가이드. 자동화된 리팩토링과 코드 생성을 안내하기 위해 영향도 우선순위로 정렬된 8개 카테고리에 걸쳐 57개 규칙을 포함합니다.

## 적용 시점

다음과 같은 경우에 이 가이드라인을 참조하세요:
- 새로운 React 컴포넌트 또는 Next.js 페이지 작성 시
- 데이터 페칭(클라이언트 또는 서버 사이드) 구현 시
- 성능 문제를 위한 코드 리뷰 시
- 기존 React/Next.js 코드 리팩토링 시
- 번들 크기 또는 로드 시간 최적화 시

## 우선순위별 규칙 카테고리

| 우선순위 | 카테고리 | 영향도 | 접두사 |
|---------|---------|-------|-------|
| 1 | 워터폴 제거 | CRITICAL | `async-` |
| 2 | 번들 크기 최적화 | CRITICAL | `bundle-` |
| 3 | 서버 사이드 성능 | HIGH | `server-` |
| 4 | 클라이언트 사이드 데이터 페칭 | MEDIUM-HIGH | `client-` |
| 5 | 리렌더 최적화 | MEDIUM | `rerender-` |
| 6 | 렌더링 성능 | MEDIUM | `rendering-` |
| 7 | JavaScript 성능 | LOW-MEDIUM | `js-` |
| 8 | 고급 패턴 | LOW | `advanced-` |

## 빠른 참조

### 1. 워터폴 제거 (CRITICAL)

- `async-defer-await` - 실제로 사용되는 분기로 await 이동
- `async-parallel` - 독립적인 작업에 Promise.all() 사용
- `async-dependencies` - 부분 의존성에 better-all 사용
- `async-api-routes` - API 라우트에서 프로미스를 일찍 시작하고 늦게 await
- `async-suspense-boundaries` - 콘텐츠 스트리밍을 위해 Suspense 사용

### 2. 번들 크기 최적화 (CRITICAL)

- `bundle-barrel-imports` - 직접 임포트 사용, barrel 파일 지양
- `bundle-dynamic-imports` - 무거운 컴포넌트에 next/dynamic 사용
- `bundle-defer-third-party` - 하이드레이션 후에 분석/로깅 로드
- `bundle-conditional` - 기능이 활성화될 때만 모듈 로드
- `bundle-preload` - 인식 속도를 위해 hover/focus 시 미리 로드

### 3. 서버 사이드 성능 (HIGH)

- `server-auth-actions` - API 라우트처럼 서버 액션 인증
- `server-cache-react` - 요청별 중복 제거를 위해 React.cache() 사용
- `server-cache-lru` - 요청 간 캐싱을 위해 LRU 캐시 사용
- `server-dedup-props` - RSC props에서 중복 직렬화 지양
- `server-serialization` - 클라이언트 컴포넌트로 전달되는 데이터 최소화
- `server-parallel-fetching` - 패치를 병렬화하기 위해 컴포넌트 재구성
- `server-after-nonblocking` - 비차단 작업에 after() 사용

### 4. 클라이언트 사이드 데이터 페칭 (MEDIUM-HIGH)

- `client-swr-dedup` - 자동 요청 중복 제거를 위해 SWR 사용
- `client-event-listeners` - 전역 이벤트 리스너 중복 제거
- `client-passive-event-listeners` - 스크롤에 passive 리스너 사용
- `client-localstorage-schema` - localStorage 데이터 버전 관리 및 최소화

### 5. 리렌더 최적화 (MEDIUM)

- `rerender-defer-reads` - 콜백에서만 사용되는 상태 구독 지양
- `rerender-memo` - 비용이 많이 드는 작업을 메모이즈된 컴포넌트로 추출
- `rerender-memo-with-default-value` - 기본 비원시형 props 호이스팅
- `rerender-dependencies` - effect에서 원시형 의존성 사용
- `rerender-derived-state` - 원시 값이 아닌 파생된 불리언 구독
- `rerender-derived-state-no-effect` - effect가 아닌 렌더 중 상태 파생
- `rerender-functional-setstate` - 안정적인 콜백을 위해 함수형 setState 사용
- `rerender-lazy-state-init` - 비용이 많이 드는 값에 useState에 함수 전달
- `rerender-simple-expression-in-memo` - 단순 원시형에는 memo 지양
- `rerender-move-effect-to-event` - 상호작용 로직을 이벤트 핸들러에 배치
- `rerender-transitions` - 긴급하지 않은 업데이트에 startTransition 사용
- `rerender-use-ref-transient-values` - 일시적으로 자주 변하는 값에 ref 사용

### 6. 렌더링 성능 (MEDIUM)

- `rendering-animate-svg-wrapper` - SVG 요소가 아닌 div 래퍼 애니메이션
- `rendering-content-visibility` - 긴 목록에 content-visibility 사용
- `rendering-hoist-jsx` - 컴포넌트 외부로 정적 JSX 추출
- `rendering-svg-precision` - SVG 좌표 정밀도 줄이기
- `rendering-hydration-no-flicker` - 클라이언트 전용 데이터에 인라인 스크립트 사용
- `rendering-hydration-suppress-warning` - 예상되는 불일치 경고 억제
- `rendering-activity` - show/hide에 Activity 컴포넌트 사용
- `rendering-conditional-render` - 조건부 렌더링에 &&가 아닌 삼항 연산자 사용
- `rendering-usetransition-loading` - 로딩 상태에 useTransition 선호

### 7. JavaScript 성능 (LOW-MEDIUM)

- `js-batch-dom-css` - class나 cssText를 통해 CSS 변경 그룹화
- `js-index-maps` - 반복 조회를 위해 Map 구축
- `js-cache-property-access` - 루프에서 객체 속성 캐시
- `js-cache-function-results` - 모듈 레벨 Map에서 함수 결과 캐시
- `js-cache-storage` - localStorage/sessionStorage 읽기 캐시
- `js-combine-iterations` - 여러 filter/map을 하나의 루프로 결합
- `js-length-check-first` - 비용이 많이 드는 비교 전에 배열 길이 확인
- `js-early-exit` - 함수에서 조기 반환
- `js-hoist-regexp` - 루프 외부로 RegExp 생성 호이스팅
- `js-min-max-loop` - sort 대신 루프로 min/max 구하기
- `js-set-map-lookups` - O(1) 조회를 위해 Set/Map 사용
- `js-tosorted-immutable` - 불변성을 위해 toSorted() 사용

### 8. 고급 패턴 (LOW)

- `advanced-event-handler-refs` - ref에 이벤트 핸들러 저장
- `advanced-init-once` - 앱 로드당 한 번만 앱 초기화
- `advanced-use-latest` - 안정적인 콜백 ref를 위해 useLatest 사용

## 사용 방법

개별 규칙 파일에서 상세 설명과 코드 예시 확인:

```
rules/async-parallel.md
rules/bundle-barrel-imports.md
```

각 규칙 파일에는 다음이 포함됩니다:
- 왜 중요한지에 대한 간략한 설명
- 설명이 있는 잘못된 코드 예시
- 설명이 있는 올바른 코드 예시
- 추가 컨텍스트 및 참조

## 전체 컴파일 문서

모든 규칙이 확장된 완전한 가이드: `AGENTS.md`
