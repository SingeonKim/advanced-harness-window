---
name: nextjs-frontend-guidelines
description: Next.js 15 frontend development guidelines for YGS (영영사) React 19/TypeScript application. Modern patterns including App Router, Server/Client Components, shadcn/ui components, Tailwind CSS 4, multi-method authentication (Firebase/Kakao/JWT), admin dashboard patterns, and Korean localization. Use when creating components, pages, API routes, fetching data, styling, or working with frontend code.
---

# YGS를 위한 Next.js 15 프론트엔드 개발 가이드라인

## 목적

Next.js 15, React 19를 사용한 YGS (영영사) 프론트엔드 개발 종합 가이드입니다. App Router 패턴, Server/Client component 분리, shadcn/ui components, Tailwind CSS 4 스타일링, 다중 인증, 한국어 로컬라이제이션을 강조합니다.

## 이 스킬을 사용할 때

- 새 컴포넌트 또는 페이지 생성 시
- App Router로 새 기능 개발 시
- Server Components 또는 클라이언트 사이드 패턴으로 데이터 패칭 시
- shadcn/ui와 Tailwind CSS 4로 컴포넌트 스타일링 시
- API routes 또는 Server Actions 설정 시
- 인증 흐름 구현 시 (Firebase, Kakao, 커스텀 JWT)
- 관리자 대시보드 개발 시
- 성능 최적화 시
- 프론트엔드 코드 정리 시
- TypeScript 모범 사례 적용 시

---

## 빠른 시작

### 새 컴포넌트 체크리스트

컴포넌트를 만들 때 이 체크리스트를 따르세요:

- [ ] Server 또는 Client Component 여부 결정
- [ ] 필요한 경우에만 `'use client'` 지시문 사용
- [ ] TypeScript interface로 Props 타입 정의
- [ ] 프로젝트 imports에 `@/` import alias 사용
- [ ] 해당하는 경우 shadcn/ui components 사용
- [ ] 조건부 클래스에 `cn()` 유틸리티 사용
- [ ] 컴포넌트에 Named export 사용
- [ ] 가능하면 데이터 패칭에 Async Server Components 사용
- [ ] 인터랙티비티에 Client Components 사용 (useState, useEffect, 이벤트 핸들러)
- [ ] UI 레이블은 한국어 텍스트 사용

### 새 기능 체크리스트

기능을 만들 때 이 구조를 설정하세요:

- [ ] `src/components/{feature-name}/` 디렉토리 생성
- [ ] Server와 Client components 분리
- [ ] 필요시 API route 생성: `src/app/api/{feature}/route.ts`
- [ ] `src/types/`에 TypeScript 타입 설정
- [ ] `src/app/{feature-name}/page.tsx`에 라우트 생성
- [ ] 기본적으로 Server Components 사용
- [ ] 인터랙티비티에만 Client Components 추가
- [ ] 적절한 경우 변이에 Server Actions 사용
- [ ] 필요시 `src/constants/enums.ts`에 constants/enums 추가

---

## 프로젝트 구조

YGS 프로젝트 구조 (`@/` alias로 import):

```
src/
├── app/                        # Next.js App Router
│   ├── page.tsx                # 홈/랜딩 페이지
│   ├── layout.tsx              # 메타데이터가 있는 루트 레이아웃
│   ├── error.tsx               # 에러 경계
│   ├── admin/                  # 관리자 대시보드 (보호됨)
│   │   ├── page.tsx            # 대시보드 통계
│   │   ├── layout.tsx          # 인증 체크가 있는 Admin 레이아웃
│   │   ├── members/            # 회원 관리
│   │   │   ├── page.tsx        # 회원 목록
│   │   │   └── [id]/page.tsx   # 회원 상세
│   │   ├── consultations/      # 상담 관리
│   │   ├── matching/           # 매칭 인터페이스
│   │   └── couples/            # 커플 관리
│   ├── login/                  # 인증
│   │   ├── page.tsx            # 로그인 페이지
│   │   └── kakao-callback/     # Kakao OAuth 콜백
│   ├── form/                   # 사용자 프로필 폼
│   ├── match/                  # 매칭 인터페이스
│   ├── buy/                    # 멤버십 구매
│   └── api/
│       └── auth/session/       # 토큰 동기화 엔드포인트
├── components/                 # React 컴포넌트 (~60개)
│   ├── admin/                  # Admin 컴포넌트 (27개)
│   │   ├── DashboardStats.tsx
│   │   ├── MemberTable.tsx
│   │   ├── MemberFilters.tsx
│   │   ├── ConsultationFormModal.tsx
│   │   └── modals/             # 편집 모달
│   ├── auth/                   # Auth 컴포넌트 (4개)
│   │   ├── LoginForm.tsx
│   │   ├── SignupForm.tsx
│   │   └── SocialLoginButton.tsx
│   ├── layout/                 # 레이아웃 컴포넌트 (2개)
│   │   ├── Navbar.tsx
│   │   └── Footer.tsx
│   ├── match/                  # 매칭 컴포넌트 (3개)
│   ├── sections/               # 랜딩 페이지 섹션 (7개)
│   ├── seo/                    # SEO 스키마 컴포넌트 (4개)
│   └── ui/                     # shadcn/ui 컴포넌트 (11개)
│       ├── button.tsx
│       ├── card.tsx
│       ├── input.tsx
│       ├── dialog.tsx
│       ├── select.tsx
│       └── ...
├── lib/                        # 핵심 유틸리티
│   ├── api.ts                  # 메인 API 클라이언트 (토큰 관리)
│   ├── adminApi.ts             # Admin 전용 API 메서드
│   ├── serverAuth.ts           # 서버 사이드 인증 유효성 검사
│   ├── firebaseAuth.ts         # Firebase SDK 통합
│   ├── firebase.ts             # Firebase 설정
│   ├── kakao.ts                # Kakao SDK 통합
│   ├── s3Upload.ts             # S3 업로드 유틸리티
│   └── utils.ts                # cn() 헬퍼
├── providers/                  # Context providers
│   └── AuthProvider.tsx        # Auth 상태 context
├── types/                      # TypeScript 타입 정의
│   ├── index.ts                # 공통 타입
│   ├── admin.ts                # Admin 타입 (362줄)
│   └── match.ts                # Match 타입
├── constants/                  # 상수 & 열거형
│   └── enums.ts                # 한국어 레이블이 있는 열거형 옵션
└── middleware.ts               # 라우트 보호
```

---

## Import 패턴

| 패턴 | 용도 | 예시 |
|------|------|------|
| `@/` | 프로젝트 imports (기본) | `import { api } from '@/lib/api'` |
| 상대 경로 | 같은 디렉토리 | `import { Component } from './Component'` |
| `type` | 타입 전용 imports | `import type { User } from '@/types'` |

---

## 공통 Imports 치트시트

```typescript
// Server Component ('use client' 없음)
import { Suspense } from 'react';
import { redirect } from 'next/navigation';
import { cookies } from 'next/headers';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { getServerSession } from '@/lib/serverAuth';
import type { Metadata } from 'next';

// Client Component
'use client';

import { useState, useEffect, useCallback } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { useAuth } from '@/providers/AuthProvider';
import { api } from '@/lib/api';
import { cn } from '@/lib/utils';
import { Loader2 } from 'lucide-react';

// Admin API
import { getMembers, updateMemberBasic, getDashboardStats } from '@/lib/adminApi';

// 타입
import type { AdminMember, MemberDetail, MemberFilter } from '@/types/admin';
import type { MatchCard } from '@/types/match';

// 상수
import { USER_STATUS_OPTIONS, GENDER_OPTIONS, getEnumLabel } from '@/constants/enums';
```

---

## 주제별 가이드

### shadcn/ui 개요

**shadcn/ui란?**
- Radix UI 기반의 아름답고 접근 가능한 컴포넌트
- 프로젝트에 복사/붙여넣기하는 컴포넌트 (npm 패키지 의존성 아님)
- Tailwind CSS로 완전히 커스터마이즈 가능
- 완전한 타입 안전성을 갖춘 TypeScript 우선

**핵심 개념:**
- 컴포넌트는 `src/components/ui/`에 위치
- 클래스 병합에 `cn()` 유틸리티 사용 (clsx + tailwind-merge)
- class-variance-authority (cva)를 통한 Variants
- Radix UI 접근성 패턴 준수

**YGS에서 사용 가능한 컴포넌트:**
- button, input, textarea, card, dialog, select, checkbox, badge, alert, skeleton, image-upload

**컴포넌트 추가:**
```bash
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add dialog
npx shadcn@latest add form
```

---

### Component 패턴

**Server vs Client Components:**
- **Server Components (기본값)**: 데이터 패칭, 정적 콘텐츠, 인터랙티비티 없음
- **Client Components ('use client')**: 상태, effects, 이벤트 핸들러, 브라우저 API

**핵심 개념:**
- Server Components는 async이며 직접 데이터를 패칭
- Client Components는 파일 상단에 'use client' 지시문 필요
- 더 나은 성능을 위해 Client Components 최소화
- Server에서 Client Components로 props를 통해 데이터 전달
- 컴포넌트 구조: Props -> Hooks -> Handlers -> Render -> Export

**YGS 특화 패턴:**
- 대부분의 admin 컴포넌트는 Client Components (상태 관리가 많음)
- 랜딩 페이지 섹션은 Server Components (정적 콘텐츠)
- 폼은 수동 상태 관리 사용 (react-hook-form 대신)

**[전체 가이드: resources/component-patterns.md](resources/component-patterns.md)**

---

### 인증 (YGS 특화)

**다중 인증:**
1. **Firebase 소셜 인증**: Google, Apple
2. **Kakao OAuth**: Firebase 교환을 통한 서버 사이드 토큰 유효성 검사
3. **커스텀 JWT**: 60분 access token, 30일 refresh token

**AuthProvider 패턴:**
```typescript
'use client';

import { useAuth } from '@/providers/AuthProvider';

export function MyComponent() {
  const {
    user,
    isLoading,
    isAuthenticated,
    signupRequired,
    loginWithKakao,
    loginWithGoogle,
    logout,
    refreshUser,
  } = useAuth();

  if (isLoading) return <Loading />;
  if (!isAuthenticated) return <LoginPrompt />;

  return <div>Welcome, {user?.nickname}</div>;
}
```

**서버 사이드 인증 체크 (Admin 레이아웃):**
```typescript
// app/admin/layout.tsx
import { redirect } from 'next/navigation';
import { getServerSession } from '@/lib/serverAuth';

export default async function AdminLayout({ children }) {
  const session = await getServerSession();

  if (!session.isAuthenticated) {
    redirect('/login');
  }

  // JWT에서 admin claim 확인
  const claims = parseJwtClaims(session.accessToken);
  if (!claims?.is_admin) {
    redirect('/');
  }

  return (
    <div className="flex min-h-screen">
      <AdminSidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

**Hydration 보호 패턴:**
```typescript
'use client';

import { useState, useEffect } from 'react';
import { useAuth } from '@/providers/AuthProvider';

export function Navbar() {
  const { user, isLoading, isAuthenticated } = useAuth();
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  return (
    <nav>
      {/* 정적 콘텐츠는 항상 렌더링 */}
      <Logo />

      {/* mount 후에만 인증 의존 콘텐츠 표시 */}
      {mounted && !isLoading ? (
        isAuthenticated ? <UserMenu user={user} /> : <LoginButton />
      ) : (
        <div className="w-[72px] h-10" /> // 플레이스홀더
      )}
    </nav>
  );
}
```

---

### 데이터 패칭

**주요 패턴:**

**Server Component 데이터 패칭 (권장):**
```typescript
// app/admin/page.tsx
import { getDashboardStats, getRegistrationTrend } from '@/lib/adminApi';

export default async function AdminDashboard() {
  const [stats, trend] = await Promise.all([
    getDashboardStats(),
    getRegistrationTrend(7),
  ]);

  return <DashboardStats stats={stats} trend={trend} />;
}
```

**클라이언트 사이드 데이터 패칭 (Admin 패턴):**
```typescript
'use client';

import { useState, useEffect } from 'react';
import { getMembers } from '@/lib/adminApi';
import type { AdminMember, MemberFilter } from '@/types/admin';

export function MemberList() {
  const [members, setMembers] = useState<AdminMember[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchMembers() {
      try {
        setLoading(true);
        const response = await getMembers({ skip: 0, limit: 20 });
        setMembers(response.items);
      } catch (err) {
        setError(err instanceof Error ? err.message : '데이터를 불러오는데 실패했습니다.');
      } finally {
        setLoading(false);
      }
    }
    fetchMembers();
  }, []);

  if (loading) return <Skeleton />;
  if (error) return <Alert variant="destructive">{error}</Alert>;

  return <MemberTable members={members} />;
}
```

**[전체 가이드: resources/data-fetching.md](resources/data-fetching.md)**

---

### API 클라이언트 패턴

**메인 API 클라이언트 (`lib/api.ts`):**
```typescript
import { api } from '@/lib/api';

// 토큰 관리
import { getAccessToken, setTokens, clearTokens, hasTokens } from '@/lib/api';

// 인증 메서드
await api.post('/api/v1/auth/login', { phone, password });
await firebaseLogin(idToken);
await kakaoLogin(code, redirectUri);

// 제네릭 메서드
const data = await api.get<ResponseType>('/api/v1/endpoint');
await api.post('/api/v1/endpoint', body);
await api.patch('/api/v1/endpoint', changes);
```

**Admin API 클라이언트 (`lib/adminApi.ts`):**
```typescript
import {
  // 대시보드
  getDashboardStats,
  getRegistrationTrend,
  getGenderRatio,
  getReferralStats,

  // 회원 관리
  getMembers,
  getMemberDetail,
  updateMemberBasic,
  updateMemberProfile,
  updateMemberLifestyle,
  updateMemberPreference,
  updateMemberSubscription,
  exportMembersToExcel,

  // 상담
  getConsultations,
  createConsultation,
  updateConsultation,
  deleteConsultation,

  // 매칭
  getCandidates,
  getCompatibilityScore,
} from '@/lib/adminApi';
```

---

### 상수 & 열거형 패턴

**`constants/enums.ts`에 정의:**
```typescript
// constants/enums.ts
export const USER_STATUS_OPTIONS = [
  { value: "draft", label: "상담 전" },
  { value: "pending_review", label: "상담 예정" },
  { value: "active", label: "상담 완료" },
  { value: "suspended", label: "정지" },
  { value: "withdrawn", label: "탈퇴" },
] as const;

export const GENDER_OPTIONS = [
  { value: "male", label: "남성" },
  { value: "female", label: "여성" },
] as const;

// 헬퍼 함수
export function getEnumLabel(
  options: readonly { value: string; label: string }[],
  value: string | null | undefined
): string {
  if (!value) return "-";
  return options.find(o => o.value === value)?.label ?? value;
}

export function getUserStatusLabel(value: string | null | undefined): string {
  return getEnumLabel(USER_STATUS_OPTIONS, value);
}
```

**컴포넌트에서 사용:**
```typescript
import { USER_STATUS_OPTIONS, getEnumLabel } from '@/constants/enums';

// Select 컴포넌트에서
<Select value={status} onValueChange={setStatus}>
  <SelectTrigger>
    <SelectValue placeholder="상태 선택" />
  </SelectTrigger>
  <SelectContent>
    {USER_STATUS_OPTIONS.map(option => (
      <SelectItem key={option.value} value={option.value}>
        {option.label}
      </SelectItem>
    ))}
  </SelectContent>
</Select>

// 레이블 표시
<span>{getEnumLabel(USER_STATUS_OPTIONS, member.status)}</span>
```

---

### 폼 패턴 (YGS 특화)

**패턴 1: 수동 상태 관리 (YGS에서 가장 일반적):**
```typescript
'use client';

import { useState, useCallback } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Loader2 } from 'lucide-react';

interface FormData {
  phone: string;
  name: string;
  gender: string;
  birthYear: string;
}

interface FormErrors {
  phone?: string;
  name?: string;
  gender?: string;
  birthYear?: string;
}

export function SignupForm() {
  const [formData, setFormData] = useState<FormData>({
    phone: '', name: '', gender: '', birthYear: ''
  });
  const [errors, setErrors] = useState<FormErrors>({});
  const [loading, setLoading] = useState(false);

  const validateForm = (): boolean => {
    const newErrors: FormErrors = {};

    if (!/^010-\d{4}-\d{4}$/.test(formData.phone)) {
      newErrors.phone = "올바른 전화번호 형식이 아닙니다.";
    }
    if (formData.name.length < 2) {
      newErrors.name = "이름은 2자 이상이어야 합니다.";
    }
    if (!formData.gender) {
      newErrors.gender = "성별을 선택해주세요.";
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleInputChange = useCallback((field: keyof FormData, value: string) => {
    setFormData(prev => ({ ...prev, [field]: value }));
    if (errors[field]) {
      setErrors(prev => ({ ...prev, [field]: undefined }));
    }
  }, [errors]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!validateForm()) return;

    setLoading(true);
    try {
      await api.post('/signup', formData);
    } catch (err) {
      setErrors({ ...errors, phone: "회원가입에 실패했습니다." });
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div className="space-y-2">
        <Input
          value={formData.phone}
          onChange={e => handleInputChange('phone', e.target.value)}
          placeholder="전화번호"
          className={cn(errors.phone && "border-red-500")}
        />
        {errors.phone && <p className="text-sm text-red-500">{errors.phone}</p>}
      </div>
      <Button type="submit" disabled={loading} className="w-full">
        {loading ? <Loader2 className="animate-spin mr-2 h-4 w-4" /> : null}
        {loading ? "처리 중..." : "가입하기"}
      </Button>
    </form>
  );
}
```

**패턴 2: 모달 폼 (Admin 편집):**
```typescript
'use client';

import { useState, useEffect } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { updateMemberBasic } from '@/lib/adminApi';
import type { MemberDetail } from '@/types/admin';

interface EditModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSuccess: (member: MemberDetail) => void;
  member: MemberDetail;
}

export function BasicInfoEditModal({ isOpen, onClose, onSuccess, member }: EditModalProps) {
  const [formData, setFormData] = useState({ name: '', status: '' });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    if (isOpen) {
      setFormData({ name: member.name, status: member.status });
      setError('');
    }
  }, [isOpen, member]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    try {
      // 변경된 필드만 전송
      const changes: Record<string, string> = {};
      if (formData.name !== member.name) changes.name = formData.name;
      if (formData.status !== member.status) changes.status = formData.status;

      if (Object.keys(changes).length === 0) {
        onClose();
        return;
      }

      const result = await updateMemberBasic(member.id, changes);
      onSuccess(result);
      onClose();
    } catch (err) {
      setError(err instanceof Error ? err.message : '수정에 실패했습니다.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <Dialog open={isOpen} onOpenChange={onClose}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>기본 정보 수정</DialogTitle>
        </DialogHeader>
        <form onSubmit={handleSubmit} className="space-y-4">
          <Input
            value={formData.name}
            onChange={e => setFormData(prev => ({ ...prev, name: e.target.value }))}
            placeholder="이름"
          />
          {error && <p className="text-sm text-red-500">{error}</p>}
          <DialogFooter>
            <Button type="button" variant="outline" onClick={onClose}>취소</Button>
            <Button type="submit" disabled={loading}>
              {loading ? '저장 중...' : '저장'}
            </Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

---

### URL 기반 상태 패턴 (페이지네이션/필터링)

```typescript
'use client';

import { useCallback } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';
import type { MemberFilter } from '@/types/admin';

export function useMemberFilters() {
  const searchParams = useSearchParams();
  const router = useRouter();

  const filters: MemberFilter = {
    status: searchParams.get('status') || '',
    gender: searchParams.get('gender') || '',
    search: searchParams.get('search') || '',
    skip: Number(searchParams.get('skip')) || 0,
    limit: Number(searchParams.get('limit')) || 20,
  };

  const updateURL = useCallback((newFilters: Partial<MemberFilter>) => {
    const params = new URLSearchParams();
    const merged = { ...filters, ...newFilters };

    if (merged.status) params.set('status', merged.status);
    if (merged.gender) params.set('gender', merged.gender);
    if (merged.search) params.set('search', merged.search);
    if (merged.skip) params.set('skip', String(merged.skip));
    if (merged.limit !== 20) params.set('limit', String(merged.limit));

    router.push(`/admin/members${params.toString() ? `?${params}` : ''}`);
  }, [filters, router]);

  return { filters, updateURL };
}
```

---

### 스타일링

**shadcn/ui + Tailwind CSS 4:**
- 기본: Tailwind와 함께 shadcn/ui 사전 빌드 컴포넌트 사용
- 커스터마이즈: Tailwind 유틸리티 클래스로 덮어쓰기
- 클래스 병합: 조건부/병합 클래스 이름에 `cn()` 유틸리티 사용

**cn() 유틸리티 (중요):**
```typescript
import { cn } from '@/lib/utils';

// 조건부 클래스
<div className={cn(
  "flex items-center gap-2",
  isActive && "bg-primary text-white",
  isScrolled && "shadow-sm",
  className
)}>

// 상태 기반 스타일링
const statusColors: Record<string, string> = {
  draft: "bg-slate-100 text-slate-600",
  active: "bg-green-100 text-green-700",
  suspended: "bg-red-100 text-red-600",
};

<Badge className={cn(statusColors[status] || "bg-gray-100")}>
  {getStatusLabel(status)}
</Badge>
```

**YGS 색상 시스템:**
```typescript
// 브랜드 기본 색상 (코랄/오렌지)
className="bg-primary text-white"
className="hover:bg-primary-dark"
className="text-primary"

// 그라데이션
className="bg-gradient-to-r from-amber-500 to-orange-500"

// 상태 색상
className="text-red-500"     // 에러, 파괴적 액션
className="bg-green-50"      // 성공
className="text-gray-500"    // 흐린 텍스트
```

**반응형 패턴:**
```typescript
// 그리드
className="grid grid-cols-1 lg:grid-cols-4 gap-4"
className="grid grid-cols-2 lg:grid-cols-4 gap-4"

// 텍스트
className="text-xl sm:text-2xl md:text-3xl lg:text-4xl"

// 표시
className="hidden md:flex"   // 모바일에서 숨김
className="md:hidden"        // 모바일만 표시

// 패딩
className="px-4 sm:px-6 md:px-12"
```

**[전체 가이드: resources/styling-guide.md](resources/styling-guide.md)**

---

### 파일 구성

**App Router 구조:**
```
src/
  app/
    page.tsx              # 홈 페이지 (/)
    layout.tsx            # 루트 레이아웃
    {route}/
      page.tsx            # 라우트 페이지
      layout.tsx          # 라우트 레이아웃 (선택)
      loading.tsx         # 라우트 로딩
      error.tsx           # 라우트 에러
    api/
      {route}/
        route.ts          # API 라우트 핸들러
  components/
    ui/                   # shadcn/ui 컴포넌트
      button.tsx
      card.tsx
      input.tsx
    admin/                # Admin 전용 컴포넌트
      DashboardStats.tsx
      MemberTable.tsx
      modals/             # 편집 모달
    {feature}/            # 기능 전용 컴포넌트
      Component.tsx
```

**컴포넌트 구성:**
- shadcn/ui 컴포넌트는 `src/components/ui/`에
- 기능 컴포넌트는 `src/components/{feature}/`에
- Admin 컴포넌트는 `src/components/admin/`에
- Server와 Client components 분리 유지

**[전체 가이드: resources/file-organization.md](resources/file-organization.md)**

---

### 로딩 & 에러 상태

**App Router 컨벤션:**

**로딩:**
```typescript
// loading.tsx (라우트 레벨)
import { Skeleton } from '@/components/ui/skeleton';

export default function Loading() {
  return (
    <div className="space-y-4">
      <Skeleton className="h-8 w-48" />
      <div className="grid grid-cols-1 lg:grid-cols-4 gap-4">
        {[...Array(4)].map((_, i) => (
          <Skeleton key={i} className="h-24" />
        ))}
      </div>
    </div>
  );
}
```

**에러 처리 (한국어):**
```typescript
// error.tsx (라우트 레벨)
'use client';

import { Button } from '@/components/ui/button';
import { AlertCircle } from 'lucide-react';

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div className="flex flex-col items-center justify-center py-12">
      <AlertCircle className="h-12 w-12 text-destructive mb-4" />
      <h2 className="text-xl font-bold mb-2">오류가 발생했습니다</h2>
      <p className="text-muted-foreground mb-4">페이지를 불러오는 중 문제가 발생했습니다.</p>
      <Button onClick={reset}>다시 시도</Button>
    </div>
  );
}
```

**[전체 가이드: resources/loading-and-error-states.md](resources/loading-and-error-states.md)**

---

### 성능

**Next.js 15 최적화:**
- 기본값으로 Server Components (클라이언트에 JS 없음)
- Dynamic imports: `const Heavy = dynamic(() => import('./Heavy'))`
- 이미지 최적화: `next/image` 컴포넌트
- 폰트 최적화: 빌트인 폰트 로딩
- Turbopack: 더 빠른 개발 빌드 (이미 활성화됨)

**React 19 패턴:**
- `useMemo`: 계산 비용이 큰 연산
- `useCallback`: 자식 컴포넌트에 전달되는 이벤트 핸들러
- `React.memo`: 불필요한 리렌더링 방지

**[전체 가이드: resources/performance.md](resources/performance.md)**

---

### TypeScript

**표준:**
- Strict 모드 활성화
- 함수에 명시적 반환 타입
- 타입 imports: `import type { User } from '@/types'`
- JSDoc이 있는 컴포넌트 prop interfaces
- `any` 타입 사용 금지 (필요시 `unknown` 사용)

**YGS 타입 패턴:**
```typescript
// types/admin.ts
interface AdminMember {
  id: string;
  name: string;
  gender: "male" | "female";
  phone: string;
  status: string;
  birth_year: number | null;
  created_at: string;
}

interface MemberDetail extends AdminMember {
  profile: UserProfile | null;
  lifestyle: UserLifestyle | null;
  preference: UserPreference | null;
  subscription: UserSubscription | null;
  documents: UserDocument[];
  photos: UserPhoto[];
}

// 수정 요청 타입 (부분)
interface BasicInfoUpdateRequest {
  name?: string;
  status?: string;
  is_admin?: boolean;
  birth_year?: number;
  gender?: "male" | "female";
}
```

**[전체 가이드: resources/typescript-standards.md](resources/typescript-standards.md)**

---

## 탐색 가이드

| 필요한 경우... | 이 리소스 읽기 |
|--------------|---------------|
| 컴포넌트 생성 | [component-patterns.md](resources/component-patterns.md) |
| 데이터 패칭 | [data-fetching.md](resources/data-fetching.md) |
| 파일/폴더 구성 | [file-organization.md](resources/file-organization.md) |
| 컴포넌트 스타일링 | [styling-guide.md](resources/styling-guide.md) |
| 라우팅 설정 | [routing-guide.md](resources/routing-guide.md) |
| 로딩/에러 처리 | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| 성능 최적화 | [performance.md](resources/performance.md) |
| TypeScript 타입 | [typescript-standards.md](resources/typescript-standards.md) |
| 폼/인증/API Routes | [common-patterns.md](resources/common-patterns.md) |
| 전체 예시 보기 | [complete-examples.md](resources/complete-examples.md) |

---

## 핵심 원칙

1. **Server Components 우선**: 기본값으로 Server Components 사용, 인터랙티비티에만 Client Components
2. **Async 데이터 패칭**: Server Components에서 직접 데이터 패칭
3. **클라이언트 JS 최소화**: 브라우저에 전송되는 JavaScript 적을수록 = 더 나은 성능
4. **App Router 컨벤션**: loading.tsx, error.tsx, layout.tsx를 적절히 사용
5. **shadcn/ui Components**: `@/components/ui/`에서 사전 빌드된 접근 가능한 컴포넌트 사용
6. **클래스에 cn()**: 조건부/병합 클래스 이름에 항상 `cn()` 사용
7. **@/ alias로 import**: 프로젝트 전체에서 일관된 import 경로
8. **타입 안전성**: 명시적 타입과 함께 엄격한 TypeScript
9. **한국어 로컬라이제이션**: 사용자 대면 텍스트는 모두 한국어
10. **AuthProvider**: 클라이언트 사이드 인증 상태에 `useAuth()` 훅 사용

---

## 빠른 참조: 템플릿

### Server Component 템플릿

```typescript
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { getDashboardStats } from '@/lib/adminApi';
import type { DashboardStats } from '@/types/admin';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: '대시보드 | YGS 관리자',
};

export default async function DashboardPage() {
  const stats: DashboardStats = await getDashboardStats();

  return (
    <div className="container py-8">
      <h1 className="text-2xl font-bold mb-6">대시보드</h1>
      <div className="grid grid-cols-2 lg:grid-cols-4 gap-4">
        <Card>
          <CardHeader>
            <CardTitle>전체 회원</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-3xl font-bold">{stats.total_members}</p>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

### Client Component 템플릿

```typescript
'use client';

import { useState, useCallback, useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Skeleton } from '@/components/ui/skeleton';
import { getMembers } from '@/lib/adminApi';
import { cn } from '@/lib/utils';
import { Loader2 } from 'lucide-react';
import type { AdminMember } from '@/types/admin';

interface MemberListProps {
  className?: string;
}

export function MemberList({ className }: MemberListProps) {
  const [members, setMembers] = useState<AdminMember[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchMembers = useCallback(async () => {
    try {
      setLoading(true);
      const response = await getMembers({ skip: 0, limit: 20 });
      setMembers(response.items);
    } catch (err) {
      setError(err instanceof Error ? err.message : '데이터를 불러오는데 실패했습니다.');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchMembers();
  }, [fetchMembers]);

  if (loading) {
    return (
      <div className="space-y-4">
        {[...Array(5)].map((_, i) => (
          <Skeleton key={i} className="h-16 w-full" />
        ))}
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center py-8">
        <p className="text-red-500 mb-4">{error}</p>
        <Button onClick={fetchMembers}>다시 시도</Button>
      </div>
    );
  }

  return (
    <div className={cn("space-y-4", className)}>
      {members.map(member => (
        <div key={member.id} className="p-4 border rounded-lg">
          <p className="font-medium">{member.name}</p>
          <p className="text-sm text-muted-foreground">{member.phone}</p>
        </div>
      ))}
    </div>
  );
}
```

전체 예시는 [resources/complete-examples.md](resources/complete-examples.md) 참조

---

## 관련 스킬

- **error-tracking**: Sentry를 사용한 에러 추적 (프론트엔드에도 적용)
- **fastapi-backend-guidelines**: 프론트엔드가 사용하는 백엔드 API 패턴

---

**스킬 상태**: YGS 프로젝트의 실제 코드베이스 패턴을 종합적으로 다루도록 업데이트됨
