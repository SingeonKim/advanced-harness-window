# Next.js 프론트엔드 가이드라인 스킬

## 개요

이 스킬은 YGS (영영사) 프로젝트 기술 스택에 맞춘 종합 프론트엔드 개발 가이드라인을 제공합니다:

- **Next.js 15** (App Router)
- **React 19**
- **TypeScript**
- **shadcn/ui** (Radix UI + Tailwind CSS)
- **Tailwind CSS 4**
- **다중 인증** (Firebase, Kakao OAuth, 커스텀 JWT)
- **한국어 로컬라이제이션**

## 스킬 내용

1. **Component 패턴** - Server Components vs Client Components, 언제 사용할지
2. **데이터 패칭** - Server Component async 패칭, 클라이언트 사이드 패턴, API routes
3. **스타일링** - shadcn/ui components + Tailwind CSS 4 조합
4. **라우팅** - App Router 파일 기반 라우팅, 동적 라우트, 네비게이션
5. **로딩 & 에러 상태** - loading.tsx, error.tsx, Suspense 경계
6. **성능** - Server Components 최적화, dynamic imports, 이미지 최적화
7. **TypeScript** - 엄격한 타입 지정, Next.js 타입, component props
8. **인증** - Firebase, Kakao, 커스텀 JWT 다중 인증
9. **관리자 대시보드** - 회원 관리, 상담, 매칭 인터페이스
10. **공통 패턴** - 수동 유효성 검사를 사용한 폼, 모달 패턴, URL 기반 상태
11. **한국어 로컬라이제이션** - 모든 UI 텍스트는 한국어

## YGS 특화 기능

### 인증 시스템

```typescript
// AuthProvider를 통한 다중 인증
import { useAuth } from '@/providers/AuthProvider';

const { user, isLoading, loginWithKakao, loginWithGoogle, logout } = useAuth();
```

### API 클라이언트

```typescript
// 메인 API 클라이언트
import { api } from '@/lib/api';

// 관리자 전용 API
import { getMembers, updateMemberBasic, getDashboardStats } from '@/lib/adminApi';
```

### 상수 패턴

```typescript
// 한국어 레이블이 있는 열거형
import { USER_STATUS_OPTIONS, getEnumLabel } from '@/constants/enums';

<Select>
  {USER_STATUS_OPTIONS.map(opt => (
    <SelectItem key={opt.value} value={opt.value}>{opt.label}</SelectItem>
  ))}
</Select>
```

## 컴포넌트 현황

| 카테고리 | 개수 | 위치 |
|----------|------|------|
| Admin Components | 27 | `src/components/admin/` |
| Auth Components | 4 | `src/components/auth/` |
| Layout Components | 2 | `src/components/layout/` |
| Match Components | 3 | `src/components/match/` |
| Landing Sections | 7 | `src/components/sections/` |
| SEO Components | 4 | `src/components/seo/` |
| UI Components | 11 | `src/components/ui/` |
| **합계** | **~60** | |

## shadcn/ui 빠른 참조

### 컴포넌트 설치

```bash
# shadcn/ui 초기화
npx shadcn@latest init

# 컴포넌트 추가
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add input
npx shadcn@latest add dialog
npx shadcn@latest add form
npx shadcn@latest add select
```

### cn() 유틸리티

```typescript
// lib/utils.ts
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

// 사용 예시
<div className={cn("flex items-center", isActive && "bg-primary", className)}>
```

### 기본 컴포넌트 사용

```typescript
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';

<Card>
  <CardHeader>
    <CardTitle>제목</CardTitle>
  </CardHeader>
  <CardContent>
    <Input placeholder="입력해주세요" />
    <Button>확인</Button>
  </CardContent>
</Card>
```

## 프로젝트 구조 대응

스킬은 실제 프로젝트 구조를 참조합니다:

```
frontend/
  src/
    app/                  # Next.js App Router (스킬에서 참조)
      admin/              # 관리자 대시보드 라우트
      login/              # 인증 라우트
      form/               # 사용자 프로필 폼
      match/              # 매칭 인터페이스
    components/
      admin/              # 관리자 전용 컴포넌트
        modals/           # 편집 모달
      auth/               # 인증 컴포넌트
      layout/             # Navbar, Footer
      match/              # 매칭 카드
      sections/           # 랜딩 페이지 섹션
      seo/                # SEO 스키마 컴포넌트
      ui/                 # shadcn/ui 컴포넌트
    lib/
      api.ts              # 메인 API 클라이언트 (예시에 사용)
      adminApi.ts         # Admin API 메서드
      utils.ts            # cn() 유틸리티
      serverAuth.ts       # 서버 사이드 인증 (예시에 사용)
      firebaseAuth.ts     # Firebase 통합
      kakao.ts            # Kakao SDK 통합
      s3Upload.ts         # S3 업로드 유틸리티
    providers/
      AuthProvider.tsx    # Auth context provider
    types/                # TypeScript 타입 정의
      admin.ts            # Admin 타입 (362줄)
      match.ts            # Match 타입
    constants/
      enums.ts            # 한국어 레이블이 있는 열거형 옵션
```

## 파일 구조

```
.claude/skills/nextjs-frontend-guidelines/
  ├── skill.md                              # 메인 스킬 개요
  ├── README.md                             # 이 파일
  └── resources/
      ├── component-patterns.md             # Server vs Client components
      ├── data-fetching.md                  # Async 패칭, API routes
      ├── styling-guide.md                  # shadcn/ui + Tailwind CSS 4
      ├── file-organization.md              # 프로젝트 구조
      ├── routing-guide.md                  # App Router 패턴
      ├── loading-and-error-states.md       # 로딩 및 에러 처리
      ├── performance.md                    # 최적화 패턴
      ├── typescript-standards.md           # 타입 안전성
      ├── common-patterns.md                # 인증, 폼, 업로드
      └── complete-examples.md              # 완전한 예시
```

## 스킬 활성화

스킬은 다음 조건에서 활성화됩니다:

### 파일 트리거
- `frontend/src/**/*.tsx` 또는 `frontend/src/**/*.ts` 작업 시
- shadcn/ui imports, 'use client', Next.js 패턴을 포함하는 파일

### 프롬프트 트리거
- 키워드: "component", "React", "UI", "page", "Next.js", "server component", "shadcn", "styling", "admin", "auth"
- 의도 패턴: 컴포넌트 생성/편집, 스타일링 질문, Next.js 패턴

## 기술 스택 호환성

- **Next.js 15**: 모든 패턴이 App Router 사용
- **React 19**: Server/Client component 패턴
- **shadcn/ui**: Radix UI + Tailwind CSS 컴포넌트
- **TypeScript**: 전체적으로 엄격한 타입 지정
- **Tailwind CSS 4**: shadcn/ui와 조합
- **API 클라이언트**: 예시가 `src/lib/api.ts`와 `src/lib/adminApi.ts` 사용
- **인증**: 예시가 `src/providers/AuthProvider.tsx` 사용
- **상수**: 예시가 `src/constants/enums.ts` 사용

## 의존성

shadcn/ui 핵심 의존성 (이미 설치됨):

```bash
# 핵심 의존성
pnpm add clsx tailwind-merge class-variance-authority

# 폼용 (선택, YGS는 수동 유효성 검사 사용)
pnpm add react-hook-form @hookform/resolvers zod

# 아이콘 (이미 설치됨)
pnpm add lucide-react
```

---

**상태**: YGS 프로젝트용으로 업데이트됨
**업데이트**: 2026-01-14
**스택**: Next.js 15 + React 19 + shadcn/ui + Tailwind CSS 4 + 다중 인증 + 한국어
