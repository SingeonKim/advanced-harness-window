# 계층형 아키텍처 - FastAPI

## 개요

YGS 백엔드는 3계층 아키텍처 패턴을 따릅니다:

**Router → Service → Repository**

각 계층은 특정 책임이 있으며 우회해서는 안 됩니다.

---

## 3계층

### 1. Router 레이어 (API/프레젠테이션)

**위치**: `backend/api/v1/routers/`

**책임**:
- HTTP 요청/응답 처리
- 요청 유효성 검사 (Pydantic DTO를 통해)
- 응답 포맷팅
- HTTP 상태 코드
- 인증/인가 확인
- 서비스 레이어 호출

**하지 않는 것**:
- 비즈니스 로직
- 직접 데이터베이스 접근
- 복잡한 데이터 변환

**예시**:
```python
# backend/api/v1/routers/admin.py
from fastapi import APIRouter, Depends, status, HTTPException
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.db.orm import get_read_session_dependency, get_write_session_dependency
from backend.domain.admin.service import AdminService
from backend.dtos.admin import MemberDetailResponse, AdminBasicInfoUpdateRequest
from backend.error import NotFoundError

router = APIRouter(prefix="/api/v1/admin", tags=["admin"])

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
```

### 2. Service 레이어 (비즈니스 로직)

**위치**: `backend/domain/{domain}/service.py`

**책임**:
- 비즈니스 로직 구현
- 도메인 규칙 적용
- 트랜잭션 오케스트레이션
- 데이터를 위해 레포지토리 호출
- 데이터 변환 (model → DTO)
- 도메인 예외로 에러 처리
- UserDataLoader로 N+1 방지
- S3 자산을 위한 presigned URL 생성

**하지 않는 것**:
- HTTP 관련 사항 (상태 코드, 헤더)
- 직접 SQL 쿼리
- 데이터베이스 세션 관리

**예시**:
```python
# backend/domain/admin/service.py
import asyncio
from typing import List
from datetime import datetime, timedelta
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.domain.user.repository import UserRepository, UserDataLoader
from backend.dtos.admin import DashboardStatsResponse, MemberDetailResponse, AdminBasicInfoUpdateRequest
from backend.error import NotFoundError
from backend.utils.s3 import generate_presigned_url

class AdminService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._user_repository = UserRepository(session)
        self._data_loader = UserDataLoader(session)

    async def get_dashboard_stats(self) -> DashboardStatsResponse:
        """병렬 쿼리로 대시보드 통계 조회"""
        now = datetime.utcnow()
        week_ago = now - timedelta(days=7)

        # 모든 쿼리를 병렬로 실행
        total, weekly, male, female = await asyncio.gather(
            self._user_repository.count_all(),
            self._user_repository.count_since(week_ago),
            self._user_repository.count_by_gender("male"),
            self._user_repository.count_by_gender("female"),
        )

        return DashboardStatsResponse(
            total_members=total,
            weekly_signups=weekly,
            male_count=male,
            female_count=female,
        )

    async def get_member_detail(self, user_id: str) -> MemberDetailResponse:
        """관계를 포함한 회원 전체 상세 조회"""
        user_with_relations = await self._data_loader.load_user_with_relations(
            user_id,
            load_profile=True,
            load_photos=True,
        )

        if not user_with_relations:
            raise NotFoundError(f"User {user_id} not found")

        # 사진의 presigned URL 생성
        photo_urls = []
        if user_with_relations.photos:
            photo_urls = await asyncio.gather(*[
                generate_presigned_url(photo.s3_key)
                for photo in user_with_relations.photos
            ])

        return MemberDetailResponse.from_user_with_relations(
            user_with_relations,
            photo_urls=photo_urls,
        )

    async def update_member_basic_info(
        self,
        user_id: str,
        dto: AdminBasicInfoUpdateRequest,
    ) -> MemberDetailResponse:
        """회원 기본 정보 수정"""
        user = await self._user_repository.get_by_id(user_id)
        if not user:
            raise NotFoundError(f"User {user_id} not found")

        # 업데이트 적용
        update_data = dto.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            if hasattr(user, field):
                setattr(user, field, value)

        user.updated_at = datetime.utcnow()
        await self._user_repository.update(user)

        return await self.get_member_detail(user_id)
```

### 3. Repository 레이어 (데이터 접근)

**위치**: `backend/domain/{domain}/repository.py`

**책임**:
- 데이터베이스 쿼리 (SELECT, INSERT, UPDATE, DELETE)
- 데이터 조회 및 영속화
- 쿼리 최적화
- 도메인 모델 반환
- 소프트 삭제 필터링

**하지 않는 것**:
- 비즈니스 로직
- DTO (도메인 모델만 작업)
- HTTP 관련 사항
- 데이터베이스 제약 이상의 유효성 검사

**예시**:
```python
# backend/domain/user/repository.py
from typing import List, Optional, Tuple
from sqlmodel import select, or_, func
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.domain.user.model import User
from backend.domain.shared.base_repository import BaseRepository

class UserRepository(BaseRepository[User]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, User)

    async def find_by_phone(self, phone: str) -> Optional[User]:
        """전화번호로 사용자 조회"""
        stmt = select(User).where(
            User.phone == phone,
            User.deleted_at.is_(None)
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def search_members(
        self,
        keyword: Optional[str] = None,
        status: Optional[str] = None,
        limit: int = 20,
        offset: int = 0
    ) -> Tuple[List[User], int]:
        """필터가 있는 회원 검색"""
        stmt = select(User).where(User.deleted_at.is_(None))

        if keyword:
            stmt = stmt.where(
                or_(
                    User.name.ilike(f"%{keyword}%"),
                    User.phone.ilike(f"%{keyword}%")
                )
            )
        if status:
            stmt = stmt.where(User.status == status)

        # 전체 개수
        count_stmt = select(func.count()).select_from(stmt.subquery())
        count_result = await self.session.execute(count_stmt)
        total = count_result.scalar()

        # 페이지네이션 적용
        stmt = stmt.order_by(User.created_at.desc()).offset(offset).limit(limit)
        result = await self.session.execute(stmt)
        users = list(result.scalars().all())

        return users, total

    async def count_all(self) -> int:
        """모든 활성 사용자 수"""
        stmt = select(func.count(User.id)).where(User.deleted_at.is_(None))
        result = await self.session.execute(stmt)
        return result.scalar()

    async def count_by_gender(self, gender: str) -> int:
        """성별로 사용자 수"""
        stmt = select(func.count(User.id)).where(
            User.gender == gender,
            User.deleted_at.is_(None)
        )
        result = await self.session.execute(stmt)
        return result.scalar()
```

---

## 데이터 흐름

### 읽기 작업 흐름

```
1. 클라이언트 요청
   ↓
2. 라우터가 요청 유효성 검사 (Pydantic)
   ↓
3. 라우터가 서비스 호출
   ↓
4. 서비스가 레포지토리 / UserDataLoader 호출
   ↓
5. 레포지토리가 데이터베이스 쿼리 (소프트 삭제 필터 포함)
   ↓
6. 레포지토리가 Model(s) 반환
   ↓
7. 서비스가 비즈니스 로직 적용
   ↓
8. 서비스가 S3 자산을 위한 presigned URL 생성
   ↓
9. 서비스가 Model → DTO 변환
   ↓
10. 라우터가 HTTP 응답 반환
```

### 쓰기 작업 흐름

```
1. DTO가 있는 클라이언트 요청
   ↓
2. 라우터가 DTO 유효성 검사 (Pydantic + field_validator)
   ↓
3. 라우터가 DTO와 함께 서비스 호출
   ↓
4. 서비스가 비즈니스 규칙 적용
   ↓
5. 서비스가 DTO → Model 변환
   ↓
6. 서비스가 Model과 함께 레포지토리 호출
   ↓
7. 레포지토리가 데이터베이스에 영속화
   ↓
8. 레포지토리가 저장된 Model 반환
   ↓
9. 서비스가 Model → DTO 변환
   ↓
10. 라우터가 HTTP 응답 반환 (201 Created)
```

---

## 왜 이 아키텍처인가?

### 관심사 분리
- 각 레이어는 명확한 책임을 가짐
- 각 레이어를 독립적으로 테스트하기 쉬움
- 한 레이어의 변경이 다른 레이어에 영향을 주지 않음

### 유지보수성
- 서비스에 중앙화된 비즈니스 로직
- 레포지토리에 데이터 접근 로직
- 라우터에 API 계약

### 테스트 가능성
- 라우터 테스트에서 서비스 모킹
- 서비스 테스트에서 레포지토리 모킹
- 데이터베이스에 대한 레포지토리 테스트

### 유연성
- 데이터베이스 구현 교체 (레포지토리)
- 비즈니스 규칙 변경 (서비스)
- API 계약 수정 (라우터)

---

## YGS 특화 패턴

### 읽기/쓰기 세션 분리

```python
# 읽기 작업 (GET)
session: AsyncSession = Depends(get_read_session_dependency)

# 쓰기 작업 (POST/PATCH/DELETE)
session: AsyncSession = Depends(get_write_session_dependency)
```

### N+1 방지를 위한 UserDataLoader

```python
# 서비스에서 - 병렬로 관계가 있는 사용자 로드
user_with_relations = await self._data_loader.load_user_with_relations(
    user_id,
    load_profile=True,
    load_photos=True,
    load_subscription=True,
)
```

### asyncio.gather를 사용한 병렬 쿼리

```python
# 대시보드 통계 - 모든 쿼리가 병렬로 실행
total, weekly, male, female = await asyncio.gather(
    self._repository.count_all(),
    self._repository.count_since(week_ago),
    self._repository.count_by_gender("male"),
    self._repository.count_by_gender("female"),
)
```

### 소프트 삭제 패턴

```python
# 레포지토리는 항상 소프트 삭제된 레코드를 필터링
stmt = select(User).where(
    User.id == user_id,
    User.deleted_at.is_(None)  # 소프트 삭제된 것 제외
)
```

---

## 모범 사례

1. **레이어를 절대 우회하지 않음**: 항상 Router → Service → Repository
2. **서비스가 비즈니스 로직 소유**: 라우터나 레포지토리에 로직 넣지 않음
3. **레포지토리는 모델 반환**: DTO는 API 레이어 전용
4. **의존성 주입 사용**: 세션에 FastAPI의 Depends() 사용
5. **전면적 비동기**: 모든 레이어에서 async/await 사용
6. **읽기/쓰기 분리**: 적절한 세션 의존성 사용
7. **요청당 하나의 서비스**: 각 라우트에서 서비스 인스턴스 생성
8. **에러 처리**: 서비스에서 도메인 예외 발생, 라우터/미들웨어에서 처리
9. **UserDataLoader**: 여러 관계가 있는 사용자 로드 시
10. **asyncio.gather**: 병렬 독립 쿼리에
11. **소프트 삭제**: 항상 `deleted_at.is_(None)` 필터
12. **ULID ID**: 읽기 쉬운 ID를 위한 엔티티 접두사 사용
