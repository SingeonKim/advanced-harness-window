# 에러 처리 - FastAPI

## 커스텀 예외

YGS는 도메인 특화 예외를 정의합니다:

```python
# backend/error/__init__.py

class AppException(Exception):
    """모든 애플리케이션 에러의 기반 예외"""
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code
        super().__init__(self.message)


class NotFoundError(AppException):
    """리소스를 찾을 수 없음 (HTTP 404)"""
    def __init__(self, message: str = "Resource not found"):
        super().__init__(message, status_code=404)


class ValidationError(AppException):
    """유효성 검사 실패 (HTTP 400)"""
    def __init__(self, message: str = "Validation failed"):
        super().__init__(message, status_code=400)


class UnauthorizedError(AppException):
    """인증되지 않은 접근 (HTTP 401)"""
    def __init__(self, message: str = "Unauthorized"):
        super().__init__(message, status_code=401)


class ForbiddenError(AppException):
    """금지된 접근 (HTTP 403)"""
    def __init__(self, message: str = "Forbidden"):
        super().__init__(message, status_code=403)


class ConflictError(AppException):
    """리소스 충돌 (HTTP 409)"""
    def __init__(self, message: str = "Conflict"):
        super().__init__(message, status_code=409)


class UserNotFoundSignupRequiredError(AppException):
    """회원가입이 필요한 OAuth 흐름을 위한 특수 예외"""
    def __init__(
        self,
        message: str = "User not found, signup required",
        firebase_id: str = None,
        kakao_id: str = None,
        email: str = None,
    ):
        super().__init__(message, status_code=404)
        self.firebase_id = firebase_id
        self.kakao_id = kakao_id
        self.email = email
```

## 서비스에서 예외 사용

```python
# backend/domain/user/service.py
from backend.error import NotFoundError, ValidationError, ConflictError

class UserService:
    async def get_user(self, user_id: str) -> UserResponseDto:
        """ID로 사용자 조회"""
        user = await self._repository.get_by_id(user_id)

        # 도메인 예외 발생
        if not user:
            raise NotFoundError(f"User {user_id} not found")

        return UserResponseDto.from_model(user)

    async def create_user(self, dto: UserCreateDto) -> UserResponseDto:
        """비즈니스 유효성 검사와 함께 사용자 생성"""
        # 중복 전화번호 확인
        existing = await self._repository.find_by_phone(dto.phone)
        if existing:
            raise ConflictError("Phone number already registered")

        # 비즈니스 유효성 검사
        if dto.birth_year > 2010:
            raise ValidationError("User must be at least 14 years old")

        user = User(**dto.model_dump())
        return await self._repository.create(user)


# backend/domain/auth/service.py
from backend.error import UnauthorizedError, UserNotFoundSignupRequiredError

class AuthService:
    async def login_with_firebase(self, dto: FirebaseLoginRequest) -> LoginResponse:
        """회원가입 리다이렉트가 있는 Firebase 로그인"""
        decoded = await verify_firebase_token(dto.id_token)
        firebase_id = decoded["uid"]

        user = await self._user_repository.find_by_firebase_id(firebase_id)
        if not user:
            # 특수 예외가 회원가입을 위한 OAuth 정보를 전달
            raise UserNotFoundSignupRequiredError(
                message="User not found, signup required",
                firebase_id=firebase_id,
                email=decoded.get("email"),
            )

        return self._generate_login_response(user)

    async def verify_token(self, token: str) -> User:
        """JWT 토큰 검증"""
        try:
            payload = jwt.decode(token, settings.JWT_SECRET_KEY, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            raise UnauthorizedError("Token has expired")
        except jwt.InvalidTokenError:
            raise UnauthorizedError("Invalid token")

        user = await self._user_repository.get_by_id(payload["sub"])
        if not user:
            raise UnauthorizedError("User not found")

        return user
```

## 예외 핸들러 (권장)

> **경고**: 에러 처리에 `BaseHTTPMiddleware`를 사용하지 마세요. FastAPI의 의존성 주입과 충돌하여
> SQLAlchemy 비동기 세션과 함께 `IllegalStateChangeError`를 일으킵니다.
> 대신 `@app.exception_handler` 데코레이터를 사용하세요.

```python
# backend/middleware/error_handler.py
import logging

from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from sqlalchemy.exc import DBAPIError, IntegrityError

from backend.error import (
    AppException,
    ConflictError,
    ForbiddenError,
    NotFoundError,
    UnauthorizedError,
    UserNotFoundSignupRequiredError,
    ValidationError,
)

logger = logging.getLogger(__name__)


def register_exception_handlers(app: FastAPI) -> None:
    """FastAPI 앱에 모든 예외 핸들러를 등록합니다."""

    @app.exception_handler(NotFoundError)
    async def not_found_handler(request: Request, exc: NotFoundError) -> JSONResponse:
        logger.info(f"Not found: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_404_NOT_FOUND,
            content={"detail": exc.message},
        )

    @app.exception_handler(ValidationError)
    async def validation_error_handler(
        request: Request, exc: ValidationError
    ) -> JSONResponse:
        logger.info(f"Validation error: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_400_BAD_REQUEST,
            content={"detail": exc.message},
        )

    @app.exception_handler(UnauthorizedError)
    async def unauthorized_handler(
        request: Request, exc: UnauthorizedError
    ) -> JSONResponse:
        logger.info(f"Unauthorized: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_401_UNAUTHORIZED,
            content={"detail": exc.message},
        )

    @app.exception_handler(UserNotFoundSignupRequiredError)
    async def signup_required_handler(
        request: Request, exc: UserNotFoundSignupRequiredError
    ) -> JSONResponse:
        logger.info(f"User not found, signup required: {request.url}")
        content: dict[str, str] = {
            "detail": exc.message,
            "error_code": "USER_NOT_FOUND_SIGNUP_REQUIRED",
        }
        if exc.firebase_email is not None:
            content["firebase_email"] = exc.firebase_email
        if exc.firebase_name is not None:
            content["firebase_name"] = exc.firebase_name
        if exc.firebase_provider is not None:
            content["firebase_provider"] = exc.firebase_provider

        return JSONResponse(
            status_code=452,
            content=content,
        )

    @app.exception_handler(ForbiddenError)
    async def forbidden_handler(request: Request, exc: ForbiddenError) -> JSONResponse:
        logger.warning(f"Forbidden: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_403_FORBIDDEN,
            content={"detail": exc.message},
        )

    @app.exception_handler(ConflictError)
    async def conflict_handler(request: Request, exc: ConflictError) -> JSONResponse:
        logger.info(f"Conflict: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT,
            content={"detail": exc.message},
        )

    @app.exception_handler(AppException)
    async def app_exception_handler(
        request: Request, exc: AppException
    ) -> JSONResponse:
        logger.error(f"Application error: {request.url} - {exc.message}")
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={"detail": exc.message},
        )

    @app.exception_handler(IntegrityError)
    async def integrity_error_handler(
        request: Request, exc: IntegrityError
    ) -> JSONResponse:
        logger.warning(f"Database integrity error: {request.url} - {str(exc)}")
        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT,
            content={"detail": "Database constraint violation"},
        )

    @app.exception_handler(DBAPIError)
    async def dbapi_error_handler(request: Request, exc: DBAPIError) -> JSONResponse:
        if "ConnectionDoesNotExistError" in str(exc) or "connection was closed" in str(
            exc
        ):
            logger.debug(f"Client disconnected during request: {request.url}")
            return JSONResponse(status_code=499, content={"detail": "Client Closed Request"})

        logger.exception(f"Database error during request to {request.url}: {exc}")
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={"detail": "Database error"},
        )

    @app.exception_handler(Exception)
    async def general_exception_handler(
        request: Request, exc: Exception
    ) -> JSONResponse:
        logger.exception(
            f"Unexpected error during request to {request.url}: {exc}",
            exc_info=True,
        )
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={"detail": "Internal server error"},
        )
```

## 예외 핸들러 등록

```python
# backend/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from backend.middleware.error_handler import register_exception_handlers

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 시작
    yield
    # 종료

def create_application() -> FastAPI:
    app = FastAPI(
        title="YGS API",
        lifespan=lifespan,
    )

    # 예외 핸들러 등록 (미들웨어 대신!)
    register_exception_handlers(app)

    # 라우터 등록
    app.include_router(auth_router)
    app.include_router(user_router)
    app.include_router(admin_router)
    app.include_router(match_router)

    return app

app = create_application()
```

## BaseHTTPMiddleware를 사용하지 않는 이유

`BaseHTTPMiddleware`는 FastAPI의 의존성 주입 라이프사이클과 충돌하는 내부 스트리밍 패턴을 사용합니다.
SQLAlchemy 비동기 세션을 사용할 때:

```
# 이것은 IllegalStateChangeError를 일으킵니다:
# Method 'close()' can't be called here; method '_connection_for_bind()'
# is already in progress

class ErrorHandlerMiddleware(BaseHTTPMiddleware):  # 이렇게 하지 마세요
    async def dispatch(self, request, call_next):
        try:
            return await call_next(request)
        except Exception as e:
            return JSONResponse(...)
```

문제는 다음과 같이 발생합니다:
1. `call_next()`는 요청 처리에 내부 스트림을 사용
2. 의존성 정리 (session.close()) 중 예외가 발생할 때
3. 미들웨어가 세션이 아직 중간 상태에 있는 동안 처리하려고 시도

**해결책**: FastAPI의 요청 라이프사이클과 올바르게 통합되는 `@app.exception_handler` 데코레이터를 사용하세요.

## FastAPI HTTPException (간단한 경우)

라우터의 간단한 경우에는 HTTPException을 직접 사용할 수 있습니다:

```python
from fastapi import HTTPException, status

@router.get("/{id}")
async def get_item(id: str):
    item = await get_item_from_db(id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found"
        )
    return item
```

그러나 일관성을 위해 **서비스에서는 도메인 예외를 선호하세요**.

## 라우터 수준 예외 처리

```python
# backend/api/v1/routers/admin.py
from fastapi import APIRouter, HTTPException
from backend.error import NotFoundError, ForbiddenError

router = APIRouter(prefix="/api/v1/admin", tags=["admin"])

@router.get("/members/{user_id}")
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
    except ForbiddenError as e:
        raise HTTPException(status_code=403, detail=str(e))
```

## 유효성 검사 에러 (Pydantic)

Pydantic은 유효성 검사를 자동으로 처리하고 422를 반환합니다:

```python
from pydantic import BaseModel, field_validator
from backend.domain.user.enums import UserStatusEnum

class AdminBasicInfoUpdateRequest(BaseModel):
    status: Optional[str] = None

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

유효성 검사 실패 시 FastAPI가 반환하는 응답:
```json
{
  "detail": [
    {
      "loc": ["body", "status"],
      "msg": "Invalid status: xyz. Valid: ['pending', 'approved', 'rejected']",
      "type": "value_error"
    }
  ]
}
```

## 모범 사례

1. **예외 핸들러**: `@app.exception_handler` 사용 (`BaseHTTPMiddleware` 대신)
2. **도메인 예외**: 서비스에서 커스텀 예외 사용
3. **구체적인 에러**: NotFoundError, ValidationError, ConflictError 등
4. **HTTP 매핑**: 핸들러가 HTTP 응답으로 변환
5. **로깅**: 컨텍스트와 함께 예상치 못한 에러 로그
6. **일관된 형식**: 같은 에러 응답 구조
7. **특수 예외**: OAuth 흐름에 UserNotFoundSignupRequiredError 사용
8. **Pydantic 유효성 검사**: Pydantic이 DTO 유효성 검사 처리하도록
9. **Extra forbid**: 알 수 없는 필드 거부에 `model_config = {"extra": "forbid"}` 사용
10. **미들웨어 회피**: 비동기 DB 세션과 함께 `BaseHTTPMiddleware` 절대 사용하지 않음
