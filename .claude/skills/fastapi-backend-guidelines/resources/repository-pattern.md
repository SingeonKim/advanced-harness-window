# Repository 패턴 - FastAPI

## Repository 패턴

Repository는 도메인의 모든 데이터베이스 접근을 캡슐화합니다.

### BaseRepository

YGS는 제네릭 BaseRepository를 사용합니다:

```python
# backend/domain/shared/base_repository.py
from typing import Generic, TypeVar, Optional, List
from sqlmodel import select
from sqlmodel.ext.asyncio.session import AsyncSession
from datetime import datetime

T = TypeVar("T")

class BaseRepository(Generic[T]):
    def __init__(self, session: AsyncSession, model_class: type[T]):
        self.session = session
        self.model_class = model_class

    async def get_by_id(self, id: str) -> Optional[T]:
        """ID로 조회 (소프트 삭제된 것 제외)"""
        stmt = select(self.model_class).where(
            self.model_class.id == id,
            self.model_class.deleted_at.is_(None)
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, entity: T) -> T:
        """새 엔티티 생성"""
        self.session.add(entity)
        await self.session.flush()
        await self.session.refresh(entity)
        return entity

    async def update(self, entity: T) -> T:
        """엔티티 수정"""
        entity.updated_at = datetime.utcnow()
        self.session.add(entity)
        await self.session.flush()
        await self.session.refresh(entity)
        return entity

    async def soft_delete(self, entity: T) -> T:
        """엔티티 소프트 삭제 (deleted_at 설정)"""
        entity.deleted_at = datetime.utcnow()
        self.session.add(entity)
        await self.session.flush()
        return entity
```

### 도메인 특화 메서드가 있는 UserRepository

```python
# backend/domain/user/repository.py
from typing import List, Optional, Tuple
from sqlmodel import select, or_, func
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlalchemy.orm import selectinload

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

    async def find_by_kakao_id(self, kakao_id: str) -> Optional[User]:
        """Kakao ID로 사용자 조회"""
        stmt = select(User).where(
            User.kakao_id == kakao_id,
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
        """필터와 페이지네이션으로 회원 검색"""
        # 기본 쿼리 구성
        stmt = select(User).where(User.deleted_at.is_(None))

        # 필터 적용
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
```

## N+1 방지를 위한 UserDataLoader

YGS는 N+1 쿼리 방지를 위해 `asyncio.gather`와 함께 UserDataLoader 패턴을 사용합니다:

```python
# backend/domain/user/repository.py
import asyncio
from typing import Optional
from dataclasses import dataclass

@dataclass
class UserWithRelations:
    """관계가 로드된 사용자 컨테이너"""
    user: User
    profile: Optional[UserProfile] = None
    lifestyle: Optional[UserLifestyle] = None
    preference: Optional[UserPreference] = None
    subscription: Optional[UserSubscription] = None
    photos: List[UserPhoto] = None
    documents: List[UserDocument] = None

class UserDataLoader:
    """N+1 쿼리 방지를 위한 병렬 로더"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def load_user_with_relations(
        self,
        user_id: str,
        load_profile: bool = False,
        load_lifestyle: bool = False,
        load_preference: bool = False,
        load_subscription: bool = False,
        load_photos: bool = False,
        load_documents: bool = False,
    ) -> Optional[UserWithRelations]:
        """선택적 관계를 병렬로 사용자와 함께 로드"""

        # 병렬로 실행할 쿼리 목록 구성
        queries = []
        query_names = []

        # 항상 사용자 로드
        async def load_user():
            stmt = select(User).where(
                User.id == user_id,
                User.deleted_at.is_(None)
            )
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

        if load_lifestyle:
            async def load_lifestyle_fn():
                stmt = select(UserLifestyle).where(UserLifestyle.user_id == user_id)
                result = await self.session.execute(stmt)
                return result.scalar_one_or_none()
            queries.append(load_lifestyle_fn())
            query_names.append("lifestyle")

        if load_preference:
            async def load_preference_fn():
                stmt = select(UserPreference).where(UserPreference.user_id == user_id)
                result = await self.session.execute(stmt)
                return result.scalar_one_or_none()
            queries.append(load_preference_fn())
            query_names.append("preference")

        if load_subscription:
            async def load_subscription_fn():
                stmt = select(UserSubscription).where(
                    UserSubscription.user_id == user_id,
                    UserSubscription.deleted_at.is_(None)
                )
                result = await self.session.execute(stmt)
                return result.scalar_one_or_none()
            queries.append(load_subscription_fn())
            query_names.append("subscription")

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

        if load_documents:
            async def load_documents_fn():
                stmt = select(UserDocument).where(
                    UserDocument.user_id == user_id,
                    UserDocument.deleted_at.is_(None)
                )
                result = await self.session.execute(stmt)
                return list(result.scalars().all())
            queries.append(load_documents_fn())
            query_names.append("documents")

        # 모든 쿼리 병렬 실행
        results = await asyncio.gather(*queries)

        # 결과를 이름 딕셔너리에 매핑
        result_dict = dict(zip(query_names, results))

        # 사용자 존재 확인
        user = result_dict.get("user")
        if not user:
            return None

        return UserWithRelations(
            user=user,
            profile=result_dict.get("profile"),
            lifestyle=result_dict.get("lifestyle"),
            preference=result_dict.get("preference"),
            subscription=result_dict.get("subscription"),
            photos=result_dict.get("photos", []),
            documents=result_dict.get("documents", []),
        )
```

### 서비스에서 UserDataLoader 사용

```python
# backend/domain/user/service.py
class UserService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._repository = UserRepository(session)
        self._data_loader = UserDataLoader(session)

    async def get_member_detail(self, user_id: str) -> MemberDetailResponse:
        """관계를 포함한 회원 전체 상세 조회"""
        # 단일 호출로 모든 것을 병렬로 로드
        user_with_relations = await self._data_loader.load_user_with_relations(
            user_id,
            load_profile=True,
            load_lifestyle=True,
            load_preference=True,
            load_subscription=True,
            load_photos=True,
            load_documents=True,
        )

        if not user_with_relations:
            raise NotFoundError(f"User {user_id} not found")

        return MemberDetailResponse.from_user_with_relations(user_with_relations)
```

## Repository 책임

1. **데이터베이스 쿼리**: 모든 SELECT/INSERT/UPDATE/DELETE
2. **쿼리 최적화**: 인덱스를 활용한 효율적인 쿼리
3. **모델 반환**: 항상 도메인 모델 반환
4. **비즈니스 로직 없음**: 순수 데이터 접근만
5. **DTO 없음**: 모델만 사용
6. **소프트 삭제 인식**: 항상 `deleted_at.is_(None)` 필터링

## 모범 사례

1. **BaseRepository 확장**: 공통 CRUD 작업 재사용
2. **도메인 특화 메서드**: 도메인 쿼리를 위한 메서드 추가
3. **모델 반환**: DTO는 절대 반환하지 않음
4. **비동기 쿼리**: 모든 메서드 async
5. **타입 힌트**: 명시적 반환 타입
6. **트랜잭션 없음**: Repository는 commit/rollback하지 않음
7. **UserDataLoader 사용**: 여러 관계를 병렬로 로드할 때
8. **소프트 삭제 필터**: 모든 쿼리에 `deleted_at.is_(None)` 포함
