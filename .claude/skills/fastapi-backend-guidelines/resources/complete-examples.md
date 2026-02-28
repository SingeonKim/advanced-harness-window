# 완전한 예시 - FastAPI

## 전체 사용자 도메인 예시

### 모델

```python
# backend/domain/user/model.py
from sqlmodel import SQLModel, Field, Relationship
from typing import Optional, List
from datetime import datetime
from ulid import ULID


def generate_user_id() -> str:
    return f"usr_{ULID()}"


def generate_photo_id() -> str:
    return f"pho_{ULID()}"


class User(SQLModel, table=True):
    __tablename__ = "users"

    id: str = Field(default_factory=generate_user_id, primary_key=True)
    firebase_id: Optional[str] = Field(default=None, index=True)
    kakao_id: Optional[str] = Field(default=None, index=True)
    email: Optional[str] = Field(default=None, index=True)
    phone: str = Field(unique=True, index=True)
    name: str = Field(max_length=50)
    gender: str = Field(index=True)
    birth_year: int
    status: str = Field(default="pending", index=True)
    is_admin: bool = Field(default=False)

    # 타임스탬프
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: Optional[datetime] = None
    deleted_at: Optional[datetime] = None  # 소프트 삭제

    # 관계 (일대일)
    profile: Optional["UserProfile"] = Relationship(back_populates="user")
    photos: List["UserPhoto"] = Relationship(back_populates="user")


class UserProfile(SQLModel, table=True):
    __tablename__ = "user_profiles"

    id: str = Field(primary_key=True)
    user_id: str = Field(foreign_key="users.id", unique=True)
    height: Optional[int] = None
    education: Optional[str] = None
    university: Optional[str] = None
    job: Optional[str] = None
    salary_range: Optional[str] = None
    district: Optional[str] = None
    mbti: Optional[str] = None
    about_me: Optional[str] = None

    user: Optional[User] = Relationship(back_populates="profile")


class UserPhoto(SQLModel, table=True):
    __tablename__ = "user_photos"

    id: str = Field(default_factory=generate_photo_id, primary_key=True)
    user_id: str = Field(foreign_key="users.id", index=True)
    s3_key: str  # S3 키만 저장, URL 저장 금지
    thumbnail_s3_key: Optional[str] = None
    display_order: int = Field(default=0)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    deleted_at: Optional[datetime] = None

    user: Optional[User] = Relationship(back_populates="photos")
```

### 레포지토리

```python
# backend/domain/user/repository.py
from typing import List, Optional, Tuple
from sqlmodel import select, or_, func
from sqlmodel.ext.asyncio.session import AsyncSession
from datetime import datetime
import asyncio
from dataclasses import dataclass

from backend.domain.user.model import User, UserProfile, UserPhoto
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

    async def find_by_firebase_id(self, firebase_id: str) -> Optional[User]:
        """Firebase ID로 사용자 조회"""
        stmt = select(User).where(
            User.firebase_id == firebase_id,
            User.deleted_at.is_(None)
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def search_members(
        self,
        keyword: Optional[str] = None,
        status: Optional[str] = None,
        gender: Optional[str] = None,
        limit: int = 20,
        offset: int = 0
    ) -> Tuple[List[User], int]:
        """필터 및 페이지네이션이 있는 회원 검색"""
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
        if gender:
            stmt = stmt.where(User.gender == gender)

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


@dataclass
class UserWithRelations:
    """관계가 로드된 사용자 컨테이너"""
    user: User
    profile: Optional[UserProfile] = None
    photos: List[UserPhoto] = None


class UserDataLoader:
    """N+1 쿼리를 방지하는 병렬 로더"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def load_user_with_relations(
        self,
        user_id: str,
        load_profile: bool = False,
        load_photos: bool = False,
    ) -> Optional[UserWithRelations]:
        """선택적 관계를 병렬로 사용자 로드"""
        queries = []
        query_names = []

        async def load_user():
            stmt = select(User).where(User.id == user_id, User.deleted_at.is_(None))
            result = await self.session.execute(stmt)
            return result.scalar_one_or_none()

        queries.append(load_user())
        query_names.append("user")

        if load_profile:
            async def load_profile_fn():
                stmt = select(UserProfile).where(UserProfile.user_id == user_id)
                result = await self.session.execute(stmt)
                return result.scalar_one_or_none()
            queries.append(load_profile_fn())
            query_names.append("profile")

        if load_photos:
            async def load_photos_fn():
                stmt = select(UserPhoto).where(
                    UserPhoto.user_id == user_id,
                    UserPhoto.deleted_at.is_(None)
                ).order_by(UserPhoto.display_order)
                result = await self.session.execute(stmt)
                return list(result.scalars().all())
            queries.append(load_photos_fn())
            query_names.append("photos")

        results = await asyncio.gather(*queries)
        result_dict = dict(zip(query_names, results))

        user = result_dict.get("user")
        if not user:
            return None

        return UserWithRelations(
            user=user,
            profile=result_dict.get("profile"),
            photos=result_dict.get("photos", []),
        )
```

### 서비스

```python
# backend/domain/user/service.py
from typing import List, Optional
from datetime import datetime
import asyncio
from sqlmodel.ext.asyncio.session import AsyncSession

from backend.domain.user.repository import UserRepository, UserDataLoader
from backend.domain.user.model import User, UserProfile
from backend.dtos.user import UserCreateDto, UserResponseDto, MemberDetailResponse
from backend.error import NotFoundError, ConflictError
from backend.utils.s3 import generate_presigned_url


class UserService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._repository = UserRepository(session)
        self._data_loader = UserDataLoader(session)

    async def get_user(self, user_id: str) -> UserResponseDto:
        """ID로 사용자 조회"""
        user = await self._repository.get_by_id(user_id)
        if not user:
            raise NotFoundError(f"User {user_id} not found")
        return UserResponseDto.from_model(user)

    async def get_user_by_phone(self, phone: str) -> Optional[UserResponseDto]:
        """전화번호로 사용자 조회"""
        user = await self._repository.find_by_phone(phone)
        if not user:
            return None
        return UserResponseDto.from_model(user)

    async def create_user(self, dto: UserCreateDto) -> UserResponseDto:
        """새 사용자 생성"""
        # 비즈니스 규칙: 전화번호는 고유해야 함
        existing = await self._repository.find_by_phone(dto.phone)
        if existing:
            raise ConflictError("Phone number already registered")

        user = User(**dto.model_dump())
        created = await self._repository.create(user)
        return UserResponseDto.from_model(created)

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

    async def update_user(self, user_id: str, dto: UserUpdateDto) -> UserResponseDto:
        """사용자 수정"""
        user = await self._repository.get_by_id(user_id)
        if not user:
            raise NotFoundError(f"User {user_id} not found")

        # 업데이트 적용
        update_data = dto.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            if hasattr(user, field):
                setattr(user, field, value)

        user.updated_at = datetime.utcnow()
        await self._repository.update(user)
        return UserResponseDto.from_model(user)

    async def soft_delete_user(self, user_id: str) -> bool:
        """사용자 소프트 삭제"""
        user = await self._repository.get_by_id(user_id)
        if not user:
            return False
        await self._repository.soft_delete(user)
        return True
```

### DTO

```python
# backend/dtos/user.py
from pydantic import BaseModel, Field, field_validator
from typing import Optional, List
from datetime import datetime

from backend.domain.user.enums import GenderEnum, UserStatusEnum


class UserCreateDto(BaseModel):
    """사용자 생성 요청"""
    phone: str = Field(pattern=r'^01[0-9]\d{7,8}$')
    name: str = Field(min_length=1, max_length=50)
    gender: str
    birth_year: int = Field(ge=1950, le=2010)

    model_config = {"extra": "forbid"}

    @field_validator("gender")
    @classmethod
    def validate_gender(cls, v: str) -> str:
        valid_values = [e.value for e in GenderEnum]
        if v not in valid_values:
            raise ValueError(f"Invalid gender: {v}")
        return v


class UserResponseDto(BaseModel):
    """사용자 응답"""
    id: str
    phone: str
    name: str
    gender: str
    birth_year: int
    status: str
    created_at: datetime
    updated_at: Optional[datetime] = None

    @classmethod
    def from_model(cls, model: User) -> "UserResponseDto":
        return cls(
            id=model.id,
            phone=model.phone,
            name=model.name,
            gender=model.gender,
            birth_year=model.birth_year,
            status=model.status,
            created_at=model.created_at,
            updated_at=model.updated_at,
        )

    model_config = {"from_attributes": True}


class MemberDetailResponse(BaseModel):
    """관계를 포함한 회원 전체 상세"""
    id: str
    phone: str
    name: str
    gender: str
    birth_year: int
    status: str
    created_at: datetime

    # 프로필 정보
    height: Optional[int] = None
    education: Optional[str] = None
    job: Optional[str] = None
    district: Optional[str] = None
    mbti: Optional[str] = None
    about_me: Optional[str] = None

    # 사진 URL (presigned)
    photo_urls: List[str] = Field(default_factory=list)

    @classmethod
    def from_user_with_relations(
        cls,
        data: UserWithRelations,
        photo_urls: List[str] = None,
    ) -> "MemberDetailResponse":
        return cls(
            id=data.user.id,
            phone=data.user.phone,
            name=data.user.name,
            gender=data.user.gender,
            birth_year=data.user.birth_year,
            status=data.user.status,
            created_at=data.user.created_at,
            height=data.profile.height if data.profile else None,
            education=data.profile.education if data.profile else None,
            job=data.profile.job if data.profile else None,
            district=data.profile.district if data.profile else None,
            mbti=data.profile.mbti if data.profile else None,
            about_me=data.profile.about_me if data.profile else None,
            photo_urls=photo_urls or [],
        )
```

### 라우터

```python
# backend/api/v1/routers/user.py
from fastapi import APIRouter, Depends, status, Query, HTTPException
from sqlmodel.ext.asyncio.session import AsyncSession
from typing import List

from backend.db.orm import get_read_session_dependency, get_write_session_dependency
from backend.domain.user.service import UserService
from backend.dtos.user import UserCreateDto, UserResponseDto, MemberDetailResponse
from backend.error import NotFoundError, ConflictError

router = APIRouter(prefix="/api/v1/users", tags=["users"])


@router.get("/{user_id}", response_model=MemberDetailResponse)
async def get_user(
    user_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """프로필과 사진을 포함한 사용자 상세 조회"""
    service = UserService(session)
    try:
        return await service.get_member_detail(user_id)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@router.post("/", response_model=UserResponseDto, status_code=status.HTTP_201_CREATED)
async def create_user(
    dto: UserCreateDto,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """새 사용자 생성"""
    service = UserService(session)
    try:
        return await service.create_user(dto)
    except ConflictError as e:
        raise HTTPException(status_code=409, detail=str(e))


@router.patch("/{user_id}", response_model=UserResponseDto)
async def update_user(
    user_id: str,
    dto: UserUpdateDto,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """사용자 수정"""
    service = UserService(session)
    try:
        return await service.update_user(user_id, dto)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: str,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """사용자 소프트 삭제"""
    service = UserService(session)
    result = await service.soft_delete_user(user_id)
    if not result:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
```

### 라우터 등록

```python
# backend/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from backend.api.v1.routers.user import router as user_router
from backend.middleware.error_handler import ErrorHandlerMiddleware


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield


def create_application() -> FastAPI:
    app = FastAPI(title="YGS API", lifespan=lifespan)
    app.add_middleware(ErrorHandlerMiddleware)
    app.include_router(user_router)
    return app


app = create_application()
```

## 이 완전한 예시가 보여주는 것

- 계층형 아키텍처 (Router → Service → Repository)
- Domain-Driven Design 구조
- 엔티티 접두사를 사용한 ULID ID 생성
- 관계가 있는 SQLModel 모델
- BaseRepository를 사용한 Repository 패턴
- N+1 방지를 위한 UserDataLoader
- 비즈니스 로직이 있는 서비스 레이어
- field_validator를 사용한 Pydantic DTO
- 의존성 주입이 있는 FastAPI 라우터
- 읽기/쓰기 세션 분리
- 소프트 삭제 패턴
- asyncio.gather를 사용한 Async/await
- 커스텀 예외로 에러 처리
- 사진용 S3 presigned URL
- 페이지네이션 지원
- 전체에 타입 힌트
