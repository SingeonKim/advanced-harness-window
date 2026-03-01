# 파일 구성 - YGS Next.js 15

## 프로젝트 구조

```
src/
├── app/                        # Next.js App Router
│   ├── page.tsx                # 홈/랜딩 페이지 (/)
│   ├── layout.tsx              # 메타데이터가 있는 루트 레이아웃
│   ├── loading.tsx             # 루트 로딩 UI
│   ├── error.tsx               # 루트 에러 UI
│   ├── robots.ts               # SEO robots
│   ├── sitemap.ts              # SEO sitemap
│   │
│   ├── admin/                  # 관리자 대시보드 (보호됨)
│   │   ├── page.tsx            # 대시보드 (통계, 차트)
│   │   ├── layout.tsx          # 사이드바 + 인증이 있는 Admin 레이아웃
│   │   ├── loading.tsx         # Admin 로딩 UI
│   │   ├── error.tsx           # Admin 에러 UI
│   │   ├── members/
│   │   │   ├── page.tsx        # 페이지네이션이 있는 회원 목록
│   │   │   └── [id]/
│   │   │       └── page.tsx    # 회원 상세 (6개 탭)
│   │   ├── consultations/
│   │   │   └── page.tsx        # 상담 관리
│   │   ├── matching/
│   │   │   └── page.tsx        # 매칭 인터페이스
│   │   └── couples/
│   │       └── page.tsx        # 커플 관리
│   │
│   ├── login/                  # 인증
│   │   ├── page.tsx            # 로그인 페이지
│   │   └── kakao-callback/
│   │       └── page.tsx        # Kakao OAuth 콜백
│   │
│   ├── form/
│   │   └── page.tsx            # 사용자 프로필 폼
│   │
│   ├── match/
│   │   └── page.tsx            # 매칭 인터페이스
│   │
│   ├── buy/
│   │   └── page.tsx            # 멤버십 구매
│   │
│   └── api/                    # API Routes
│       └── auth/
│           └── session/
│               └── route.ts    # 토큰 동기화 엔드포인트 (POST/DELETE)
│
├── components/                 # React 컴포넌트 (~60개)
│   ├── admin/                  # Admin 컴포넌트 (27개)
│   │   ├── DashboardStats.tsx  # 통계 카드
│   │   ├── RegistrationTrendChart.tsx  # 라인 차트
│   │   ├── GenderRatioChart.tsx        # 파이 차트
│   │   ├── ReferralSourcesChart.tsx    # 막대 차트
│   │   ├── AdminSidebar.tsx    # 내비게이션 사이드바
│   │   ├── AdminHeader.tsx     # 사용자 정보가 있는 헤더
│   │   ├── MemberFilters.tsx   # 검색 + 필터 UI
│   │   ├── MemberTable.tsx     # 페이지네이션이 있는 테이블
│   │   ├── MemberBasicInfo.tsx # 기본 정보 표시
│   │   ├── MemberProfileTab.tsx
│   │   ├── MemberLifestyleTab.tsx
│   │   ├── MemberPreferenceTab.tsx
│   │   ├── MemberSubscriptionTab.tsx
│   │   ├── MemberDocumentsTab.tsx
│   │   ├── MemberSelector.tsx  # 회원 검색/선택
│   │   ├── ConsultationList.tsx
│   │   ├── ConsultationFormModal.tsx
│   │   ├── ConsultationCalendar.tsx
│   │   ├── CandidateList.tsx   # 매칭 후보
│   │   ├── MatchComparisonCard.tsx
│   │   ├── CoupleCard.tsx
│   │   ├── MatchingFilters.tsx
│   │   ├── ScoreBreakdown.tsx
│   │   └── modals/             # 편집 모달
│   │       ├── BasicInfoEditModal.tsx
│   │       ├── ProfileEditModal.tsx
│   │       ├── LifestyleEditModal.tsx
│   │       ├── PreferenceEditModal.tsx
│   │       ├── SubscriptionEditModal.tsx
│   │       └── PhotoEditModal.tsx
│   │
│   ├── auth/                   # Auth 컴포넌트 (4개)
│   │   ├── LoginForm.tsx       # 소셜 로그인 버튼 + 에러 처리
│   │   ├── SignupForm.tsx      # 유효성 검사가 있는 전화번호 가입
│   │   ├── SocialLoginButton.tsx # 일반 소셜 버튼
│   │   └── KakaoLoginButton.tsx  # Kakao 전용 버튼
│   │
│   ├── layout/                 # 레이아웃 컴포넌트 (2개)
│   │   ├── Navbar.tsx          # 인증이 있는 반응형 네비게이션
│   │   └── Footer.tsx          # 간단한 푸터
│   │
│   ├── match/                  # 매칭 컴포넌트 (3개)
│   │   ├── MatchCard.tsx       # 이미지 캐러셀이 있는 카드
│   │   ├── MatchCardDetailModal.tsx  # 상세 모달
│   │   └── MatchCardList.tsx   # 카드 그리드
│   │
│   ├── sections/               # 랜딩 페이지 섹션 (7개)
│   │   ├── Hero.tsx            # 배경 이미지가 있는 히어로
│   │   ├── Problem.tsx         # 문제 제기
│   │   ├── SocialProof.tsx     # 후기
│   │   ├── Process.tsx         # 이용 방법
│   │   ├── Founder.tsx         # 창업자 소개
│   │   ├── Pricing.tsx         # 요금제
│   │   └── ContactForm.tsx     # 문의 폼
│   │
│   ├── seo/                    # SEO 스키마 컴포넌트 (4개)
│   │   ├── OrganizationSchema.tsx
│   │   ├── ServiceSchema.tsx
│   │   ├── FAQSchema.tsx
│   │   └── WebSiteSchema.tsx
│   │
│   └── ui/                     # shadcn/ui 컴포넌트 (11개)
│       ├── button.tsx          # CVA variants
│       ├── input.tsx           # 간단한 래퍼
│       ├── card.tsx            # 서브 컴포넌트가 있는 Card
│       ├── dialog.tsx          # Radix Dialog 래퍼
│       ├── select.tsx          # Radix Select 래퍼
│       ├── textarea.tsx        # 간단한 래퍼
│       ├── checkbox.tsx        # Radix Checkbox 래퍼
│       ├── badge.tsx           # 상태 배지
│       ├── alert.tsx           # 알림 메시지
│       ├── skeleton.tsx        # 로딩 스켈레톤
│       └── image-upload.tsx    # 커스텀 파일 업로드
│
├── lib/                        # 핵심 유틸리티 (8개 파일)
│   ├── api.ts                  # 메인 API 클라이언트 (361줄)
│   │                           # - 토큰 관리
│   │                           # - 401 시 자동 갱신
│   │                           # - 제네릭 메서드
│   ├── adminApi.ts             # Admin 전용 API 메서드
│   │                           # - 대시보드 통계
│   │                           # - 회원 CRUD
│   │                           # - 상담
│   │                           # - 매칭
│   ├── serverAuth.ts           # 서버 사이드 인증 유효성 검사
│   ├── firebaseAuth.ts         # Firebase SDK 통합
│   ├── firebase.ts             # Firebase 설정
│   ├── kakao.ts                # Kakao SDK 통합
│   ├── s3Upload.ts             # S3 업로드 유틸리티
│   └── utils.ts                # cn() 헬퍼
│
├── providers/                  # Context providers
│   └── AuthProvider.tsx        # Auth 상태 context
│                               # - 다중 인증
│                               # - 회원가입 필요 흐름
│                               # - 토큰에서 자동 초기화
│
├── types/                      # TypeScript 타입 정의
│   ├── index.ts                # 공통 타입
│   ├── admin.ts                # Admin 타입 (362줄)
│   │                           # - 대시보드 타입
│   │                           # - 회원 타입
│   │                           # - 상담 타입
│   │                           # - 수정 요청 타입
│   └── match.ts                # Match 타입
│
├── constants/                  # 상수 & 열거형
│   └── enums.ts                # 한국어 레이블이 있는 열거형 옵션
│                               # - USER_STATUS_OPTIONS
│                               # - GENDER_OPTIONS
│                               # - 헬퍼 함수
│
├── fonts/                      # 커스텀 폰트
│   └── PretendardVariable.woff2
│
└── middleware.ts               # 라우트 보호 미들웨어
```

## 컴포넌트 구성

### 기능 기반 (권장)

관련 컴포넌트를 기능/도메인별로 그룹화:

```
components/
├── admin/              # Admin 대시보드 컴포넌트
│   ├── DashboardStats.tsx
│   ├── MemberTable.tsx
│   └── modals/         # 서브 기능 그룹화
├── auth/               # 인증 컴포넌트
├── match/              # 매칭 인터페이스 컴포넌트
├── sections/           # 랜딩 페이지 섹션
├── seo/                # SEO 관련 컴포넌트
├── layout/             # 앱 전체 레이아웃 컴포넌트
└── ui/                 # shadcn/ui 기본 컴포넌트
```

### shadcn/ui 컴포넌트

모든 shadcn/ui 컴포넌트는 `src/components/ui/`에 위치:

```
components/ui/
├── button.tsx          # variants가 있는 Button (CVA)
├── card.tsx            # CardHeader, CardContent 등이 있는 Card
├── input.tsx           # className 병합이 있는 Input
├── dialog.tsx          # DialogContent, DialogHeader 등이 있는 Dialog
├── select.tsx          # SelectTrigger, SelectContent 등이 있는 Select
└── ...
```

**ui/ 컴포넌트를 직접 수정하지 마세요.** 필요한 경우 래퍼 컴포넌트를 생성하세요.

## 파일 네이밍 컨벤션

| 타입 | 컨벤션 | 예시 |
|------|--------|------|
| 컴포넌트 | PascalCase | `MemberTable.tsx` |
| 유틸리티 | camelCase | `serverAuth.ts` |
| 타입 | camelCase | `admin.ts` |
| 상수 | camelCase | `enums.ts` |
| App 라우트 | lowercase | `page.tsx`, `layout.tsx` |
| API 라우트 | lowercase | `route.ts` |

## Import 패턴

### 절대 경로 Imports (권장)

```typescript
// 프로젝트 imports에 항상 @/ 사용
import { Button } from '@/components/ui/button';
import { api } from '@/lib/api';
import { useAuth } from '@/providers/AuthProvider';
import type { AdminMember } from '@/types/admin';
import { USER_STATUS_OPTIONS } from '@/constants/enums';
```

### 상대 경로 Imports (같은 디렉토리만)

```typescript
// 같은 디렉토리 또는 바로 아래 자식에만 사용
import { MemberRow } from './MemberRow';
import { BasicInfoEditModal } from './modals/BasicInfoEditModal';
```

### 타입 Imports

```typescript
// 타입 전용 imports에 'type' 키워드 사용
import type { AdminMember, MemberFilter } from '@/types/admin';
import type { Metadata } from 'next';
```

## 모범 사례

1. **기능 그룹화**: 관련 컴포넌트를 함께 유지
2. **일관된 네이밍**: 컴포넌트는 PascalCase, 유틸리티는 camelCase
3. **Index Exports**: 이 프로젝트에서는 사용하지 않음 (직접 imports 선호)
4. **관심사 분리**: Server Components, Client Components, 유틸리티 분리
5. **타입 배치**: 타입은 types/ 디렉토리에 위치
6. **상수 중앙화**: 모든 열거형은 constants/enums.ts에
7. **API 분리**: 메인 API는 api.ts에, admin은 adminApi.ts에
8. **모달 구성**: 관련 모달은 modals/ 서브디렉토리에 그룹화
