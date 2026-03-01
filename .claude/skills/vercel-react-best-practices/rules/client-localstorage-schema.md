---
title: Version and Minimize localStorage Data
impact: MEDIUM
impactDescription: prevents schema conflicts, reduces storage size
tags: client, localStorage, storage, versioning, data-minimization
---

## localStorage 데이터 버전 관리 및 최소화

키에 버전 접두사를 추가하고 필요한 필드만 저장합니다. 스키마 충돌을 방지하고 민감한 데이터가 실수로 저장되는 것을 막습니다.

**잘못된 방법:**

```typescript
// 버전 없음, 모든 것 저장, 오류 처리 없음
localStorage.setItem('userConfig', JSON.stringify(fullUserObject))
const data = localStorage.getItem('userConfig')
```

**올바른 방법:**

```typescript
const VERSION = 'v2'

function saveConfig(config: { theme: string; language: string }) {
  try {
    localStorage.setItem(`userConfig:${VERSION}`, JSON.stringify(config))
  } catch {
    // 시크릿/프라이빗 브라우징, 할당량 초과, 또는 비활성화된 경우 예외 발생
  }
}

function loadConfig() {
  try {
    const data = localStorage.getItem(`userConfig:${VERSION}`)
    return data ? JSON.parse(data) : null
  } catch {
    return null
  }
}

// v1에서 v2로 마이그레이션
function migrate() {
  try {
    const v1 = localStorage.getItem('userConfig:v1')
    if (v1) {
      const old = JSON.parse(v1)
      saveConfig({ theme: old.darkMode ? 'dark' : 'light', language: old.lang })
      localStorage.removeItem('userConfig:v1')
    }
  } catch {}
}
```

**서버 응답에서 최소 필드만 저장:**

```typescript
// User 객체에는 20개 이상의 필드가 있지만, UI에 필요한 것만 저장
function cachePrefs(user: FullUser) {
  try {
    localStorage.setItem('prefs:v1', JSON.stringify({
      theme: user.preferences.theme,
      notifications: user.preferences.notifications
    }))
  } catch {}
}
```

**항상 try-catch로 감싸기:** `getItem()`과 `setItem()`은 시크릿/프라이빗 브라우징(Safari, Firefox), 할당량 초과, 또는 비활성화된 경우 예외를 발생시킵니다.

**장점:** 버전 관리를 통한 스키마 진화, 저장 크기 감소, 토큰/개인 정보/내부 플래그 저장 방지.
