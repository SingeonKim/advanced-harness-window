---
name: fastapi-backend-guidelines
description: Python 비동기 애플리케이션을 위한 FastAPI 백엔드 개발 가이드라인. FastAPI 라우터를 사용한 Domain-Driven Design, SQLModel/SQLAlchemy ORM, Repository 패턴, 서비스 레이어, async/await 패턴, Pydantic 유효성 검사, 에러 처리. API, 라우트, 서비스, 레포지토리 생성 또는 백엔드 코드 작업 시 사용하세요.
---

# FastAPI 백엔드 개발 가이드라인

## 목적

async Python을 사용한 현대적인 FastAPI 개발을 위한 포괄적인 가이드. Domain-Driven Design, 계층형 아키텍처 (Router → Service → Repository), SQLModel ORM, 비동기 모범 사례를 강조합니다.

## 이 스킬을 사용하는 경우

- 새 API 라우트 또는 엔드포인트 생성 시
- 도메인 서비스 및 비즈니스 로직 구축 시
- 데이터 접근을 위한 레포지토리 구현 시
- SQLModel로 데이터베이스 모델 설정 시
- Async/await 패턴 및 에러 처리 시
- DDD로 백엔드 코드 구성 시
- Pydantic 유효성 검사 및 DTO 작업 시
- Python 비동기 모범 사례 적용 시

---

## 빠른 시작

### 새 API 라우트 체크리스트

API 엔드포인트를 만들고 있나요? 이 체크리스트를 따르세요:

- [ ] `backend/api/v1/routers/{domain}.py`에 라우트 정의
- [ ] 세션을 위한 FastAPI 의존성 주입 사용
- [ ] 읽기에는 `get_read_session_dependency`, 쓰기에는 `get_write_session_dependency` 사용
- [ ] 서비스 레이어 호출 (레포지토리에 직접 접근하지 않음)
- [ ] 요청/응답에 Pydantic DTO 사용
- [ ] 커스텀 예외로 에러 처리
- [ ] 적절한 HTTP 상태 코드 추가
- [ ] 전반적으로 async/await 사용
- [ ] docstring으로 문서화
- [ ] 모든 파라미터에 타입 힌트 사용

### 새 도메인 기능 체크리스트

새 도메인을 만들고 있나요? 이 구조를 설정하세요:

- [ ] `backend/domain/{domain}/` 디렉토리 생성
- [ ] `model.py` - ULID ID 생성이 있는 SQLModel 데이터베이스 모델
- [ ] `repository.py` - BaseRepository를 확장하는 데이터 접근 레이어
- [ ] `service.py` - 비즈니스 로직 레이어
- [ ] `backend/dtos/{domain}.py`에 DTO 생성
- [ ] `backend/api/v1/routers/{domain}.py`에 라우터 생성
- [ ] `main.py`에 라우터 등록
- [ ] 전반적으로 비동기 패턴 따르기
- [ ] 필요한 경우 `backend/domain/user/enums.py`에 열거형 추가

---

## 프로젝트 구조 빠른 참조

YGS 백엔드 구조:

```
backend/
  backend/
    main.py                  # lifespan이 있는 FastAPI 앱 생성

    api/
      v1/
        routers/             # API 라우트 핸들러
          admin.py           # 대시보드, 회원, 매칭
          auth.py            # 로그인, 회원가입, OAuth
          match.py           # 매치 주, 이력
          user.py            # 사용자 관리
          upload.py          # S3 presigned URL

    domain/                  # Domain-Driven Design
      user/
        model.py             # User, UserProfile, UserLifestyle 등
        repository.py        # UserRepository, UserDataLoader
        service.py           # UserService
        enums.py             # 모든 도메인 열거형
      auth/
        service.py           # AuthService (JWT, Firebase, Kakao)
        repository.py        # AuthRepository
      admin/
        model.py             # ConsultSchedule
        service.py           # AdminService
        repository.py        # AdminRepository
        matching_service.py  # MatchingService (점수 알고리즘)
      match/
        model.py             # MatchWeek, MatchHistory, MatchFeedback
        service.py           # MatchService
        repository.py        # MatchRepository
      llm/
        matching_service.py  # LLM 향상 매칭
      shared/
        base_repository.py   # 제네릭 BaseRepository

    dtos/                    # Pydantic DTO
      admin.py               # 대시보드, 회원 DTO
      auth.py                # 로그인, 회원가입, OAuth DTO
      match.py               # 매치 주, 이력 DTO
      user.py                # 사용자, 프로필 DTO
      llm_match.py           # LLM 매칭 DTO

    db/
      orm.py                 # 읽기/쓰기 세션 관리

    core/
      config.py              # Pydantic Settings 설정

    middleware/              # 미들웨어
      error_handler.py       # ErrorHandlerMiddleware
      admin_auth.py          # 관리자 인증

    utils/                   # 유틸리티
      s3.py                  # S3 presigned URL
      s3_private.py          # 개인 사용자 데이터 S3
      firebase.py            # Firebase 검증
      password.py            # bcrypt 해싱
      excel.py               # Excel 내보내기

    error/                   # 커스텀 예외
      __init__.py            # AppException, NotFoundError 등
```

---

## 공통 임포트 치트시트

```python
# FastAPI
from fastapi import APIRouter, Depends, HTTPException, Query, Request
from fastapi.responses import StreamingResponse

# SQLModel & SQLAlchemy
from sqlmodel import select, or_, and_, col
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlalchemy import func, desc
from sqlalchemy.orm import selectinload

# 데이터베이스
from backend.db.orm import get_write_session_dependency, get_read_session_dependency

# Pydantic
from pydantic import BaseModel, Field, field_validator, EmailStr

# 도메인
from backend.domain.user.model import User, UserProfile
from backend.domain.user.service import UserService
from backend.dtos.user import UserResponse, UserCreateRequest
from backend.error import NotFoundError, ForbiddenError, UnauthorizedError

# 타입 힌트
from typing import List, Optional, Dict, Any

# ID 생성
from ulid import ULID
```

---

## 주제별 가이드

### 계층형 아키텍처

**3계층 패턴:**
1. **Router 레이어**: API 엔드포인트, 요청 유효성 검사, 응답 포맷팅
2. **Service 레이어**: 비즈니스 로직, 오케스트레이션, 도메인 규칙
3. **Repository 레이어**: 데이터 접근, 쿼리, 데이터베이스 작업

**핵심 개념:**
- 라우터는 서비스를 호출 (레포지토리에 직접 접근하지 않음)
- 서비스는 비즈니스 로직을 오케스트레이션
- 레포지토리는 모든 데이터베이스 작업 처리
- 각 레이어는 명확한 책임을 가짐
- 스택 전반에서 Async/await 사용
- 읽기/쓰기 세션 분리

**[전체 가이드: resources/layered-architecture.md](resources/layered-architecture.md)**

---

### API 라우트 & 라우터

**주요 패턴: FastAPI 라우터**
- `backend/api/v1/routers/`에 라우터 생성
- 세션에 의존성 주입 사용
- GET 요청에는 `get_read_session_dependency` 사용
- POST/PATCH/DELETE에는 `get_write_session_dependency` 사용
- REST 관례 준수
- 적절한 HTTP 메서드와 상태 코드 사용
- 비동기 라우트 핸들러

**라우터 구조:**
```python
from fastapi import APIRouter, Depends
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.db.orm import get_read_session_dependency, get_write_session_dependency

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
async def get_user(
    user_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
) -> UserResponse:
    service = UserService(session)
    return await service.get_user(user_id)

@router.post("", status_code=201)
async def create_user(
    request: UserCreateRequest,
    session: AsyncSession = Depends(get_write_session_dependency),
) -> UserResponse:
    service = UserService(session)
    return await service.create_user(request)
```

**[전체 가이드: resources/api-routes.md](resources/api-routes.md)**

---

### 데이터베이스 & ORM

**SQLModel + SQLAlchemy:**
- 모델에 SQLModel 사용 (SQLAlchemy + Pydantic 결합)
- asyncpg 드라이버를 사용한 비동기 세션
- 캐싱이 있는 읽기/쓰기 세션 분리
- 모든 쿼리에 Repository 패턴
- 접두사를 사용한 ULID 기반 ID 생성

**모델 패턴:**
```python
from sqlmodel import SQLModel, Field, Column, DateTime, Text
from datetime import datetime, timezone
from ulid import ULID

def generate_user_id() -> str:
    """접두사를 사용하여 사용자 ID 생성."""
    return f"usr_{ULID()}"

class User(SQLModel, table=True):
    __tablename__ = "user"

    id: str = Field(
        default_factory=generate_user_id,
        primary_key=True,
        max_length=30,
    )
    phone: str = Field(sa_column=Column(Text, nullable=False, unique=True))
    name: str = Field(sa_column=Column(Text, nullable=False))
    gender: GenderEnum = Field(sa_column=Column(Text, nullable=False))

    # 소프트 삭제 패턴
    deleted_at: Optional[datetime] = Field(
        sa_column=Column(DateTime(timezone=True), nullable=True),
        default=None,
    )

    # 타임스탬프
    created_at: datetime = Field(
        sa_column=Column(DateTime(timezone=True), nullable=False),
        default_factory=lambda: datetime.now(tz=timezone.utc),
    )
    updated_at: datetime = Field(
        sa_column=Column(DateTime(timezone=True), nullable=False),
        default_factory=lambda: datetime.now(tz=timezone.utc),
    )
```

**[전체 가이드: resources/database-orm.md](resources/database-orm.md)**

---

### Domain-Driven Design

**도메인 구성:**
- `backend/domain/{name}/`에 각 도메인
- 포함: `model.py`, `repository.py`, `service.py`
- 명확한 관심사 분리
- 서비스에 비즈니스 로직
- 레포지토리에 데이터 접근

**도메인:**
- **user**: 사용자 관리 (User, UserProfile, UserLifestyle, UserPreference 등)
- **auth**: 인증 (JWT, Firebase, Kakao OAuth)
- **admin**: 관리자 대시보드, 회원 관리, 상담
- **match**: 매치 주, 이력, 피드백
- **llm**: Gemini를 사용한 LLM 향상 매칭
- **shared**: BaseRepository, 공통 유틸리티

**[전체 가이드: resources/domain-driven-design.md](resources/domain-driven-design.md)**

---

### 서비스 레이어

**서비스 패턴:**
- 비즈니스 로직 오케스트레이션
- 도메인 규칙 적용
- 데이터를 위해 레포지토리 호출
- 모델이 아닌 DTO 반환
- 트랜잭션 관리
- 병렬 쿼리에 asyncio.gather 사용

**서비스 구조:**
```python
class UserService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._user_repo = UserRepository(session)
        self._profile_repo = UserProfileRepository(session)
        self._data_loader = UserDataLoader(session)

    async def get_user_detail(self, user_id: str) -> UserDetailResponse:
        # N+1 방지를 위한 UserDataLoader 사용
        user_with_relations = await self._data_loader.load_user_with_relations(
            user_id,
            load_profile=True,
            load_photos=True,
        )
        if not user_with_relations:
            raise NotFoundError(f"User {user_id} not found")
        return self._to_detail_response(user_with_relations)
```

**[전체 가이드: resources/service-layer.md](resources/service-layer.md)**

---

### Repository 패턴

**Repository 패턴:**
- 데이터 접근 캡슐화
- CRUD를 위한 BaseRepository 확장
- 도메인 특화 쿼리
- 도메인 모델 반환
- 모든 쿼리는 비동기
- 소프트 삭제 지원

**Repository 구조:**
```python
from backend.domain.shared.base_repository import BaseRepository

class UserRepository(BaseRepository[User]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, User)

    async def find_by_phone(self, phone: str) -> Optional[User]:
        stmt = select(User).where(
            User.phone == phone,
            User.deleted_at.is_(None),
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()
```

**UserDataLoader 패턴 (N+1 방지):**
```python
@dataclass
class UserWithRelations:
    user: User
    profile: Optional[UserProfile] = None
    lifestyle: Optional[UserLifestyle] = None
    photos: List[UserPhoto] = field(default_factory=list)

class UserDataLoader:
    async def load_user_with_relations(
        self,
        user_id: str,
        load_profile: bool = False,
        load_photos: bool = False,
    ) -> Optional[UserWithRelations]:
        # 플래그에 따라 쿼리 구성
        queries = [self._load_user(user_id)]
        if load_profile:
            queries.append(self._load_profile(user_id))
        if load_photos:
            queries.append(self._load_photos(user_id))

        # 모든 쿼리를 병렬로 실행
        results = await asyncio.gather(*queries)
        # ... 결과 결합
```

**[전체 가이드: resources/repository-pattern.md](resources/repository-pattern.md)**

---

### DTO & 유효성 검사

**Pydantic DTO:**
- 요청/응답 데이터 전송 객체
- Pydantic을 사용한 유효성 검사
- 도메인 모델과 분리
- `backend/dtos/`에 위치
- 열거형 유효성 검사에 field_validator 사용

**DTO 패턴:**
```python
from pydantic import BaseModel, Field, field_validator
from backend.domain.user.enums import GenderEnum, UserStatusEnum

class AdminBasicInfoUpdateRequest(BaseModel):
    """기본 사용자 정보 업데이트 요청 DTO (관리자 전용)."""

    name: Optional[str] = Field(None, description="사용자 이름")
    status: Optional[str] = Field(None, description="사용자 상태")
    is_admin: Optional[bool] = Field(None, description="관리자 플래그")
    birth_year: Optional[int] = Field(None, ge=1940, le=2010, description="출생 연도")

    model_config = {"extra": "forbid"}  # 알 수 없는 필드 거부

    @field_validator("status")
    @classmethod
    def validate_status(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in UserStatusEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid status: {v}. Valid: {valid_values}")
        return v
```

**[전체 가이드: resources/dtos-validation.md](resources/dtos-validation.md)**

---

### Async/Await 패턴

**비동기 모범 사례:**
- 전반적으로 async/await 사용
- 비동기 데이터베이스 세션
- 적절한 세션 정리
- 블로킹 작업 방지
- 병렬 쿼리에 asyncio.gather 사용

**비동기 패턴:**
```python
# asyncio.gather를 사용한 병렬 쿼리
async def get_dashboard_data(self) -> dict:
    # 모든 쿼리를 병렬로 실행
    total, monthly, weekly, today = await asyncio.gather(
        self._get_total_count(),
        self._get_monthly_count(),
        self._get_weekly_count(),
        self._get_today_count(),
    )
    return {
        "total_members": total,
        "monthly_members": monthly,
        "weekly_members": weekly,
        "today_members": today,
    }
```

**[전체 가이드: resources/async-patterns.md](resources/async-patterns.md)**

---

### 에러 처리

**에러 처리 전략:**
- `backend/error/`의 커스텀 예외 클래스
- ErrorHandlerMiddleware를 통한 HTTP 예외 매핑
- 에러 처리를 위한 미들웨어
- 일관된 에러 응답

**에러 패턴:**
```python
# backend/error/__init__.py
class AppException(Exception):
    def __init__(self, message: str):
        self.message = message
        super().__init__(self.message)

class NotFoundError(AppException):
    pass

class ForbiddenError(AppException):
    pass

class UnauthorizedError(AppException):
    pass

# 서비스에서
if not user:
    raise NotFoundError(f"User {user_id} not found")

# ErrorHandlerMiddleware가 HTTP 응답으로 변환을 처리
# NotFoundError → 404, ForbiddenError → 403 등
```

**[전체 가이드: resources/error-handling.md](resources/error-handling.md)**

---

### 완전한 예시

**완전히 작동하는 예시:**
- 완전한 도메인 (model + repository + service + router)
- async를 사용한 CRUD 작업
- SQLModel을 사용한 복잡한 쿼리
- Firebase/Kakao 인증 패턴
- S3 presigned URL 생성
- 페이지네이션 및 필터링
- UserDataLoader를 사용한 N+1 방지

**[전체 가이드: resources/complete-examples.md](resources/complete-examples.md)**

---

## 탐색 가이드

| 해야 할 일... | 이 리소스를 읽으세요 |
|--------------|-------------------|
| 아키텍처 이해 | [layered-architecture.md](resources/layered-architecture.md) |
| API 라우트 생성 | [api-routes.md](resources/api-routes.md) |
| 데이터베이스 작업 | [database-orm.md](resources/database-orm.md) |
| 도메인 구성 | [domain-driven-design.md](resources/domain-driven-design.md) |
| 서비스 구축 | [service-layer.md](resources/service-layer.md) |
| 레포지토리 생성 | [repository-pattern.md](resources/repository-pattern.md) |
| 요청 유효성 검사 | [dtos-validation.md](resources/dtos-validation.md) |
| 비동기 패턴 사용 | [async-patterns.md](resources/async-patterns.md) |
| 에러 처리 | [error-handling.md](resources/error-handling.md) |
| 전체 예시 보기 | [complete-examples.md](resources/complete-examples.md) |

---

## 핵심 원칙

1. **계층형 아키텍처**: Router → Service → Repository (레이어 절대 건너뛰지 않음)
2. **Domain-Driven Design**: 타입이 아닌 도메인으로 구성
3. **전면적 비동기**: 스택 전반에서 async/await 사용
4. **Repository 패턴**: 레포지토리를 통한 모든 데이터 접근
5. **서비스 레이어**: 라우터나 레포지토리가 아닌 서비스에 비즈니스 로직
6. **API용 DTO**: 요청/응답에 Pydantic DTO 사용
7. **타입 힌트**: 모든 함수와 파라미터에 명시적 타입
8. **에러 처리**: 커스텀 예외, HTTP 매핑을 위한 미들웨어
9. **읽기/쓰기 분리**: 읽기와 쓰기 작업에 별도 세션
10. **의존성 주입**: 세션에 FastAPI의 Depends() 사용
11. **ULID ID**: 엔티티 접두사를 사용한 ULID (usr_, mw_, mh_ 등)
12. **소프트 삭제**: 하드 삭제 대신 deleted_at 타임스탬프 사용
13. **N+1 방지**: asyncio.gather 및 DataLoader 패턴 사용

---

## 빠른 참조: 새 도메인 템플릿

```python
# backend/domain/myfeature/model.py
from sqlmodel import SQLModel, Field, Column, DateTime, Text
from datetime import datetime, timezone
from typing import Optional
from ulid import ULID

def generate_myfeature_id() -> str:
    return f"mf_{ULID()}"

class MyFeature(SQLModel, table=True):
    __tablename__ = "my_feature"

    id: str = Field(
        default_factory=generate_myfeature_id,
        primary_key=True,
        max_length=30,
    )
    name: str = Field(sa_column=Column(Text, nullable=False))

    created_at: datetime = Field(
        sa_column=Column(DateTime(timezone=True), nullable=False),
        default_factory=lambda: datetime.now(tz=timezone.utc),
    )
    deleted_at: Optional[datetime] = Field(
        sa_column=Column(DateTime(timezone=True), nullable=True),
        default=None,
    )

# backend/domain/myfeature/repository.py
from backend.domain.shared.base_repository import BaseRepository
from sqlmodel.ext.asyncio.session import AsyncSession

class MyFeatureRepository(BaseRepository[MyFeature]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, MyFeature)

    async def find_by_name(self, name: str) -> Optional[MyFeature]:
        stmt = select(MyFeature).where(
            MyFeature.name == name,
            MyFeature.deleted_at.is_(None),
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

# backend/domain/myfeature/service.py
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.error import NotFoundError

class MyFeatureService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._repository = MyFeatureRepository(session)

    async def get_feature(self, id: str) -> MyFeatureResponse:
        feature = await self._repository.get_by_id(id)
        if not feature:
            raise NotFoundError(f"Feature {id} not found")
        return MyFeatureResponse.model_validate(feature)

# backend/dtos/myfeature.py
from pydantic import BaseModel, Field
from datetime import datetime

class MyFeatureResponse(BaseModel):
    id: str
    name: str
    created_at: datetime

class MyFeatureCreateRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)

# backend/api/v1/routers/myfeature.py
from fastapi import APIRouter, Depends, HTTPException
from sqlmodel.ext.asyncio.session import AsyncSession
from backend.db.orm import get_read_session_dependency, get_write_session_dependency

router = APIRouter(prefix="/myfeature", tags=["myfeature"])

@router.get("/{id}")
async def get_feature(
    id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
) -> MyFeatureResponse:
    service = MyFeatureService(session)
    return await service.get_feature(id)

@router.post("", status_code=201)
async def create_feature(
    request: MyFeatureCreateRequest,
    session: AsyncSession = Depends(get_write_session_dependency),
) -> MyFeatureResponse:
    service = MyFeatureService(session)
    return await service.create_feature(request)
```

---

## 관련 스킬

- **nextjs-frontend-guidelines**: 이 API를 사용하는 프론트엔드 패턴
- **error-tracking**: Sentry를 사용한 에러 추적 (백엔드 통합)
- **pytest-backend-testing**: FastAPI 백엔드 테스팅 패턴

---

**스킬 상태**: 최적의 컨텍스트 관리를 위한 점진적 로딩이 있는 모듈식 구조
