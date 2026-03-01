# 완전한 예제 - Next.js 15

## 완전한 Server Component 예제

```typescript
// app/artists/page.tsx
import { Suspense } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';
import { api } from '@/lib/api';
import { ArtistCard } from '@/components/artist/ArtistCard';
import type { Artist } from '@/types/artist';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Artists - Little-Boy',
  description: 'Browse our collection of talented artists',
};

export const revalidate = 60; // 60초마다 재검증

export default async function ArtistsPage() {
  // 서버에서 직접 데이터 패칭
  const artists: Artist[] = await api.artists.getAll();

  return (
    <div className="container py-8">
      <h1 className="text-3xl font-bold mb-8">Artists</h1>

      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
        {artists.map((artist) => (
          <ArtistCard key={artist.id} artist={artist} />
        ))}
      </div>
    </div>
  );
}
```

## 완전한 Client Component 예제

```typescript
// components/artist/ArtistFilters.tsx
'use client';

import { useState, useCallback } from 'react';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';

interface ArtistFiltersProps {
  onFilterChange: (filters: FilterState) => void;
}

interface FilterState {
  search: string;
  category: string;
  sortBy: string;
}

export function ArtistFilters({ onFilterChange }: ArtistFiltersProps) {
  const [filters, setFilters] = useState<FilterState>({
    search: '',
    category: 'all',
    sortBy: 'name',
  });

  const handleSearchChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const newFilters = { ...filters, search: e.target.value };
    setFilters(newFilters);
    onFilterChange(newFilters);
  }, [filters, onFilterChange]);

  const handleCategoryChange = useCallback((value: string) => {
    const newFilters = { ...filters, category: value };
    setFilters(newFilters);
    onFilterChange(newFilters);
  }, [filters, onFilterChange]);

  const handleReset = useCallback(() => {
    const defaultFilters = { search: '', category: 'all', sortBy: 'name' };
    setFilters(defaultFilters);
    onFilterChange(defaultFilters);
  }, [onFilterChange]);

  return (
    <div className="flex flex-col sm:flex-row gap-4 mb-6">
      <Input
        placeholder="아티스트 검색..."
        value={filters.search}
        onChange={handleSearchChange}
        className="flex-1"
      />

      <Select value={filters.category} onValueChange={handleCategoryChange}>
        <SelectTrigger className="w-full sm:w-[180px]">
          <SelectValue placeholder="카테고리" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="all">전체 카테고리</SelectItem>
          <SelectItem value="painting">회화</SelectItem>
          <SelectItem value="sculpture">조각</SelectItem>
          <SelectItem value="digital">디지털 아트</SelectItem>
        </SelectContent>
      </Select>

      <Button variant="outline" onClick={handleReset}>
        필터 초기화
      </Button>
    </div>
  );
}
```

## API Route 예제

```typescript
// app/api/artists/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import { getAuth } from '@/lib/serverAuth';

export async function GET(request: NextRequest) {
  try {
    // 검색 파라미터 가져오기
    const searchParams = request.nextUrl.searchParams;
    const page = parseInt(searchParams.get('page') || '1');
    const limit = parseInt(searchParams.get('limit') || '10');

    // 아티스트 데이터 패칭 (데이터베이스, 외부 API 등에서)
    const artists = await fetchArtistsFromDatabase({ page, limit });

    return NextResponse.json({
      success: true,
      data: artists,
      page,
      limit,
    });
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { success: false, error: '아티스트 목록을 불러오는데 실패했습니다' },
      { status: 500 }
    );
  }
}

export async function POST(request: NextRequest) {
  try {
    // 인증 확인
    const cookieStore = await cookies();
    const auth = await getAuth({ cookies: cookieStore });

    if (!auth.isAuthenticated) {
      return NextResponse.json(
        { success: false, error: 'Unauthorized' },
        { status: 401 }
      );
    }

    // 요청 본문 가져오기
    const body = await request.json();

    // 유효성 검사 및 아티스트 생성
    const newArtist = await createArtist(body);

    return NextResponse.json(
      { success: true, data: newArtist },
      { status: 201 }
    );
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { success: false, error: '아티스트 생성에 실패했습니다' },
      { status: 500 }
    );
  }
}
```

## react-hook-form + zod + shadcn/ui를 사용한 폼

```typescript
// components/artist/CreateArtistForm.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { api } from '@/lib/api';
import { useRouter } from 'next/navigation';
import { Loader2 } from 'lucide-react';

const artistSchema = z.object({
  name: z.string().min(2, '이름은 2자 이상이어야 합니다'),
  email: z.string().email('올바른 이메일 형식이 아닙니다'),
  bio: z.string().min(10, '소개는 10자 이상이어야 합니다'),
  category: z.string().min(1, '카테고리를 선택해주세요'),
});

type ArtistFormValues = z.infer<typeof artistSchema>;

export function CreateArtistForm() {
  const router = useRouter();
  const form = useForm<ArtistFormValues>({
    resolver: zodResolver(artistSchema),
    defaultValues: {
      name: '',
      email: '',
      bio: '',
      category: '',
    },
  });

  async function onSubmit(values: ArtistFormValues) {
    try {
      await api.artists.create(values);
      router.push('/artists');
    } catch (error) {
      console.error('아티스트 생성 실패:', error);
    }
  }

  return (
    <Card className="w-full max-w-2xl mx-auto">
      <CardHeader>
        <CardTitle>새 아티스트 생성</CardTitle>
      </CardHeader>
      <CardContent>
        <Form {...form}>
          <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
            <FormField
              control={form.control}
              name="name"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>이름</FormLabel>
                  <FormControl>
                    <Input placeholder="아티스트 이름" {...field} />
                  </FormControl>
                  <FormDescription>
                    아티스트의 표시 이름입니다.
                  </FormDescription>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="email"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>이메일</FormLabel>
                  <FormControl>
                    <Input type="email" placeholder="artist@example.com" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="category"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>카테고리</FormLabel>
                  <Select onValueChange={field.onChange} defaultValue={field.value}>
                    <FormControl>
                      <SelectTrigger>
                        <SelectValue placeholder="카테고리 선택" />
                      </SelectTrigger>
                    </FormControl>
                    <SelectContent>
                      <SelectItem value="painting">회화</SelectItem>
                      <SelectItem value="sculpture">조각</SelectItem>
                      <SelectItem value="digital">디지털 아트</SelectItem>
                      <SelectItem value="photography">사진</SelectItem>
                    </SelectContent>
                  </Select>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="bio"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>소개</FormLabel>
                  <FormControl>
                    <Textarea
                      placeholder="아티스트에 대해 소개해주세요..."
                      className="min-h-[120px]"
                      {...field}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <div className="flex gap-4">
              <Button
                type="button"
                variant="outline"
                onClick={() => router.back()}
              >
                취소
              </Button>
              <Button type="submit" disabled={form.formState.isSubmitting}>
                {form.formState.isSubmitting && (
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                )}
                아티스트 생성
              </Button>
            </div>
          </form>
        </Form>
      </CardContent>
    </Card>
  );
}
```

## Suspense 경계가 있는 페이지

```typescript
// app/artists/[id]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';
import { api } from '@/lib/api';
import { ArtworkList } from '@/components/artwork/ArtworkList';

interface PageProps {
  params: { id: string };
}

// 작품 목록 로딩 스켈레톤
function ArtworksSkeleton() {
  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
      {[...Array(6)].map((_, i) => (
        <Card key={i}>
          <Skeleton className="h-48 w-full rounded-t-lg" />
          <CardContent className="pt-4 space-y-2">
            <Skeleton className="h-4 w-3/4" />
            <Skeleton className="h-4 w-1/2" />
          </CardContent>
        </Card>
      ))}
    </div>
  );
}

// 작품 목록을 가져오는 async 컴포넌트
async function ArtistArtworks({ artistId }: { artistId: string }) {
  const artworks = await api.artworks.getByArtist(artistId);
  return <ArtworkList artworks={artworks} />;
}

export default async function ArtistDetailPage({ params }: PageProps) {
  const artist = await api.artists.getById(params.id);

  if (!artist) {
    notFound();
  }

  return (
    <div className="container py-8">
      <Card className="mb-8">
        <CardHeader>
          <CardTitle className="text-3xl">{artist.name}</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-muted-foreground">{artist.bio}</p>
        </CardContent>
      </Card>

      <h2 className="text-2xl font-bold mb-6">작품 목록</h2>

      <Suspense fallback={<ArtworksSkeleton />}>
        <ArtistArtworks artistId={params.id} />
      </Suspense>
    </div>
  );
}
```

## 완전한 기능 예제 구조

```
src/
  components/
    artist/
      ArtistCard.tsx          # 아티스트 표시용 카드 컴포넌트
      ArtistProfile.tsx       # 프로필을 위한 Server Component
      ArtistFilters.tsx       # 필터를 위한 Client Component
      CreateArtistForm.tsx    # react-hook-form을 사용한 폼

  app/
    artists/
      page.tsx                # Server Component - 목록 페이지
      [id]/
        page.tsx              # Server Component - 상세 페이지
      new/
        page.tsx              # 아티스트 생성 페이지
      loading.tsx             # 로딩 UI
      error.tsx               # 에러 UI

  types/
    artist.ts                 # TypeScript 타입 정의

  lib/
    api.ts                    # 아티스트 메서드가 있는 API 클라이언트
    utils.ts                  # cn() 유틸리티
```

## ArtistCard 컴포넌트

```typescript
// components/artist/ArtistCard.tsx
import Link from 'next/link';
import Image from 'next/image';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import type { Artist } from '@/types/artist';
import { cn } from '@/lib/utils';

interface ArtistCardProps {
  artist: Artist;
  className?: string;
}

export function ArtistCard({ artist, className }: ArtistCardProps) {
  return (
    <Link href={`/artists/${artist.id}`}>
      <Card className={cn(
        "overflow-hidden transition-all hover:shadow-lg hover:-translate-y-1",
        className
      )}>
        {artist.imageUrl && (
          <div className="aspect-square relative">
            <Image
              src={artist.imageUrl}
              alt={artist.name}
              fill
              className="object-cover"
            />
          </div>
        )}
        <CardHeader>
          <CardTitle className="flex items-center justify-between">
            {artist.name}
            <Badge variant="secondary">{artist.category}</Badge>
          </CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-sm text-muted-foreground line-clamp-2">
            {artist.bio}
          </p>
        </CardContent>
      </Card>
    </Link>
  );
}
```

## 로딩 UI

```typescript
// app/artists/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';
import { Card, CardContent, CardHeader } from '@/components/ui/card';

export default function Loading() {
  return (
    <div className="container py-8">
      <Skeleton className="h-10 w-48 mb-8" />

      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
        {[...Array(6)].map((_, i) => (
          <Card key={i}>
            <Skeleton className="aspect-square w-full" />
            <CardHeader>
              <Skeleton className="h-6 w-3/4" />
            </CardHeader>
            <CardContent>
              <Skeleton className="h-4 w-full" />
              <Skeleton className="h-4 w-2/3 mt-2" />
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

## 에러 UI

```typescript
// app/artists/error.tsx
'use client';

import { useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { AlertCircle } from 'lucide-react';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function Error({ error, reset }: ErrorProps) {
  useEffect(() => {
    console.error('Error:', error);
  }, [error]);

  return (
    <div className="container py-8 flex justify-center">
      <Card className="w-full max-w-md">
        <CardHeader>
          <CardTitle className="flex items-center gap-2 text-destructive">
            <AlertCircle className="h-5 w-5" />
            오류가 발생했습니다
          </CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <p className="text-muted-foreground">
            페이지를 불러오는 중 오류가 발생했습니다. 다시 시도해주세요.
          </p>
          <Button onClick={reset}>다시 시도</Button>
        </CardContent>
      </Card>
    </div>
  );
}
```

이 구조가 제공하는 것:
- 서버 사이드 데이터 패칭 및 렌더링
- 필요한 곳에서만 클라이언트 사이드 인터랙티비티
- 전반적인 타입 안전성
- 적절한 에러 및 로딩 상태 처리
- 관심사의 명확한 분리
- 일관된 디자인을 위한 shadcn/ui 컴포넌트
- 커스텀 스타일링을 위한 Tailwind CSS
