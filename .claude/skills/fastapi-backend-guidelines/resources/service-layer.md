# Service 레이어 - FastAPI

## Service 패턴

Service는 비즈니스 로직을 포함하고 Repository를 오케스트레이션합니다.

### UserService 예시

```python
# backend/domain/user/service.py
from typing import List, Optional
from datetime import datetime
from sqlmodel.ext.asyncio.session import AsyncSession

from backend.domain.user.repository import UserRepository, UserDataLoader
from backend.domain.user.model import User, UserProfile, UserPhoto
from backend.dtos.user import (
    UserCreateDto,
    UserResponseDto,
    MemberDetailResponse,
    MemberListResponse,
)
from backend.error import NotFoundError, ValidationError, ConflictError
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
        """비즈니스 유효성 검사와 함께 사용자 생성"""
        # 비즈니스 규칙: 전화번호 중복 불가
        existing = await self._repository.find_by_phone(dto.phone)
        if existing:
            raise ConflictError("Phone number already registered")

        user = User(**dto.model_dump())
        created = await self._repository.create(user)
        return UserResponseDto.from_model(created)
```

### 복잡한 로직이 있는 AdminService

```python
# backend/domain/admin/service.py
import asyncio
from typing import List, Optional, Tuple
from datetime import datetime, timedelta
from sqlmodel.ext.asyncio.session import AsyncSession

from backend.domain.user.repository import UserRepository, UserDataLoader
from backend.domain.admin.repository import ConsultScheduleRepository
from backend.dtos.admin import (
    DashboardStatsResponse,
    MemberListResponse,
    MemberDetailResponse,
    AdminBasicInfoUpdateRequest,
)
from backend.error import NotFoundError, ValidationError
from backend.utils.s3 import generate_presigned_url

class AdminService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._user_repository = UserRepository(session)
        self._consult_repository = ConsultScheduleRepository(session)
        self._data_loader = UserDataLoader(session)

    async def get_dashboard_stats(self) -> DashboardStatsResponse:
        """병렬 쿼리로 대시보드 통계 조회"""
        now = datetime.utcnow()
        week_ago = now - timedelta(days=7)
        month_ago = now - timedelta(days=30)

        # 모든 통계 쿼리를 병렬로 실행
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

    async def get_member_detail(self, user_id: str) -> MemberDetailResponse:
        """관계를 포함한 회원 전체 상세 조회"""
        # 병렬 관계 로딩을 위한 데이터 로더 사용
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

        # 사진의 presigned URL 생성
        photo_urls = []
        for photo in user_with_relations.photos:
            url = await generate_presigned_url(photo.s3_key)
            photo_urls.append(url)

        return MemberDetailResponse.from_user_with_relations(
            user_with_relations,
            photo_urls=photo_urls,
        )

    async def update_member_basic_info(
        self,
        user_id: str,
        dto: AdminBasicInfoUpdateRequest,
    ) -> MemberDetailResponse:
        """회원 기본 정보 수정 (관리자 액션)"""
        user = await self._user_repository.get_by_id(user_id)
        if not user:
            raise NotFoundError(f"User {user_id} not found")

        # DTO에서 업데이트 적용 (None이 아닌 필드만)
        update_data = dto.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            if hasattr(user, field):
                setattr(user, field, value)

        user.updated_at = datetime.utcnow()
        await self._user_repository.update(user)

        return await self.get_member_detail(user_id)
```

## Service 책임

1. **비즈니스 로직**: 도메인 규칙 구현
2. **유효성 검사**: 비즈니스 수준 유효성 검사 (DTO 이상)
3. **오케스트레이션**: 여러 Repository 조율
4. **변환**: Model ↔ DTO 변환
5. **에러 처리**: 도메인 예외 발생
6. **Presigned URL**: 사진/문서를 위한 S3 URL 생성

## Firebase & Kakao를 사용하는 AuthService

```python
# backend/domain/auth/service.py
from typing import Optional
from datetime import datetime
from sqlmodel.ext.asyncio.session import AsyncSession

from backend.domain.user.repository import UserRepository
from backend.domain.user.model import User
from backend.dtos.auth import (
    LoginRequest,
    LoginResponse,
    KakaoLoginRequest,
    FirebaseLoginRequest,
)
from backend.error import UnauthorizedError, NotFoundError, UserNotFoundSignupRequiredError
from backend.utils.firebase import verify_firebase_token, create_custom_token
from backend.utils.password import verify_password
from backend.core.config import settings
import jwt

class AuthService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._user_repository = UserRepository(session)

    async def login_with_phone(self, dto: LoginRequest) -> LoginResponse:
        """전화번호와 비밀번호로 로그인"""
        user = await self._user_repository.find_by_phone(dto.phone)
        if not user:
            raise UnauthorizedError("Invalid credentials")

        if not verify_password(dto.password, user.hashed_password):
            raise UnauthorizedError("Invalid credentials")

        tokens = self._generate_tokens(user)
        return LoginResponse(**tokens, user=UserResponseDto.from_model(user))

    async def login_with_firebase(self, dto: FirebaseLoginRequest) -> LoginResponse:
        """Firebase ID 토큰으로 로그인"""
        # Firebase 토큰 검증
        decoded = await verify_firebase_token(dto.id_token)
        firebase_id = decoded["uid"]

        # Firebase ID로 사용자 조회
        user = await self._user_repository.find_by_firebase_id(firebase_id)
        if not user:
            # 회원가입 필요 - Firebase 정보와 함께 특수 예외 발생
            raise UserNotFoundSignupRequiredError(
                message="User not found, signup required",
                firebase_id=firebase_id,
                email=decoded.get("email"),
            )

        tokens = self._generate_tokens(user)
        return LoginResponse(**tokens, user=UserResponseDto.from_model(user))

    async def login_with_kakao(self, dto: KakaoLoginRequest) -> LoginResponse:
        """Kakao 액세스 토큰으로 로그인"""
        # Kakao 토큰 검증 및 사용자 정보 조회
        kakao_user = await self._validate_kakao_token(dto.access_token)
        kakao_id = str(kakao_user["id"])

        # Kakao ID로 사용자 조회
        user = await self._user_repository.find_by_kakao_id(kakao_id)
        if not user:
            raise UserNotFoundSignupRequiredError(
                message="User not found, signup required",
                kakao_id=kakao_id,
            )

        # 클라이언트를 위한 Firebase custom token 생성
        firebase_token = await create_custom_token(user.firebase_id)

        tokens = self._generate_tokens(user)
        return LoginResponse(
            **tokens,
            user=UserResponseDto.from_model(user),
            firebase_custom_token=firebase_token,
        )

    def _generate_tokens(self, user: User) -> dict:
        """JWT 액세스 및 리프레시 토큰 생성"""
        now = datetime.utcnow()

        access_payload = {
            "sub": user.id,
            "type": "access",
            "exp": now + timedelta(minutes=60),
            "iat": now,
        }
        refresh_payload = {
            "sub": user.id,
            "type": "refresh",
            "exp": now + timedelta(days=30),
            "iat": now,
        }

        return {
            "access_token": jwt.encode(access_payload, settings.JWT_SECRET_KEY),
            "refresh_token": jwt.encode(refresh_payload, settings.JWT_SECRET_KEY),
        }
```

## 다중 Repository Service 패턴

```python
class MatchService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self._match_repository = MatchHistoryRepository(session)
        self._user_repository = UserRepository(session)
        self._week_repository = MatchWeekRepository(session)

    async def create_match(
        self,
        dto: MatchHistoryCreateRequest
    ) -> MatchHistoryResponse:
        """도메인 간 유효성 검사와 함께 매치 생성"""
        # 주차 존재 확인
        week = await self._week_repository.get_by_id(dto.week_id)
        if not week:
            raise NotFoundError("Match week not found")

        # 후보 사용자 존재 확인
        if dto.candidate_user_id:
            candidate = await self._user_repository.get_by_id(dto.candidate_user_id)
            if not candidate:
                raise NotFoundError("Candidate user not found")

        # 대상 사용자 존재 확인
        if dto.target_user_id:
            target = await self._user_repository.get_by_id(dto.target_user_id)
            if not target:
                raise NotFoundError("Target user not found")

        # 매치 생성
        match = MatchHistory(**dto.model_dump())
        created = await self._match_repository.create(match)

        return MatchHistoryResponse.from_model(created)
```

## 모범 사례

1. **도메인당 하나의 서비스**: user 도메인에 UserService
2. **세션 주입**: 생성자에서 AsyncSession 수용
3. **DTO 반환**: 라우터에 모델을 직접 반환하지 않음
4. **예외 발생**: 에러에 도메인 예외 사용
5. **비즈니스 규칙**: Repository가 아닌 Service에서 적용
6. **병렬 쿼리**: 독립적인 쿼리에 `asyncio.gather()` 사용
7. **데이터 로더**: 사용자 관계 로딩에 UserDataLoader 사용
8. **Presigned URL**: Repository가 아닌 Service에서 S3 URL 생성
