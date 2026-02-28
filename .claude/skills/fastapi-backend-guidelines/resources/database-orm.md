# 데이터베이스 & ORM - SQLModel + SQLAlchemy

## YGS 데이터베이스 설정

YGS는 PostgreSQL 비동기 접근을 위해 asyncpg를 사용한 SQLAlchemy ORM과 SQLModel을 사용합니다.

### ULID를 사용한 ID 생성

모든 엔티티는 읽기 쉽고 정렬 가능한 ID를 위해 엔티티 접두사가 있는 ULID를 사용합니다:

```python
# backend/domain/user/model.py
from ulid import ULID

def generate_user_id() -> str:
    return f"usr_{ULID()}"  # usr_01HQ5K3NXYZ...

def generate_document_id() -> str:
    return f"doc_{ULID()}"  # doc_01HQ5K3NXYZ...

def generate_photo_id() -> str:
    return f"pho_{ULID()}"  # pho_01HQ5K3NXYZ...

# 엔티티 접두사:
# usr_ - User
# doc_ - UserDocument
# pho_ - UserPhoto
# sub_ - UserSubscription
# aud_ - UserAccessAudit
# mw_  - MatchWeek
# mh_  - MatchHistory
# mf_  - MatchFeedback
# cs_  - ConsultSchedule
```

### 사용자 모델 (SQLModel)

```python
# backend/domain/user/model.py
from sqlmodel import SQLModel, Field, Relationship
from typing import Optional, List
from datetime import datetime

class User(SQLModel, table=True):
    __tablename__ = "users"

    id: str = Field(default_factory=generate_user_id, primary_key=True)
    firebase_id: Optional[str] = Field(default=None, index=True)
    kakao_id: Optional[str] = Field(default=None, index=True)
    email: Optional[str] = Field(default=None, index=True)
    phone: str = Field(unique=True, index=True)
    name: str = Field(max_length=50)
    gender: str = Field(index=True)  # GenderEnum 값
    birth_year: int
    status: str = Field(default="pending", index=True)  # UserStatusEnum
    is_admin: bool = Field(default=False)

    # 소프트 삭제
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: Optional[datetime] = None
    deleted_at: Optional[datetime] = None  # 소프트 삭제 마커

    # 관계 (일대일)
    profile: Optional["UserProfile"] = Relationship(back_populates="user")
    lifestyle: Optional["UserLifestyle"] = Relationship(back_populates="user")
    preference: Optional["UserPreference"] = Relationship(back_populates="user")
    subscription: Optional["UserSubscription"] = Relationship(back_populates="user")

    # 관계 (일대다)
    photos: List["UserPhoto"] = Relationship(back_populates="user")
    documents: List["UserDocument"] = Relationship(back_populates="user")
```

### 관련 모델

```python
class UserProfile(SQLModel, table=True):
    __tablename__ = "user_profiles"

    id: str = Field(primary_key=True)  # user_id와 동일
    user_id: str = Field(foreign_key="users.id", unique=True)
    height: Optional[int] = None
    education: Optional[str] = None  # EducationEnum
    university: Optional[str] = None
    job: Optional[str] = None
    salary_range: Optional[str] = None  # SalaryRangeEnum
    district: Optional[str] = None
    mbti: Optional[str] = None
    about_me: Optional[str] = None
    profile_appeal: Optional[str] = None

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

## 읽기/쓰기 세션 분리

YGS는 읽기와 쓰기 작업에 별도의 데이터베이스 연결을 사용합니다:

```python
# backend/db/orm.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlmodel.ext.asyncio.session import AsyncSession as SQLModelAsyncSession
from contextlib import asynccontextmanager
from weakref import WeakKeyDictionary
import asyncio

# 이벤트 루프별 엔진 캐싱 (노트북/멀티 루프 지원)
_read_engines: WeakKeyDictionary = WeakKeyDictionary()
_write_engines: WeakKeyDictionary = WeakKeyDictionary()

def get_read_engine():
    loop = asyncio.get_event_loop()
    if loop not in _read_engines:
        _read_engines[loop] = create_async_engine(
            settings.POSTGRES_READ_URL,
            pool_size=15,
            max_overflow=25,
            pool_recycle=3600,
        )
    return _read_engines[loop]

def get_write_engine():
    loop = asyncio.get_event_loop()
    if loop not in _write_engines:
        _write_engines[loop] = create_async_engine(
            settings.POSTGRES_WRITE_URL,
            pool_size=15,
            max_overflow=25,
            pool_recycle=3600,
        )
    return _write_engines[loop]

@asynccontextmanager
async def get_read_session():
    """SELECT 쿼리를 위한 읽기 세션 (async with를 사용한 수동 사용)"""
    Session = get_read_sessionmaker()
    async with Session() as session:
        yield session

@asynccontextmanager
async def get_write_session():
    """INSERT/UPDATE/DELETE를 위한 쓰기 세션 (async with를 사용한 수동 사용)"""
    Session = get_write_sessionmaker()
    async with Session() as session:
        yield session

# FastAPI 의존성
# 참고: 클라이언트가 요청 처리 중 연결을 끊을 때 IllegalStateChangeError를 피하기 위해
# try/finally와 안전한 close를 사용
async def get_read_session_dependency():
    Session = get_read_sessionmaker()
    session = Session()
    try:
        yield session
    finally:
        try:
            await session.close()
        except Exception as e:
            # 로그는 하되 발생시키지 않음 - 클라이언트가 연결을 끊으면 세션이 유효하지 않은 상태일 수 있음
            logger.debug(f"Session close failed (likely client disconnect): {e}")

async def get_write_session_dependency():
    Session = get_write_sessionmaker()
    session = Session()
    try:
        yield session
    finally:
        try:
            await session.close()
        except Exception as e:
            # 로그는 하되 발생시키지 않음 - 클라이언트가 연결을 끊으면 세션이 유효하지 않은 상태일 수 있음
            logger.debug(f"Session close failed (likely client disconnect): {e}")
```

> **경고**: FastAPI 의존성 함수에서 `async with Session() as sess: yield sess` 패턴을 사용하지 마세요.
> 이 패턴은 클라이언트가 요청 중간에 연결을 끊을 때 `IllegalStateChangeError`를 일으킵니다.
> 세션의 `close()`가 아직 중간 상태에 있는 동안 호출되기 때문입니다.

## SQLModel을 사용한 쿼리

### 소프트 삭제가 있는 기본 쿼리

```python
from sqlmodel import select

# ID로 조회 (소프트 삭제된 것 제외)
stmt = select(User).where(
    User.id == user_id,
    User.deleted_at.is_(None)  # 소프트 삭제된 것 제외
)
result = await session.execute(stmt)
user = result.scalar_one_or_none()

# 모든 활성 사용자 조회
stmt = select(User).where(User.deleted_at.is_(None))
result = await session.execute(stmt)
users = result.scalars().all()

# 상태로 필터
stmt = select(User).where(
    User.status == "approved",
    User.deleted_at.is_(None)
)
result = await session.execute(stmt)
approved = result.scalars().all()
```

### 복잡한 쿼리

```python
from sqlmodel import select, or_, and_
from sqlalchemy import func

# 다중 조건
stmt = select(User).where(
    and_(
        User.status == "approved",
        User.gender == "male",
        User.deleted_at.is_(None)
    )
)

# 검색이 있는 OR 조건
stmt = select(User).where(
    and_(
        or_(
            User.name.ilike(f"%{keyword}%"),
            User.phone.ilike(f"%{keyword}%")
        ),
        User.deleted_at.is_(None)
    )
)

# 정렬
stmt = select(User).order_by(User.created_at.desc())

# 페이지네이션
stmt = select(User).offset(offset).limit(limit)

# 카운트
stmt = select(func.count(User.id)).where(User.deleted_at.is_(None))
result = await session.execute(stmt)
count = result.scalar()
```

### 관련 테이블과의 조인

```python
# User와 Profile 조인
from sqlalchemy.orm import selectinload

stmt = (
    select(User)
    .options(selectinload(User.profile))
    .where(User.id == user_id)
)
result = await session.execute(stmt)
user = result.scalar_one_or_none()
# 접근: user.profile.height, user.profile.education

# 여러 관계 로드
stmt = (
    select(User)
    .options(
        selectinload(User.profile),
        selectinload(User.photos),
        selectinload(User.subscription)
    )
    .where(User.id == user_id)
)
```

## CRUD 작업

### ULID로 생성

```python
user = User(
    phone="01012345678",
    name="김철수",
    gender="male",
    birth_year=1990,
    status="pending"
)
# id 자동 생성: usr_01HQ5K3NXYZ...
session.add(user)
await session.commit()
await session.refresh(user)
```

### 소프트 삭제 확인이 있는 읽기

```python
stmt = select(User).where(
    User.id == user_id,
    User.deleted_at.is_(None)
)
result = await session.execute(stmt)
user = result.scalar_one_or_none()
```

### 수정

```python
stmt = select(User).where(User.id == user_id)
result = await session.execute(stmt)
user = result.scalar_one_or_none()

if user:
    user.name = "김영희"
    user.updated_at = datetime.utcnow()
    session.add(user)
    await session.commit()
    await session.refresh(user)
```

### 소프트 삭제

```python
# YGS는 소프트 삭제를 사용 - 실제 삭제 대신 deleted_at 설정
stmt = select(User).where(User.id == user_id)
result = await session.execute(stmt)
user = result.scalar_one_or_none()

if user:
    user.deleted_at = datetime.utcnow()
    session.add(user)
    await session.commit()
```

## 트랜잭션

```python
async def create_user_with_profile(session: AsyncSession, dto: UserCreateDto):
    # 같은 세션의 모든 작업 = 같은 트랜잭션
    user = User(
        phone=dto.phone,
        name=dto.name,
        gender=dto.gender,
        birth_year=dto.birth_year
    )
    session.add(user)
    await session.flush()  # 커밋하지 않고 user.id 획득

    profile = UserProfile(
        id=user.id,
        user_id=user.id,
        height=dto.height,
        education=dto.education
    )
    session.add(profile)

    await session.commit()  # 모든 변경사항을 원자적으로 커밋
    return user
```

## 모범 사례

1. **ULID ID**: 읽기 쉬운 ID를 위한 엔티티 접두사 사용
2. **소프트 삭제**: 모든 쿼리에서 항상 `deleted_at.is_(None)` 확인
3. **읽기/쓰기 분리**: 올바른 세션 의존성 사용
4. **전면적 비동기**: AsyncSession 사용, 모든 쿼리 await
5. **커밋 후 refresh**: DB가 생성한 값 가져오기
6. **자주 쿼리되는 컬럼 인덱싱**: `index=True`
7. **유니크 제약**: 전화번호, 이메일에 `unique=True`
8. **ID 사용 전 flush**: 생성된 ID를 가져오려면 `flush()` 사용
9. **S3 키만 저장**: S3 키만 저장하고 URL은 저장하지 않음 (필요할 때 presigned URL 생성)
