# FastAPI 백엔드 가이드라인 스킬

## 개요

이 스킬은 YGS (영영사/Youngyeolsa) 프로젝트의 기술 스택에 맞춰 특별히 조정된 포괄적인 백엔드 개발 가이드라인을 제공합니다:

- **FastAPI** (비동기 Python 프레임워크)
- **SQLModel + SQLAlchemy** (ORM)
- **Python 3.12.3** (정확한 버전)
- **PostgreSQL** with asyncpg
- **Domain-Driven Design** 아키텍처
- **계층형 아키텍처** (Router → Service → Repository)
- **ULID** 접두사를 사용한 ID 생성
- **Firebase + Kakao OAuth** 인증

## 이 스킬이 다루는 내용

1. **계층형 아키텍처** - Router → Service → Repository 패턴
2. **API 라우트 & 라우터** - FastAPI 라우터 패턴, 의존성 주입
3. **데이터베이스 & ORM** - SQLModel 모델, 비동기 쿼리, 세션 관리
4. **Domain-Driven Design** - 도메인 구성, 관심사 분리
5. **서비스 레이어** - 비즈니스 로직, 오케스트레이션, 도메인 규칙
6. **Repository 패턴** - 데이터 접근 레이어, BaseRepository 확장, UserDataLoader
7. **DTO & 유효성 검사** - Pydantic DTO, field_validator를 사용한 요청/응답 유효성 검사
8. **Async/Await 패턴** - 비동기 모범 사례, 병렬 쿼리를 위한 asyncio.gather
9. **에러 처리** - 커스텀 예외, 미들웨어 에러 처리
10. **완전한 예시** - 전체 CRUD 도메인 구현

## YGS 특화 패턴

### ULID를 사용한 ID 생성
```python
from ulid import ULID

def generate_user_id() -> str:
    return f"usr_{ULID()}"  # usr_01HQ5K3NXYZ...

def generate_match_history_id() -> str:
    return f"mh_{ULID()}"   # mh_01HQ5K3NXYZ...
```

**엔티티 접두사:**
- `usr_` - User
- `doc_` - UserDocument
- `pho_` - UserPhoto
- `sub_` - UserSubscription
- `aud_` - UserAccessAudit
- `mw_` - MatchWeek
- `mh_` - MatchHistory
- `mf_` - MatchFeedback
- `cs_` - ConsultSchedule

### 읽기/쓰기 세션 분리
```python
# 읽기 작업 (GET 요청)
@router.get("/{user_id}")
async def get_user(
    session: AsyncSession = Depends(get_read_session_dependency),
): ...

# 쓰기 작업 (POST/PATCH/DELETE)
@router.post("")
async def create_user(
    session: AsyncSession = Depends(get_write_session_dependency),
): ...
```

### UserDataLoader를 통한 N+1 방지
```python
# 관계를 포함한 사용자의 병렬 쿼리 로딩
user_with_relations = await self._data_loader.load_user_with_relations(
    user_id,
    load_profile=True,
    load_photos=True,
    load_documents=True,
)
```

### field_validator를 사용한 DTO 유효성 검사
```python
class AdminBasicInfoUpdateRequest(BaseModel):
    status: Optional[str] = Field(None)

    model_config = {"extra": "forbid"}  # 알 수 없는 필드 거부

    @field_validator("status")
    @classmethod
    def validate_status(cls, v: Optional[str]) -> Optional[str]:
        if v is not None:
            valid_values = [e.value for e in UserStatusEnum]
            if v not in valid_values:
                raise ValueError(f"Invalid: {v}")
        return v
```

## 스킬 활성화

스킬은 다음 경우에 활성화되도록 설정되어 있습니다:

### 파일 트리거
- `backend/backend/**/*.py`에서 작업 시
- FastAPI 임포트, 비동기 패턴, SQLModel, 레포지토리를 포함하는 파일

### 프롬프트 트리거
- 키워드: "backend", "FastAPI", "service", "repository", "router", "async", "SQLModel", "domain", "dto"
- 인텐트 패턴: 라우트, 서비스, 레포지토리, 데이터베이스 쿼리 생성/편집

### 강제 적용
- **유형**: 도메인 (제안하나 차단하지 않음)
- **우선순위**: 높음
- 백엔드 코드 작업 시 스킬이 스스로 제안됨

## 프로젝트 구조 대응

스킬은 실제 프로젝트 구조를 참조합니다:

```
backend/
  backend/
    main.py                    # lifespan이 있는 FastAPI 앱

    api/v1/routers/            # 라우터
      admin.py                 # 대시보드, 회원, 매칭 (950+줄)
      auth.py                  # 로그인, 회원가입, Firebase, Kakao OAuth
      match.py                 # 매치 주, 이력, 카드
      user.py                  # 사용자 CRUD, 사진, 문서
      upload.py                # S3 presigned URL

    domain/                    # 도메인
      user/
        model.py               # User, UserProfile, UserLifestyle 등
        repository.py          # UserRepository, UserDataLoader
        service.py             # UserService
        enums.py               # 모든 도메인 열거형
      auth/
        service.py             # AuthService (JWT, Firebase, Kakao)
      admin/
        service.py             # AdminService
        matching_service.py    # 호환성 점수 계산
      match/
        model.py               # MatchWeek, MatchHistory, MatchFeedback
        service.py             # MatchService
      llm/
        matching_service.py    # LLM 향상 매칭
      shared/
        base_repository.py     # 제네릭 BaseRepository

    dtos/                      # DTO
      admin.py                 # 대시보드, 회원 업데이트 DTO
      auth.py                  # OAuth DTO
      match.py                 # 매치 DTO
      user.py                  # 사용자 DTO
      llm_match.py             # LLM 매칭 DTO

    db/
      orm.py                   # 캐싱이 있는 읽기/쓰기 세션 관리

    error/
      __init__.py              # AppException 계층
```

## 통합 상태

- 스킬 디렉토리 생성: `.claude/skills/fastapi-backend-guidelines/`
- YGS 패턴에 맞게 메인 skill.md 업데이트
- FastAPI 패턴이 담긴 리소스 파일 10개
- YGS 특화 패턴 문서화:
  - 접두사를 사용한 ULID ID 생성
  - 읽기/쓰기 세션 분리
  - N+1 방지를 위한 UserDataLoader
  - Firebase/Kakao OAuth
  - field_validator 패턴
  - deleted_at을 사용한 소프트 삭제

## 기술 스택 호환성

- **FastAPI**: 모든 패턴이 FastAPI 라우터와 의존성 사용
- **SQLModel + SQLAlchemy**: 쿼리 패턴 및 모델 정의
- **Async/await**: 모든 예시에서 async를 일관되게 사용
- **Python 3.12.3**: 타입 힌트 및 현대적 Python 패턴
- **PostgreSQL + asyncpg**: 비동기 데이터베이스 작업
- **Domain-Driven Design**: 도메인 구성과 일치
- **계층형 아키텍처**: Router → Service → Repository 패턴
- **Pydantic v2**: field_validator를 사용한 DTO
- **ULID**: 엔티티 접두사를 사용한 ID 생성
- **세션 관리**: `get_read_session_dependency()` 및 `get_write_session_dependency()` 사용

## YGS 주요 도메인

| 도메인 | 설명 | 주요 모델 |
|--------|------|-----------|
| `user` | 사용자 관리 | User, UserProfile, UserLifestyle, UserPreference, UserDocument, UserPhoto, UserSubscription, UserAccessAudit |
| `auth` | 인증 | JWT 토큰, Firebase 소셜 인증, Kakao OAuth |
| `admin` | 관리자 대시보드 | ConsultSchedule, 회원 관리, 통계 |
| `match` | 매칭 시스템 | MatchWeek, MatchHistory, MatchFeedback |
| `llm` | LLM 매칭 | Gemini 향상 호환성 분석 |

## 생성된 파일

```
.claude/skills/fastapi-backend-guidelines/
  ├── skill.md                              # 메인 스킬 개요
  ├── README.md                             # 이 파일
  └── resources/
      ├── layered-architecture.md           # Router → Service → Repository
      ├── api-routes.md                     # FastAPI 라우터 & 엔드포인트
      ├── database-orm.md                   # SQLModel 쿼리 & 모델
      ├── domain-driven-design.md           # 도메인 구성
      ├── service-layer.md                  # 비즈니스 로직 레이어
      ├── repository-pattern.md             # 데이터 접근 레이어
      ├── dtos-validation.md                # Pydantic DTO
      ├── async-patterns.md                 # Async/await 모범 사례
      ├── error-handling.md                 # 커스텀 예외
      └── complete-examples.md              # 전체 CRUD 구현
```

## 핵심 원칙

1. **계층형 아키텍처**: 레이어를 절대 건너뛰지 않음 (Router → Service → Repository)
2. **Domain-Driven Design**: 타입이 아닌 도메인으로 구성
3. **전면적 비동기**: 스택 전반에서 async/await 사용
4. **Repository 패턴**: 레포지토리를 통한 모든 데이터 접근
5. **서비스 레이어**: 라우터나 레포지토리가 아닌 서비스에 비즈니스 로직
6. **API용 DTO**: 요청/응답에 Pydantic DTO 사용
7. **타입 힌트**: 모든 함수에 명시적 타입
8. **에러 처리**: 커스텀 예외, HTTP 매핑을 위한 미들웨어
9. **읽기/쓰기 분리**: 읽기와 쓰기 작업에 별도 세션
10. **의존성 주입**: 세션에 FastAPI의 Depends() 사용
11. **ULID ID**: 엔티티 접두사를 사용한 ULID (usr_, mw_, mh_ 등)
12. **소프트 삭제**: 하드 삭제 대신 deleted_at 타임스탬프 사용
13. **N+1 방지**: asyncio.gather 및 DataLoader 패턴 사용

---

**상태**: YGS 특화 패턴으로 완전 통합
**업데이트**: 2026-01-14
**프로젝트**: YGS (영영사/Youngyeolsa) - 전문 매칭 플랫폼
