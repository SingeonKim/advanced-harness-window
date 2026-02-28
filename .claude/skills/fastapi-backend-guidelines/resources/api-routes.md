# API 라우트 & 라우터 - FastAPI

## 라우터 기초

FastAPI 라우터는 도메인별로 API 엔드포인트를 구성합니다.

### 라우터 생성

```python
# backend/api/v1/routers/admin.py
from fastapi import APIRouter, Depends, status, Query, HTTPException
from sqlmodel.ext.asyncio.session import AsyncSession
from typing import List, Optional

from backend.db.orm import get_read_session_dependency, get_write_session_dependency
from backend.domain.admin.service import AdminService
from backend.dtos.admin import (
    DashboardStatsResponse,
    MemberListResponse,
    MemberDetailResponse,
    AdminBasicInfoUpdateRequest,
)
from backend.error import NotFoundError

router = APIRouter(
    prefix="/api/v1/admin",
    tags=["admin"],  # OpenAPI 문서용
)
```

### 읽기 작업 (GET)

```python
# 대시보드 통계 - 병렬 쿼리
@router.get("/dashboard/stats", response_model=DashboardStatsResponse)
async def get_dashboard_stats(
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """대시보드 통계 조회"""
    service = AdminService(session)
    return await service.get_dashboard_stats()


# 페이지네이션 및 필터가 있는 목록
@router.get("/members", response_model=MemberListResponse)
async def list_members(
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),
    keyword: Optional[str] = Query(default=None, min_length=1),
    status: Optional[str] = Query(default=None),
    gender: Optional[str] = Query(default=None),
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """페이지네이션 및 필터가 있는 회원 목록"""
    service = AdminService(session)
    return await service.list_members(
        page=page,
        page_size=page_size,
        keyword=keyword,
        status=status,
        gender=gender,
    )


# ID로 조회
@router.get("/members/{user_id}", response_model=MemberDetailResponse)
async def get_member(
    user_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """회원 상세 조회"""
    service = AdminService(session)
    try:
        return await service.get_member_detail(user_id)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

### 쓰기 작업 (POST, PATCH, DELETE)

```python
# 생성
@router.post("/consultations", response_model=ConsultationResponse, status_code=status.HTTP_201_CREATED)
async def create_consultation(
    dto: ConsultationCreateRequest,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """상담 일정 등록"""
    service = AdminService(session)
    return await service.create_consultation(dto)


# 수정 (부분 업데이트에 PATCH 사용)
@router.patch("/members/{user_id}/basic", response_model=MemberDetailResponse)
async def update_member_basic_info(
    user_id: str,
    dto: AdminBasicInfoUpdateRequest,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """회원 기본 정보 수정"""
    service = AdminService(session)
    try:
        return await service.update_member_basic_info(user_id, dto)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


# 삭제 (소프트 삭제)
@router.delete("/members/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_member(
    user_id: str,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """회원 소프트 삭제"""
    service = AdminService(session)
    result = await service.soft_delete_member(user_id)
    if not result:
        raise HTTPException(status_code=404, detail=f"Member {user_id} not found")
```

## 읽기/쓰기 세션 분리

**중요**: 올바른 세션 의존성을 사용하세요:

```python
# 읽기 작업 (SELECT)
session: AsyncSession = Depends(get_read_session_dependency)

# 쓰기 작업 (INSERT, UPDATE, DELETE)
session: AsyncSession = Depends(get_write_session_dependency)
```

## Query 파라미터

```python
from fastapi import Query
from typing import Optional, List

@router.get("/members")
async def list_members(
    # 필수
    status: str = Query(..., description="상태 필터"),

    # 기본값이 있는 선택적
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),

    # 선택적 nullable
    keyword: Optional[str] = Query(default=None, min_length=1),
    gender: Optional[str] = Query(default=None),

    # 다중 값
    statuses: Optional[List[str]] = Query(default=None),

    session: AsyncSession = Depends(get_read_session_dependency),
):
    pass
```

## Path 파라미터

```python
@router.get("/members/{user_id}/photos/{photo_id}")
async def get_photo(
    user_id: str,
    photo_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """사용자의 특정 사진 조회"""
    service = UserService(session)
    return await service.get_photo(user_id, photo_id)
```

## 요청 본문 (DTO)

```python
from pydantic import BaseModel, Field, field_validator

class AdminBasicInfoUpdateRequest(BaseModel):
    name: Optional[str] = Field(None, max_length=50)
    status: Optional[str] = None

    model_config = {"extra": "forbid"}  # 알 수 없는 필드 거부

    @field_validator("status")
    @classmethod
    def validate_status(cls, v):
        if v is not None:
            valid = ["pending", "approved", "rejected"]
            if v not in valid:
                raise ValueError(f"Invalid status: {v}")
        return v


@router.patch("/members/{user_id}/basic")
async def update_member(
    user_id: str,
    dto: AdminBasicInfoUpdateRequest,  # 요청 본문 자동 유효성 검사
    session: AsyncSession = Depends(get_write_session_dependency),
):
    service = AdminService(session)
    return await service.update_member_basic_info(user_id, dto)
```

## 상태 코드

```python
from fastapi import status

# 성공 코드
@router.post("/", status_code=status.HTTP_201_CREATED)  # 생성됨
@router.delete("/", status_code=status.HTTP_204_NO_CONTENT)  # 콘텐츠 없음
@router.get("/", status_code=status.HTTP_200_OK)  # 성공 (기본값)

# 오류 코드 (HTTPException으로 발생)
from fastapi import HTTPException

if not item:
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Item not found"
    )
```

## 공개 vs 관리자 엔드포인트

```python
# backend/api/v1/routers/match.py

# 공개 엔드포인트 - 인증 불필요
@router.get("/status", response_model=MatchingWindowStatusResponse)
async def get_matching_status():
    """매칭 창 상태를 위한 공개 엔드포인트"""
    info = get_matching_window_info()
    return MatchingWindowStatusResponse(
        is_open=info["is_open"],
        next_matching_date_str=info["next_matching_date_str"],
        message=info["message"],
    )


# 전화번호 인증이 있는 공개 엔드포인트
@router.get("/my-matches", response_model=MatchCardListResponse)
async def get_my_matches(
    phone: str = Query(..., description="인증을 위한 전화번호"),
    bypass_window: bool = Query(False, description="관리자 우회"),
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """전화번호로 매치 카드 조회"""
    # 전화번호 형식 유효성 검사
    normalized_phone = "".join(c for c in phone if c.isdigit())
    if not re.match(r"^01[0-9]\d{7,8}$", normalized_phone):
        raise HTTPException(status_code=400, detail="Invalid phone format")

    service = MatchService(session)
    return await service.get_match_cards_by_phone(normalized_phone, bypass_window)


# 관리자 엔드포인트 - 인증 필요
@router.get("/members/{user_id}", response_model=MemberDetailResponse)
async def get_member(
    user_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
    # current_user: User = Depends(require_admin),  # 관리자 인증
):
    """관리자: 회원 상세 조회"""
    service = AdminService(session)
    return await service.get_member_detail(user_id)
```

## 라우터 등록

```python
# backend/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from backend.middleware.error_handler import ErrorHandlerMiddleware

from backend.api.v1.routers.auth import router as auth_router
from backend.api.v1.routers.user import router as user_router
from backend.api.v1.routers.admin import router as admin_router
from backend.api.v1.routers.match import router as match_router
from backend.api.v1.routers.upload import router as upload_router
from backend.api.v1.routers.health import router as health_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # 시작
    yield
    # 종료


def create_application() -> FastAPI:
    app = FastAPI(
        title="YGS API",
        description="YGS 매칭 플랫폼 API",
        version="1.0.0",
        lifespan=lifespan,
    )

    # 미들웨어 추가
    app.add_middleware(ErrorHandlerMiddleware)

    # 라우터 등록
    app.include_router(health_router)
    app.include_router(auth_router)
    app.include_router(user_router)
    app.include_router(admin_router)
    app.include_router(match_router)
    app.include_router(upload_router)

    return app


app = create_application()
```

## YGS API 라우트 개요

| 라우터 | 접두사 | 설명 |
|--------|--------|------|
| `auth.py` | `/api/v1/auth` | 로그인, 회원가입, OAuth, 토큰 갱신 |
| `user.py` | `/api/v1/users` | 사용자 프로필, 사진, 문서 |
| `admin.py` | `/api/v1/admin` | 대시보드, 회원 관리 |
| `match.py` | `/api/v1/matches` | 매치 주, 이력, 카드 |
| `upload.py` | `/api/v1/upload` | S3 presigned URL 생성 |
| `health.py` | `/api/v1/health` | 헬스 체크 |

## 모범 사례

1. **접두사**: 모든 라우트에 `/api/v1/{domain}` 사용
2. **태그**: 문서를 위해 관련 엔드포인트를 태그로 그룹화
3. **응답 모델**: 항상 `response_model` 명시
4. **상태 코드**: 적절한 HTTP 상태 코드 사용
5. **유효성 검사**: Pydantic Query/Path 유효성 검사기 사용
6. **세션 의존성**: 읽기/쓰기 세션 분리
7. **Docstring**: 각 엔드포인트 문서화
8. **비동기**: 모든 라우트 핸들러는 async여야 함
9. **에러 처리**: HTTPException을 사용한 try/except
10. **Extra forbid**: DTO에서 알 수 없는 필드 거부
