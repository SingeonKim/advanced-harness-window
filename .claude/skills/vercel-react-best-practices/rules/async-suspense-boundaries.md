---
title: Strategic Suspense Boundaries
impact: HIGH
impactDescription: faster initial paint
tags: async, suspense, streaming, layout-shift
---

## 전략적 Suspense 경계

JSX를 반환하기 전에 비동기 컴포넌트에서 데이터를 await하는 대신, Suspense 경계를 사용하여 데이터가 로드되는 동안 래퍼 UI를 더 빠르게 표시합니다.

**잘못된 방법 (래퍼가 데이터 페칭으로 인해 차단됨):**

```tsx
async function Page() {
  const data = await fetchData() // 전체 페이지를 차단

  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <DataDisplay data={data} />
      </div>
      <div>Footer</div>
    </div>
  )
}
```

중간 섹션만 데이터가 필요한데도 전체 레이아웃이 데이터를 기다립니다.

**올바른 방법 (래퍼가 즉시 표시되고, 데이터가 스트리밍됨):**

```tsx
function Page() {
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <Suspense fallback={<Skeleton />}>
          <DataDisplay />
        </Suspense>
      </div>
      <div>Footer</div>
    </div>
  )
}

async function DataDisplay() {
  const data = await fetchData() // 이 컴포넌트만 차단
  return <div>{data.content}</div>
}
```

Sidebar, Header, Footer는 즉시 렌더링됩니다. DataDisplay만 데이터를 기다립니다.

**대안 (컴포넌트 간 프로미스 공유):**

```tsx
function Page() {
  // 즉시 페치를 시작하지만 await하지 않음
  const dataPromise = fetchData()

  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <Suspense fallback={<Skeleton />}>
        <DataDisplay dataPromise={dataPromise} />
        <DataSummary dataPromise={dataPromise} />
      </Suspense>
      <div>Footer</div>
    </div>
  )
}

function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise) // 프로미스를 언래핑
  return <div>{data.content}</div>
}

function DataSummary({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise) // 동일한 프로미스 재사용
  return <div>{data.summary}</div>
}
```

두 컴포넌트가 동일한 프로미스를 공유하므로 페치가 한 번만 발생합니다. 레이아웃은 즉시 렌더링되고 두 컴포넌트가 함께 기다립니다.

**이 패턴을 사용하지 말아야 할 경우:**

- 레이아웃 결정에 필요한 중요 데이터 (위치에 영향을 줌)
- 폴드 위의 SEO 중요 콘텐츠
- Suspense 오버헤드가 그만한 가치가 없는 작고 빠른 쿼리
- 레이아웃 이동(로딩 → 콘텐츠 전환)을 피하고 싶을 때

**트레이드오프:** 더 빠른 초기 페인트 vs 잠재적 레이아웃 이동. UX 우선순위에 따라 선택하세요.
