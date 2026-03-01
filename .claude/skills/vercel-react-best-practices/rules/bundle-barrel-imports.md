---
title: Avoid Barrel File Imports
impact: CRITICAL
impactDescription: 200-800ms import cost, slow builds
tags: bundle, imports, tree-shaking, barrel-files, performance
---

## Barrel 파일 임포트 지양

사용하지 않는 수천 개의 모듈을 로드하지 않도록 barrel 파일 대신 소스 파일에서 직접 임포트합니다. **Barrel 파일**은 여러 모듈을 재내보내는 진입점입니다 (예: `export * from './module'`을 하는 `index.js`).

인기 있는 아이콘 및 컴포넌트 라이브러리는 진입 파일에 **최대 10,000개의 재내보내기**를 가질 수 있습니다. 많은 React 패키지의 경우 **임포트만으로도 200-800ms가 소요**되어 개발 속도와 프로덕션 콜드 스타트 모두에 영향을 줍니다.

**트리 쉐이킹이 도움이 되지 않는 이유:** 라이브러리가 external로 표시되면(번들링되지 않음) 번들러가 최적화할 수 없습니다. 트리 쉐이킹을 위해 번들링하면 전체 모듈 그래프를 분석하느라 빌드가 상당히 느려집니다.

**잘못된 방법 (전체 라이브러리 임포트):**

```tsx
import { Check, X, Menu } from 'lucide-react'
// 1,583개 모듈 로드, 개발 시 ~2.8초 추가 소요
// 런타임 비용: 매 콜드 스타트마다 200-800ms

import { Button, TextField } from '@mui/material'
// 2,225개 모듈 로드, 개발 시 ~4.2초 추가 소요
```

**올바른 방법 (필요한 것만 임포트):**

```tsx
import Check from 'lucide-react/dist/esm/icons/check'
import X from 'lucide-react/dist/esm/icons/x'
import Menu from 'lucide-react/dist/esm/icons/menu'
// 3개 모듈만 로드 (~2KB vs ~1MB)

import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
// 사용하는 것만 로드
```

**대안 (Next.js 13.5+):**

```js
// next.config.js - optimizePackageImports 사용
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@mui/material']
  }
}

// 그러면 편리한 barrel 임포트를 유지할 수 있음:
import { Check, X, Menu } from 'lucide-react'
// 빌드 시 자동으로 직접 임포트로 변환됨
```

직접 임포트는 개발 부팅 15-70% 빠름, 빌드 28% 빠름, 콜드 스타트 40% 빠름, HMR 상당히 빠름을 제공합니다.

영향을 받는 주요 라이브러리: `lucide-react`, `@mui/material`, `@mui/icons-material`, `@tabler/icons-react`, `react-icons`, `@headlessui/react`, `@radix-ui/react-*`, `lodash`, `ramda`, `date-fns`, `rxjs`, `react-use`.

참조: [How we optimized package imports in Next.js](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js)
