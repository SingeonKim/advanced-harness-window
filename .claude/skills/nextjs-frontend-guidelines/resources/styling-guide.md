# 스타일링 가이드 - shadcn/ui + Tailwind CSS 4

## 개요

프로젝트는 모던 스타일링 방식을 사용합니다:
- **shadcn/ui**: Tailwind 스타일링이 적용된 사전 빌드된 접근 가능한 컴포넌트
- **Tailwind CSS 4**: 유틸리티 우선 CSS 프레임워크
- **cn() 유틸리티**: clsx + tailwind-merge로 클래스 병합

---

## shadcn/ui 스타일링

### shadcn/ui란?

shadcn/ui는 전통적인 의미의 컴포넌트 라이브러리가 아닙니다. 프로젝트에 복사하여 붙여넣는 재사용 가능한 컴포넌트 모음입니다. 즉:

- 컴포넌트는 YOUR 코드베이스에 위치 (`src/components/ui/`)
- 코드를 소유하며 자유롭게 커스터마이즈 가능
- 컴포넌트에 대한 외부 npm 패키지 의존성 없음
- 접근성을 위한 Radix UI 프리미티브 기반
- Tailwind CSS로 스타일링

### shadcn/ui 설치

```bash
# 프로젝트에 shadcn/ui 초기화
npx shadcn@latest init

# 개별 컴포넌트 추가
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add input
npx shadcn@latest add dialog
npx shadcn@latest add form
npx shadcn@latest add select
npx shadcn@latest add skeleton
```

### 컴포넌트 위치

모든 shadcn/ui 컴포넌트는 `src/components/ui/`에 위치:

```
src/components/ui/
  button.tsx
  card.tsx
  input.tsx
  dialog.tsx
  form.tsx
  select.tsx
  skeleton.tsx
  alert.tsx
  badge.tsx
  ...
```

---

## cn() 유틸리티 (핵심)

`cn()` 유틸리티는 shadcn/ui에 필수입니다. 클래스 이름을 지능적으로 병합합니다.

### 설정

```typescript
// lib/utils.ts
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### 사용법

```typescript
import { cn } from '@/lib/utils';

// 기본 사용
<div className={cn("flex items-center", className)}>

// 조건부 클래스
<div className={cn(
  "flex items-center gap-2 p-4",
  isActive && "bg-primary text-primary-foreground",
  isDisabled && "opacity-50 cursor-not-allowed"
)}>

// Variants 조합
<Button className={cn(
  "w-full",
  size === "lg" && "h-12 text-lg"
)}>
```

### cn()이 중요한 이유

```typescript
// cn() 없이 - 클래스가 충돌할 수 있음
<div className={`p-4 ${className}`}>  // className에 p-2가 있으면 둘 다 적용

// cn() 사용 - 나중 클래스가 이전 클래스를 덮어씀
<div className={cn("p-4", className)}>  // className의 p-2가 우선
```

---

## shadcn/ui 컴포넌트 Variants

### Button Variants

```typescript
import { Button } from '@/components/ui/button';

// 기본 (primary)
<Button>클릭</Button>

// Variants
<Button variant="default">Primary</Button>
<Button variant="secondary">Secondary</Button>
<Button variant="destructive">삭제</Button>
<Button variant="outline">Outline</Button>
<Button variant="ghost">Ghost</Button>
<Button variant="link">Link</Button>

// Sizes
<Button size="default">Default</Button>
<Button size="sm">Small</Button>
<Button size="lg">Large</Button>
<Button size="icon"><Icon /></Button>

// 커스텀 스타일링
<Button className="w-full bg-blue-600 hover:bg-blue-700">
  커스텀 버튼
</Button>
```

### Card 컴포넌트

```typescript
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from '@/components/ui/card';

<Card className="w-[350px]">
  <CardHeader>
    <CardTitle>카드 제목</CardTitle>
    <CardDescription>카드 설명이 여기에 들어갑니다</CardDescription>
  </CardHeader>
  <CardContent>
    <p>카드 내용</p>
  </CardContent>
  <CardFooter>
    <Button>액션</Button>
  </CardFooter>
</Card>
```

### Input 컴포넌트

```typescript
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

<div className="grid w-full max-w-sm items-center gap-1.5">
  <Label htmlFor="email">이메일</Label>
  <Input type="email" id="email" placeholder="이메일" />
</div>
```

### Dialog 컴포넌트

```typescript
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
  DialogFooter,
} from '@/components/ui/dialog';

<Dialog>
  <DialogTrigger asChild>
    <Button variant="outline">다이얼로그 열기</Button>
  </DialogTrigger>
  <DialogContent className="sm:max-w-[425px]">
    <DialogHeader>
      <DialogTitle>다이얼로그 제목</DialogTitle>
      <DialogDescription>
        이 다이얼로그의 기능 설명.
      </DialogDescription>
    </DialogHeader>
    <div className="py-4">
      {/* 다이얼로그 내용 */}
    </div>
    <DialogFooter>
      <Button type="submit">저장</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Select 컴포넌트

```typescript
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';

<Select>
  <SelectTrigger className="w-[180px]">
    <SelectValue placeholder="옵션 선택" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="option1">옵션 1</SelectItem>
    <SelectItem value="option2">옵션 2</SelectItem>
    <SelectItem value="option3">옵션 3</SelectItem>
  </SelectContent>
</Select>
```

---

## Tailwind CSS 4

### 유틸리티 클래스

```typescript
export function Component() {
  return (
    <div className="flex items-center gap-4 p-4 bg-background rounded-lg border">
      <img
        src="..."
        alt="..."
        className="w-16 h-16 rounded-full object-cover"
      />
      <div className="flex-1">
        <h3 className="text-lg font-bold text-foreground">제목</h3>
        <p className="text-sm text-muted-foreground">설명</p>
      </div>
    </div>
  );
}
```

### 반응형 클래스

```typescript
<div className="
  flex flex-col       /* 모바일: 세로 레이아웃 */
  md:flex-row         /* 태블릿+: 가로 레이아웃 */
  gap-4 md:gap-6      /* 다른 간격 */
  p-4 md:p-8          /* 다른 패딩 */
">
  내용
</div>
```

### shadcn/ui 색상 토큰

shadcn/ui는 테마를 위한 CSS 커스텀 속성을 사용합니다:

```typescript
// Primary 색상
<div className="bg-primary text-primary-foreground">Primary</div>

// Secondary 색상
<div className="bg-secondary text-secondary-foreground">Secondary</div>

// 흐린/부드러운
<div className="bg-muted text-muted-foreground">Muted</div>

// Accent
<div className="bg-accent text-accent-foreground">Accent</div>

// Destructive (에러, 삭제 액션)
<div className="bg-destructive text-destructive-foreground">Destructive</div>

// 배경 및 전경
<div className="bg-background text-foreground">Default</div>

// 테두리와 입력
<div className="border border-border">테두리</div>
<Input className="border-input" />

// Card
<div className="bg-card text-card-foreground">Card</div>
```

---

## 언제 무엇을 사용할지

### shadcn/ui 컴포넌트 사용 시:
- UI 요소 구성 (버튼, 카드, 다이얼로그, 폼)
- 접근 가능하고 잘 설계된 컴포넌트 필요 시
- 앱 전체에서 일관된 디자인 필요 시
- 인터랙티브 요소 필요 시 (드롭다운, 모달)

### Tailwind 클래스 사용 시:
- 레이아웃 (flex, grid, 위치 지정)
- 간격 (padding, margin, gap)
- 타이포그래피 (폰트 크기, 굵기, 색상)
- shadcn/ui 컴포넌트 커스터마이즈
- 일회성 스타일링 필요 시

### 둘 다 조합:
```typescript
import { Button } from '@/components/ui/button';
import { Card, CardContent } from '@/components/ui/card';

export function FeatureCard() {
  return (
    {/* 레이아웃을 위한 Tailwind */}
    <div className="flex flex-col gap-4 p-6">
      {/* 컴포넌트를 위한 shadcn/ui */}
      <Card>
        <CardContent className="pt-6">
          {/* 커스텀 Tailwind 스타일링 */}
          <h2 className="text-2xl font-bold mb-4">기능</h2>
          <p className="text-muted-foreground mb-4">설명</p>
          <Button className="w-full">더 알아보기</Button>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## 공통 패턴

### 반응형 카드 그리드

```typescript
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

export function CardGrid({ items }) {
  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
      {items.map((item) => (
        <Card key={item.id}>
          <CardHeader>
            <CardTitle>{item.title}</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-muted-foreground">{item.description}</p>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

### 폼 레이아웃

```typescript
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export function Form() {
  return (
    <form className="flex flex-col gap-4 max-w-md mx-auto">
      <div className="space-y-2">
        <Label htmlFor="name">이름</Label>
        <Input id="name" placeholder="이름을 입력하세요" />
      </div>
      <div className="space-y-2">
        <Label htmlFor="email">이메일</Label>
        <Input id="email" type="email" placeholder="이메일을 입력하세요" />
      </div>
      <Button type="submit" className="mt-4">
        제출
      </Button>
    </form>
  );
}
```

### 반응형 사이드바 레이아웃

```typescript
export function Layout({ children }) {
  return (
    <div className="flex min-h-screen">
      {/* 사이드바 - 모바일에서 숨김 */}
      <aside className="hidden md:flex w-64 flex-col border-r bg-muted/40">
        {/* 사이드바 내용 */}
      </aside>

      {/* 메인 콘텐츠 */}
      <main className="flex-1 p-4 md:p-8">
        {children}
      </main>
    </div>
  );
}
```

### Skeleton을 사용한 로딩 상태

```typescript
import { Skeleton } from '@/components/ui/skeleton';
import { Card, CardContent, CardHeader } from '@/components/ui/card';

export function CardSkeleton() {
  return (
    <Card>
      <CardHeader>
        <Skeleton className="h-6 w-[200px]" />
      </CardHeader>
      <CardContent className="space-y-2">
        <Skeleton className="h-4 w-full" />
        <Skeleton className="h-4 w-[80%]" />
      </CardContent>
    </Card>
  );
}
```

---

## 다크 모드 지원

shadcn/ui는 CSS 커스텀 속성을 통해 내장 다크 모드를 지원합니다:

```typescript
// globals.css에서
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    /* ... 기타 라이트 모드 변수 */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... 기타 다크 모드 변수 */
  }
}
```

컴포넌트는 현재 테마에 자동으로 적응합니다.

---

## 모범 사례

1. **항상 cn() 사용**: 조건부 또는 병합된 클래스 이름에
2. **시맨틱 토큰 사용**: `bg-blue-500` 대신 `bg-primary`
3. **컴포넌트 우선**: 순수 HTML 대신 shadcn/ui 컴포넌트 선호
4. **className으로 커스터마이즈**: shadcn/ui 컴포넌트에 Tailwind 클래스 추가
5. **반응형 디자인**: Tailwind 반응형 접두사 사용 (sm:, md:, lg:)
6. **컴포넌트는 ui/에**: 모든 shadcn/ui 컴포넌트는 `src/components/ui/`에
7. **ui/ 직접 수정 금지**: 다른 기본값이 필요하면 래퍼 컴포넌트 생성
8. **적절한 간격 토큰 사용**: 일관성을 위해 gap-4, p-4, mb-4 등 사용
