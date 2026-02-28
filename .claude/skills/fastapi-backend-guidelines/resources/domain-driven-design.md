# Domain-Driven Design - FastAPI

## 도메인 구성

각 도메인은 이 구조를 따릅니다:

```
backend/domain/{domain}/
  __init__.py
  model.py         # SQLModel 데이터베이스 모델
  repository.py    # 데이터 접근 레이어
  service.py       # 비즈니스 로직 레이어
  enums.py         # 도메인 특화 열거형 (필요한 경우)
```

## YGS 도메인

YGS 프로젝트의 현재 도메인:

| 도메인 | 설명 | 주요 모델 |
|--------|------|-----------|
| **user** | 사용자 관리 | User, UserProfile, UserLifestyle, UserPreference, UserDocument, UserPhoto, UserSubscription, UserAccessAudit |
| **auth** | 인증 | JWT 토큰, Firebase 소셜 인증, Kakao OAuth |
| **admin** | 관리자 대시보드 | ConsultSchedule, 통계, 회원 관리 |
| **match** | 매칭 시스템 | MatchWeek, MatchHistory, MatchFeedback |
| **llm** | LLM 통합 | Gemini 향상 호환성 분석 |
| **shared** | 공유 유틸리티 | BaseRepository, 공통 헬퍼 |

## 도메인 구조 예시

### User 도메인

```
backend/domain/user/
  __init__.py
  model.py         # User, UserProfile, UserLifestyle, UserPreference,
                   # UserDocument, UserPhoto, UserSubscription, UserAccessAudit
  repository.py    # UserRepository, UserDataLoader
  service.py       # UserService
  enums.py         # GenderEnum, UserStatusEnum, EducationEnum 등
```

### Match 도메인

```
backend/domain/match/
  __init__.py
  model.py         # MatchWeek, MatchHistory, MatchFeedback
  repository.py    # MatchWeekRepository, MatchHistoryRepository
  service.py       # MatchService
```

### Admin 도메인

```
backend/domain/admin/
  __init__.py
  model.py         # ConsultSchedule
  repository.py    # ConsultScheduleRepository
  service.py       # AdminService
  matching_service.py  # 호환성 점수 계산
```

## 새 도메인 만들기

1. **디렉토리 생성**: `backend/domain/newdomain/`

2. **model.py 생성**: ULID ID가 있는 데이터베이스 모델
```python
# backend/domain/newdomain/model.py
from sqlmodel import SQLModel, Field
from datetime import datetime
from ulid import ULID

def generate_newdomain_id() -> str:
    return f"nd_{ULID()}"  # 적절한 접두사 사용

class NewEntity(SQLModel, table=True):
    __tablename__ = "new_entities"

    id: str = Field(default_factory=generate_newdomain_id, primary_key=True)
    # 필드들...
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: Optional[datetime] = None
    deleted_at: Optional[datetime] = None  # 소프트 삭제
```

3. **repository.py 생성**: 데이터 접근
```python
# backend/domain/newdomain/repository.py
from backend.domain.shared.base_repository import BaseRepository
from backend.domain.newdomain.model import NewEntity

class NewEntityRepository(BaseRepository[NewEntity]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, NewEntity)

    # 도메인 특화 쿼리...
```

4. **service.py 생성**: 비즈니스 로직
```python
# backend/domain/newdomain/service.py
from backend.domain.newdomain.repository import NewEntityRepository
from backend.dtos.newdomain import NewEntityCreateDto, NewEntityResponseDto
from backend.error import NotFoundError

class NewEntityService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._repository = NewEntityRepository(session)

    async def get_entity(self, entity_id: str) -> NewEntityResponseDto:
        entity = await self._repository.get_by_id(entity_id)
        if not entity:
            raise NotFoundError(f"Entity {entity_id} not found")
        return NewEntityResponseDto.from_model(entity)
```

5. **DTO 생성**: `backend/dtos/newdomain.py`
```python
# backend/dtos/newdomain.py
from pydantic import BaseModel, Field, field_validator

class NewEntityCreateDto(BaseModel):
    # 요청 필드들...
    model_config = {"extra": "forbid"}

class NewEntityResponseDto(BaseModel):
    id: str
    # 응답 필드들...

    @classmethod
    def from_model(cls, model) -> "NewEntityResponseDto":
        return cls(id=model.id, ...)
```

6. **라우터 생성**: `backend/api/v1/routers/newdomain.py`
```python
# backend/api/v1/routers/newdomain.py
from fastapi import APIRouter, Depends
from backend.db.orm import get_read_session_dependency

router = APIRouter(prefix="/api/v1/newdomain", tags=["newdomain"])

@router.get("/{entity_id}")
async def get_entity(
    entity_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
):
    service = NewEntityService(session)
    return await service.get_entity(entity_id)
```

7. **라우터 등록**: `main.py`에 추가
```python
# backend/main.py
from backend.api.v1.routers.newdomain import router as newdomain_router

app.include_router(newdomain_router)
```

## 도메인 독립성

- 도메인은 가능한 한 독립적이어야 함
- `shared` 도메인을 통해 공통 코드 공유
- 순환 의존성 방지
- 도메인 간 통신에 DTO 사용

## 도메인 간 통신

한 서비스가 다른 도메인의 데이터가 필요한 경우:

```python
# backend/domain/match/service.py
class MatchService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._match_repository = MatchHistoryRepository(session)
        self._user_repository = UserRepository(session)  # user 도메인에서

    async def create_match(self, dto: MatchCreateDto) -> MatchResponse:
        # 사용자 존재 확인 (도메인 간 확인)
        user = await self._user_repository.get_by_id(dto.user_id)
        if not user:
            raise NotFoundError("User not found")

        # 이 도메인에서 매치 생성
        match = MatchHistory(**dto.model_dump())
        return await self._match_repository.create(match)
```

## Shared 도메인

`shared` 도메인의 내용:

```
backend/domain/shared/
  __init__.py
  base_repository.py  # 제네릭 BaseRepository[T]
```

```python
# backend/domain/shared/base_repository.py
from typing import Generic, TypeVar, Optional
from sqlmodel import select
from datetime import datetime

T = TypeVar("T")

class BaseRepository(Generic[T]):
    def __init__(self, session: AsyncSession, model_class: type[T]):
        self.session = session
        self.model_class = model_class

    async def get_by_id(self, id: str) -> Optional[T]:
        stmt = select(self.model_class).where(
            self.model_class.id == id,
            self.model_class.deleted_at.is_(None)
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, entity: T) -> T:
        self.session.add(entity)
        await self.session.flush()
        await self.session.refresh(entity)
        return entity

    async def update(self, entity: T) -> T:
        entity.updated_at = datetime.utcnow()
        self.session.add(entity)
        await self.session.flush()
        await self.session.refresh(entity)
        return entity

    async def soft_delete(self, entity: T) -> T:
        entity.deleted_at = datetime.utcnow()
        self.session.add(entity)
        await self.session.flush()
        return entity
```

## 열거형 패턴

```python
# backend/domain/user/enums.py
from enum import Enum

class GenderEnum(str, Enum):
    MALE = "male"
    FEMALE = "female"

class UserStatusEnum(str, Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    SUSPENDED = "suspended"

class EducationEnum(str, Enum):
    HIGH_SCHOOL = "high_school"
    COLLEGE = "college"
    UNIVERSITY = "university"
    GRADUATE = "graduate"

class MatchCategoryEnum(str, Enum):
    INTRO = "intro"
    EXTRA = "extra"
```

## 모범 사례

1. **단일 책임**: 각 도메인은 하나의 명확한 목적을 가짐
2. **캡슐화**: 구현 세부사항 숨기기
3. **일관된 구조**: 모든 도메인은 같은 패턴을 따름
4. **공유 유틸리티**: 공통 코드에 `shared` 도메인 사용
5. **명확한 경계**: 도메인 간 의존성 최소화
6. **ULID ID**: 엔티티 특화 접두사 사용
7. **소프트 삭제**: `deleted_at` 타임스탬프 사용
8. **열거형**: 도메인의 `enums.py`에 정의
9. **DTO**: `backend/dtos/{domain}.py`에 유지
10. **라우터**: `backend/api/v1/routers/{domain}.py`에 유지
