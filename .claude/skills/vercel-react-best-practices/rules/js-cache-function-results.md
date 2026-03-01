---
title: Cache Repeated Function Calls
impact: MEDIUM
impactDescription: avoid redundant computation
tags: javascript, cache, memoization, performance
---

## 반복 함수 호출 캐시

렌더 중 동일한 함수가 같은 입력으로 반복 호출될 때 모듈 레벨 Map을 사용하여 함수 결과를 캐시합니다.

**잘못된 방법 (중복 계산):**

```typescript
function ProjectList({ projects }: { projects: Project[] }) {
  return (
    <div>
      {projects.map(project => {
        // 동일한 프로젝트 이름에 대해 slugify() 100번 이상 호출
        const slug = slugify(project.name)

        return <ProjectCard key={project.id} slug={slug} />
      })}
    </div>
  )
}
```

**올바른 방법 (캐시된 결과):**

```typescript
// 모듈 레벨 캐시
const slugifyCache = new Map<string, string>()

function cachedSlugify(text: string): string {
  if (slugifyCache.has(text)) {
    return slugifyCache.get(text)!
  }
  const result = slugify(text)
  slugifyCache.set(text, result)
  return result
}

function ProjectList({ projects }: { projects: Project[] }) {
  return (
    <div>
      {projects.map(project => {
        // 고유한 프로젝트 이름당 한 번만 계산
        const slug = cachedSlugify(project.name)

        return <ProjectCard key={project.id} slug={slug} />
      })}
    </div>
  )
}
```

**단일 값 함수를 위한 더 간단한 패턴:**

```typescript
let isLoggedInCache: boolean | null = null

function isLoggedIn(): boolean {
  if (isLoggedInCache !== null) {
    return isLoggedInCache
  }

  isLoggedInCache = document.cookie.includes('auth=')
  return isLoggedInCache
}

// 인증이 변경될 때 캐시 초기화
function onAuthChange() {
  isLoggedInCache = null
}
```

훅이 아닌 Map을 사용하면 유틸리티, 이벤트 핸들러 등 어디서든 동작합니다.

참조: [How we made the Vercel Dashboard twice as fast](https://vercel.com/blog/how-we-made-the-vercel-dashboard-twice-as-fast)
