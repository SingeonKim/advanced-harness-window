# Async/Await 패턴 - FastAPI

## 비동기 기초

FastAPI는 비동기 우선입니다. 모든 데이터베이스 작업은 async여야 합니다.

### 비동기 라우트 핸들러

```python
# backend/api/v1/routers/admin.py
@router.get("/members/{user_id}", response_model=MemberDetailResponse)
async def get_member(
    user_id: str,
    session: AsyncSession = Depends(get_read_session_dependency),
):
    """회원 상세 조회"""
    service = AdminService(session)
    return await service.get_member_detail(user_id)  # 비동기 메서드 await
```

### 비동기 서비스 메서드

```python
# backend/domain/admin/service.py
class AdminService:
    async def get_member_detail(self, user_id: str) -> MemberDetailResponse:
        """모든 관계를 포함한 회원 전체 상세 조회"""
        user_with_relations = await self._data_loader.load_user_with_relations(
            user_id,
            load_profile=True,
            load_photos=True,
        )

        if not user_with_relations:
            raise NotFoundError(f"User {user_id} not found")

        return MemberDetailResponse.from_user_with_relations(user_with_relations)
```

### 비동기 레포지토리 쿼리

```python
# backend/domain/user/repository.py
class UserRepository:
    async def get_by_id(self, id: str) -> Optional[User]:
        stmt = select(User).where(
            User.id == id,
            User.deleted_at.is_(None)
        )
        # 데이터베이스 쿼리 await
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()
```

## asyncio.gather를 사용한 동시 작업

### 병렬 대시보드 쿼리

```python
# backend/domain/admin/service.py
import asyncio

class AdminService:
    async def get_dashboard_stats(self) -> DashboardStatsResponse:
        """병렬 쿼리로 대시보드 통계 조회"""
        now = datetime.utcnow()
        week_ago = now - timedelta(days=7)
        month_ago = now - timedelta(days=30)

        # asyncio.gather를 사용하여 모든 통계 쿼리를 병렬로 실행
        (
            total_count,
            monthly_count,
            weekly_count,
            today_count,
            male_count,
            female_count,
            pending_count,
        ) = await asyncio.gather(
            self._user_repository.count_all(),
            self._user_repository.count_since(month_ago),
            self._user_repository.count_since(week_ago),
            self._user_repository.count_today(),
            self._user_repository.count_by_gender("male"),
            self._user_repository.count_by_gender("female"),
            self._user_repository.count_by_status("pending"),
        )

        return DashboardStatsResponse(
            total_members=total_count,
            monthly_signups=monthly_count,
            weekly_signups=weekly_count,
            today_signups=today_count,
            male_count=male_count,
            female_count=female_count,
            pending_reviews=pending_count,
        )
```

### 병렬 관계 로딩이 있는 UserDataLoader

```python
# backend/domain/user/repository.py
class UserDataLoader:
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
        """선택적 관계를 병렬로 사용자 로드"""

        queries = []
        query_names = []

        # 항상 사용자 로드
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

        # 모든 쿼리를 병렬로 실행
        results = await asyncio.gather(*queries)

        # 결과 매핑
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

### 병렬 S3 Presigned URL 생성

```python
# backend/domain/admin/service.py
import asyncio

class AdminService:
    async def get_member_detail(self, user_id: str) -> MemberDetailResponse:
        """병렬 사진 URL 생성을 포함한 회원 상세 조회"""
        user_with_relations = await self._data_loader.load_user_with_relations(
            user_id,
            load_profile=True,
            load_photos=True,
        )

        if not user_with_relations:
            raise NotFoundError(f"User {user_id} not found")

        # presigned URL을 병렬로 생성
        if user_with_relations.photos:
            photo_urls = await asyncio.gather(*[
                generate_presigned_url(photo.s3_key)
                for photo in user_with_relations.photos
            ])
        else:
            photo_urls = []

        return MemberDetailResponse.from_user_with_relations(
            user_with_relations,
            photo_urls=photo_urls,
        )
```

## 순차적 의존성

```python
async def create_user_with_profile(self, dto: UserCreateDto) -> UserResponseDto:
    """사용자와 프로필을 순차적으로 생성 (프로필에 user.id 필요)"""
    # 순차적으로 실행해야 함 (프로필이 user.id에 의존)
    user = User(
        phone=dto.phone,
        name=dto.name,
        gender=dto.gender,
        birth_year=dto.birth_year,
    )
    await self._repository.create(user)  # user.id 획득

    # 이제 user.id로 프로필 생성
    profile = UserProfile(
        id=user.id,
        user_id=user.id,
        height=dto.height,
        education=dto.education,
    )
    await self._profile_repository.create(profile)

    return UserResponseDto.from_model(user)
```

## 세션 관리

### AsyncSession 컨텍스트

```python
from sqlmodel.ext.asyncio.session import AsyncSession

# 의존성 주입으로 제공되는 세션
async def route_handler(
    session: AsyncSession = Depends(get_write_session_dependency),
):
    # 세션 자동 관리 (commit/rollback)
    service = Service(session)
    return await service.do_work()
```

### 읽기/쓰기 세션 분리

```python
# 읽기 작업 (SELECT)
@router.get("/members")
async def list_members(
    session: AsyncSession = Depends(get_read_session_dependency),
):
    service = AdminService(session)
    return await service.list_members()

# 쓰기 작업 (INSERT, UPDATE, DELETE)
@router.patch("/members/{user_id}")
async def update_member(
    user_id: str,
    dto: UpdateDto,
    session: AsyncSession = Depends(get_write_session_dependency),
):
    service = AdminService(session)
    return await service.update_member(user_id, dto)
```

## 자주 발생하는 문제

### 블로킹 작업

```python
# async 함수에서 블로킹 I/O를 사용하지 마세요
async def bad_handler():
    time.sleep(1)  # 이벤트 루프를 블로킹합니다!
    return "done"

# 비동기 대안 사용
async def good_handler():
    await asyncio.sleep(1)  # 논블로킹
    return "done"
```

### await 누락

```python
# await 누락 - 결과가 아닌 코루틴 반환
async def bad_service():
    item = self._repository.get_by_id(id)  # await 누락!
    return item  # 이것은 User 객체가 아닌 코루틴 객체입니다

# 항상 비동기 호출에 await 사용
async def good_service():
    item = await self._repository.get_by_id(id)
    return item  # User 객체입니다
```

### 병렬 가능 시 순차적 실행

```python
# 순차적 쿼리 - 느림
async def bad_dashboard():
    total = await self._repo.count_all()
    male = await self._repo.count_by_gender("male")
    female = await self._repo.count_by_gender("female")
    return {"total": total, "male": male, "female": female}

# 병렬 쿼리 - 빠름
async def good_dashboard():
    total, male, female = await asyncio.gather(
        self._repo.count_all(),
        self._repo.count_by_gender("male"),
        self._repo.count_by_gender("female"),
    )
    return {"total": total, "male": male, "female": female}
```

## 모범 사례

1. **전면적 비동기**: Route → Service → Repository
2. **async 호출에 await**: await 절대 잊지 말기
3. **블로킹 I/O 없음**: 비동기 라이브러리 사용
4. **가능하면 병렬**: 독립적인 쿼리에 `asyncio.gather()` 사용
5. **요청당 세션**: 의존성 주입
6. **에러 처리**: try/except는 async에서도 동작
7. **읽기/쓰기 분리**: 올바른 세션 의존성 사용
8. **UserDataLoader**: 사용자와 여러 관계를 로드할 때
9. **필요한 경우만 순차적**: 작업 간 의존성이 있을 때
