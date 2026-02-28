# DTO & 유효성 검사 - Pydantic

## DTO (Data Transfer Object)

DTO는 Pydantic v2를 사용하여 API 계약을 정의합니다.

### field_validator가 있는 요청 DTO

```python
# backend/dtos/admin.py
from pydantic import BaseModel, Field, field_validator
from typing import Optional
from datetime import datetime

from backend.domain.user.enums import (
    UserStatusEnum,
    GenderEnum,
    EducationEnum,
    SalaryRangeEnum,
)


class AdminBasicInfoUpdateRequest(BaseModel):
    """사용자 기본 정보 수정 (관리자 액션)"""
    name: Optional[str] = Field(None, max_length=50)
    phone: Optional[str] = Field(None, max_length=15)
    status: Optional[str] = None
    gender: Optional[str] = None

    # 알 수 없는 필드 거부
    model_config = {"extra": "forbid"}

    @field_validator("status")
    @classmethod
    def validate_status(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in UserStatusEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid status: {v}. Valid: {valid_values}")
        return v

    @field_validator("gender")
    @classmethod
    def validate_gender(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in GenderEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid gender: {v}. Valid: {valid_values}")
        return v

    @field_validator("phone")
    @classmethod
    def validate_phone(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            # 숫자만 추출하고 한국 전화번호 형식 유효성 검사
            digits = "".join(c for c in v if c.isdigit())
            if not digits.startswith("01") or len(digits) < 10:
                raise ValueError("Invalid phone number format")
            return digits
        return v
```

### from_model이 있는 응답 DTO

```python
# backend/dtos/admin.py
from datetime import datetime
from typing import Optional, List

class MemberSummaryResponse(BaseModel):
    """목록 보기를 위한 회원 요약"""
    id: str
    name: str
    phone: str
    gender: str
    birth_year: int
    status: str
    created_at: datetime

    # 선택적 프로필 정보
    height: Optional[int] = None
    education: Optional[str] = None
    job: Optional[str] = None
    district: Optional[str] = None

    # 사진 URL (presigned)
    photo_url: Optional[str] = None

    @classmethod
    def from_model(
        cls,
        user: User,
        profile: Optional[UserProfile] = None,
        photo_url: Optional[str] = None,
    ) -> "MemberSummaryResponse":
        """도메인 모델을 DTO로 변환"""
        return cls(
            id=user.id,
            name=user.name,
            phone=user.phone,
            gender=user.gender,
            birth_year=user.birth_year,
            status=user.status,
            created_at=user.created_at,
            height=profile.height if profile else None,
            education=profile.education if profile else None,
            job=profile.job if profile else None,
            district=profile.district if profile else None,
            photo_url=photo_url,
        )

    model_config = {"from_attributes": True}


class MemberListResponse(BaseModel):
    """페이지네이션된 회원 목록"""
    members: List[MemberSummaryResponse]
    total: int
    page: int
    page_size: int
    total_pages: int
```

### 프로필 수정이 있는 복잡한 요청 DTO

```python
# backend/dtos/admin.py
class AdminProfileUpdateRequest(BaseModel):
    """사용자 프로필 수정 (관리자 액션)"""
    height: Optional[int] = Field(None, ge=100, le=250)
    education: Optional[str] = None
    university: Optional[str] = Field(None, max_length=100)
    job: Optional[str] = Field(None, max_length=100)
    salary_range: Optional[str] = None
    district: Optional[str] = Field(None, max_length=50)
    mbti: Optional[str] = Field(None, max_length=4)
    about_me: Optional[str] = Field(None, max_length=1000)
    profile_appeal: Optional[str] = Field(None, max_length=500)

    model_config = {"extra": "forbid"}

    @field_validator("education")
    @classmethod
    def validate_education(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in EducationEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid education: {v}. Valid: {valid_values}")
        return v

    @field_validator("salary_range")
    @classmethod
    def validate_salary_range(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in SalaryRangeEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid salary_range: {v}. Valid: {valid_values}")
        return v

    @field_validator("mbti")
    @classmethod
    def validate_mbti(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            v = v.upper()
            valid_types = [
                "INTJ", "INTP", "ENTJ", "ENTP",
                "INFJ", "INFP", "ENFJ", "ENFP",
                "ISTJ", "ISFJ", "ESTJ", "ESFJ",
                "ISTP", "ISFP", "ESTP", "ESFP",
            ]
            if v not in valid_types:
                raise ValueError(f"Invalid MBTI type: {v}")
            return v
        return v
```

## 유효성 검사 패턴

### 필드 제약 조건

```python
from pydantic import Field, field_validator

class UserCreateDto(BaseModel):
    # 길이 제약
    name: str = Field(min_length=1, max_length=50)

    # 숫자 제약
    birth_year: int = Field(ge=1950, le=2010)
    height: Optional[int] = Field(None, ge=100, le=250)

    # 전화번호 패턴 (한국 휴대폰)
    phone: str = Field(pattern=r'^01[0-9]\d{7,8}$')

    # 커스텀 유효성 검사기
    @field_validator('name')
    @classmethod
    def validate_name(cls, v: str) -> str:
        if not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()
```

### 열거형 유효성 검사 패턴

```python
# backend/dtos/match.py
from backend.domain.user.enums import MatchCategoryEnum

class MatchHistoryCreateRequest(BaseModel):
    week_id: str
    candidate_user_id: Optional[str] = None
    target_user_id: Optional[str] = None
    category: MatchCategoryEnum = Field(
        default=MatchCategoryEnum.INTRO,
        description="매치 카테고리: intro 또는 extra",
    )
    bidirectional: bool = Field(
        default=False,
        description="양방향 매칭 여부",
    )
```

### 교차 필드 유효성 검사를 위한 model_validator

```python
from pydantic import model_validator

class MatchWeekCreateRequest(BaseModel):
    year: int = Field(ge=2020, le=2100)
    week_number: int = Field(ge=1, le=53)
    label: str = Field(max_length=50)
    start_time: datetime
    end_time: datetime

    @model_validator(mode='after')
    def check_dates(self) -> 'MatchWeekCreateRequest':
        if self.end_time <= self.start_time:
            raise ValueError('end_time must be after start_time')
        return self
```

## 중첩 DTO

```python
# backend/dtos/match.py
class MatchUserSummary(BaseModel):
    """매치 컨텍스트에서 사용자 정보를 위한 중첩 DTO"""
    user_id: Optional[str] = None
    firebase_id: Optional[str] = None
    name: Optional[str] = None
    gender: Optional[GenderEnum] = None
    phone: Optional[str] = None


class MatchHistoryResponse(BaseModel):
    """중첩된 사용자 정보를 포함한 매치 이력"""
    id: str
    week_id: Optional[str] = None

    # 중첩 DTO
    candidate: MatchUserSummary
    target: MatchUserSummary

    category: MatchCategoryEnum
    matched_at: datetime
    target_selected: bool
    created_at: datetime
    updated_at: datetime
```

## 대시보드 통계 DTO

```python
# backend/dtos/admin.py
class DashboardStatsResponse(BaseModel):
    """대시보드 통계"""
    total_members: int = Field(description="총 등록 회원 수")
    monthly_signups: int = Field(description="최근 30일 가입자 수")
    weekly_signups: int = Field(description="최근 7일 가입자 수")
    today_signups: int = Field(description="오늘 가입자 수")
    male_count: int = Field(description="총 남성 회원 수")
    female_count: int = Field(description="총 여성 회원 수")
    pending_reviews: int = Field(description="검토 대기 중인 회원 수")


class GenderRatioResponse(BaseModel):
    """차트를 위한 성별 비율"""
    male_count: int
    female_count: int
    male_percentage: float
    female_percentage: float


class WeeklyTrendItem(BaseModel):
    """단일 주별 트렌드 데이터 포인트"""
    week_start: datetime
    week_label: str
    count: int


class WeeklyTrendResponse(BaseModel):
    """주별 등록 트렌드"""
    data: List[WeeklyTrendItem]
```

## 라우트에서의 사용

```python
@router.patch("/members/{user_id}/basic", response_model=MemberDetailResponse)
async def update_member_basic_info(
    user_id: str,
    dto: AdminBasicInfoUpdateRequest,  # 요청 본문 자동 유효성 검사
    session: AsyncSession = Depends(get_write_session_dependency),
):
    """회원 기본 정보 수정"""
    service = AdminService(session)
    return await service.update_member_basic_info(user_id, dto)
```

## 모범 사례

1. **요청/응답 분리**: 입출력에 다른 DTO 사용
2. **field_validator**: 열거형 및 커스텀 유효성 검사에 사용
3. **model_config = {"extra": "forbid"}**: 알 수 없는 필드 거부
4. **from_model 메서드**: 모델을 응답 DTO로 변환
5. **타입 힌트**: 모든 필드에 명시적 타입
6. **필드 설명**: Field(description=...)로 문서화
7. **비즈니스 로직 없음**: DTO는 데이터 컨테이너만
8. **중첩 DTO**: 복잡한 중첩 데이터 구조에 사용
9. **model_validator**: 교차 필드 유효성 검사에 사용
10. **열거형 타입**: 도메인 열거형을 직접 사용하거나 문자열 유효성 검사
