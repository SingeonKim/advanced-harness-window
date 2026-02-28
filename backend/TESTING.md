# 백엔드 테스팅 가이드

**마지막 업데이트**: 2025-11-03

이 가이드는 백엔드 서비스의 테스트 작성 및 실행 방법을 설명합니다.

---

## 목차

1. [빠른 시작](#빠른-시작)
2. [테스트 실행](#테스트-실행)
3. [서비스 테스트 작성](#서비스-테스트-작성)
4. [테스트 데이터 팩토리](#테스트-데이터-팩토리)
5. [모킹 전략](#모킹-전략)
6. [커버리지 가이드라인](#커버리지-가이드라인)
7. [문제 해결](#문제-해결)

---

## 빠른 시작

```bash
# 가상 환경 활성화
source .venv/bin/activate

# 모든 테스트 실행
pytest

# 커버리지 보고서와 함께 테스트 실행
pytest --cov=backend --cov-report=html
open htmlcov/index.html  # 커버리지 보고서 보기

# 특정 테스트 파일 실행
pytest tests/unit/domain/user/test_service.py -v

# 특정 테스트 메서드 실행
pytest tests/unit/domain/user/test_service.py::TestUserService::test_is_admin_user -v
```

---

## 테스트 실행

### 모든 테스트 실행

```bash
pytest
```

### 특정 테스트 디렉토리 실행

```bash
# 모든 서비스 테스트
pytest tests/unit/domain/*/test_service.py

# 특정 도메인
pytest tests/unit/domain/artist/

# 모든 단위 테스트
pytest tests/unit/
```

### 커버리지와 함께 실행

```bash
# 터미널 보고서
pytest --cov=backend --cov-report=term-missing

# HTML 보고서 (브라우저에서 열림)
pytest --cov=backend --cov-report=html
open htmlcov/index.html
```

### 실패한 테스트만 실행

```bash
# 마지막 실행에서 실패한 테스트만 재실행
pytest --lf

# 실패한 테스트를 먼저 실행 후 나머지
pytest --ff
```

### 상세 출력

```bash
# 테스트 이름 표시
pytest -v

# print 구문 표시
pytest -s

# 둘 다
pytest -vs
```

---

## 서비스 테스트 작성

### 기본 테스트 구조

```python
# tests/unit/domain/user/test_service.py
import pytest
from unittest.mock import AsyncMock, patch

from backend.domain.user.service import UserService
from backend.domain.user.model import User
from tests.factories import UserFactory


@pytest.mark.asyncio
class TestUserService:
    """UserService 테스트 스위트."""

    @patch('backend.domain.user.service.UserRepository')
    async def test_is_admin_user_returns_true_for_admin(
        self,
        mock_user_repo_class,
        mock_session,
        mock_user_repository,
    ):
        """is_admin_user가 관리자 사용자에게 True를 반환하는지 테스트."""
        # Arrange: 픽스처를 반환하도록 mock 설정
        mock_user_repo_class.return_value = mock_user_repository
        mock_user_repository.get_async.return_value = UserFactory.create_admin()

        # 서비스 생성 (자동으로 mock된 레포지토리 사용)
        service = UserService(mock_session)

        # Act
        result = await service.is_admin_user("user-1")

        # Assert
        assert result is True
        mock_user_repository.get_async.assert_called_once_with(id="user-1")
```

### 주요 패턴

#### 1. 레포지토리 모킹에 @patch 데코레이터 사용

**권장**:
```python
@patch('backend.domain.artist.service.ArtistRepository')
async def test_update_artist(mock_repo_class, mock_session, mock_artist_repository):
    mock_repo_class.return_value = mock_artist_repository
    service = ArtistService(mock_session)  # 자동으로 mock 사용
```

**비권장**:
```python
# ❌ 수동 주입은 올바르게 작동하지 않음
service = ArtistService(mock_session)
service._artist_repository = mock_artist_repository  # 너무 늦음!
```

#### 2. 정의 위치가 아닌 import 위치에서 Patch

**권장**:
```python
# IMPORT된 곳(서비스 모듈)에서 Mock
@patch('backend.domain.artwork.service.PDFGenerator')
```

**비권장**:
```python
# ❌ 잘못됨: 사용 위치가 아닌 정의를 patch함
@patch('backend.utils.pdf.PDFGenerator')
```

#### 3. 비동기 메서드에 AsyncMock 사용

**권장**:
```python
mock_repo.get_async = AsyncMock(return_value=artist)
result = await service.get_artist(artist_id="123")
```

**비권장**:
```python
# ❌ 일반 Mock은 await와 함께 작동하지 않음
mock_repo.get_async = MagicMock(return_value=artist)
```

---

## 테스트 데이터 팩토리

팩토리는 합리적인 기본값으로 실제적인 테스트 데이터를 제공합니다.

### 팩토리 사용

```python
from tests.factories import ArtistFactory, UserFactory, ArtworkFactory

# 기본값으로 생성
artist_dto = ArtistFactory.create_request_dto()
user = UserFactory.create_model()

# 특정 필드 재정의
artist_dto = ArtistFactory.create_request_dto(
    name_ko="커스텀 작가",
    host_name="custom-artist",
)

# 관리자 사용자 생성
admin = UserFactory.create_admin(email="admin@example.com")
```

### 사용 가능한 팩토리

- `ArtistFactory`: `create_request_dto()`, `create_response_dto()`, `create_model()`
- `ArtworkFactory`: `create_request_dto()`, `create_response_dto()`, `create_model()`
- `ExhibitionFactory`: `create_info_dto()`
- `UserFactory`: `create_model()`, `create_admin()`

### 새 팩토리 생성

```python
# tests/factories.py
class MyEntityFactory:
    @staticmethod
    def create_model(**overrides):
        defaults = {
            "id": "entity_123",
            "name": "Default Name",
            # ... 기본값이 있는 모든 필수 필드
        }
        defaults.update(overrides)
        return MyEntity(**defaults)
```

---

## 모킹 전략

### 레포지토리 메서드 Mock

```python
# 반환값 설정
mock_artist_repository.get_async.return_value = artist

# 다른 반환값으로 여러 호출 설정
mock_artist_repository.get_async.side_effect = [artist1, artist2, None]

# 메서드가 호출됐는지 검증
mock_artist_repository.get_async.assert_called_once_with(id="artist-1")

# 메서드가 호출되지 않았는지 검증
mock_artist_repository.create_async.assert_not_called()
```

### 외부 유틸리티 Mock

모듈 레벨 import(PDFGenerator, S3 함수)의 경우:

```python
@patch('backend.domain.artwork.service.PDFGenerator')
@patch('backend.domain.artwork.service.push_outputs')
@pytest.mark.asyncio
async def test_generate_pdf(mock_push, mock_pdf_class):
    # PDF 생성기 mock 설정
    mock_pdf_instance = MagicMock()
    mock_pdf_instance.generate_portfolio_pdf = AsyncMock(
        return_value=b'fake pdf bytes'
    )
    mock_pdf_class.return_value = mock_pdf_instance

    # S3 mock 설정
    mock_push.return_value = ['https://s3.url/file.pdf']

    # 서비스 메서드 테스트
    service = ArtworkService(mock_session)
    result = await service.generate_portfolio_to_pdf(...)

    # 검증
    mock_pdf_instance.generate_portfolio_pdf.assert_called_once()
    mock_push.assert_called_once()
```

### 공유 픽스처 사용

모든 도메인 테스트는 다음 픽스처에 접근 가능합니다 (`tests/unit/domain/conftest.py`에서):

- `mock_session`: Mock AsyncSession
- `mock_artist_repository`
- `mock_artwork_repository`
- `mock_exhibition_repository`
- `mock_user_repository`
- `mock_direct_message_repository`
- `mock_notification_repository`
- `mock_subscription_repository`

---

## 커버리지 가이드라인

### 목표 커버리지

- **전체**: 모든 서비스 파일에서 80% 이상
- **단순 서비스** (User, Subscription): 90-100% 목표
- **복잡한 서비스** (Artist, Artwork): 80% 이상 허용

### 커버리지 확인

```bash
# 커버리지와 함께 테스트 실행
pytest --cov=backend/domain --cov-report=term-missing

# HTML 보고서 생성
pytest --cov=backend/domain --cov-report=html
open htmlcov/index.html
```

### 커버리지 제외 항목

다음은 자동으로 제외됩니다 (`pyproject.toml`에 설정):

- `*/__init__.py`
- `*/tests/*`
- `if __name__ == "__main__":`
- `if TYPE_CHECKING:`
- `raise NotImplementedError`
- `def __repr__`

### 테스트해야 할 것

**테스트 해야 함**:
- ✅ 비즈니스 로직 (유효성 검사, 계산)
- ✅ 에러 처리 (NotFoundError, ValueError)
- ✅ 엣지 케이스 (빈 리스트, None 값)
- ✅ 메서드 상호작용 (서비스 → 레포지토리)

**테스트하지 않아도 됨**:
- ❌ Pydantic 모델 유효성 검사 (Pydantic이 테스트)
- ❌ SQLModel ORM 동작 (SQLModel이 테스트)
- ❌ 로직 없는 단순 getter/setter
- ❌ 서드파티 라이브러리 동작

---

## 문제 해결

### Import 에러

**문제**: `ModuleNotFoundError: No module named 'backend'`

**해결**: 백엔드 가상 환경에 있는지 확인:
```bash
cd backend
source .venv/bin/activate
uv sync --group dev
```

### AsyncMock 문제

**문제**: `TypeError: object MagicMock can't be used in 'await' expression`

**해결**: 비동기 메서드에 `AsyncMock` 사용:
```python
# ✅ 올바름
mock_repo.get_async = AsyncMock(return_value=value)

# ❌ 잘못됨
mock_repo.get_async = MagicMock(return_value=value)
```

### Patch가 작동하지 않는 경우

**문제**: 서비스에서 Mock이 사용되지 않음

**해결**: 정의된 위치가 아닌 import된 위치에서 Patch:
```python
# ✅ 올바름: 서비스 모듈에서 Patch
@patch('backend.domain.artist.service.ArtistRepository')

# ❌ 잘못됨: 정의를 Patch함
@patch('backend.domain.artist.repository.ArtistRepository')
```

### 픽스처를 찾을 수 없는 경우

**문제**: `fixture 'mock_artist_repository' not found`

**해결**: 올바른 테스트 디렉토리 구조인지 확인:
```
tests/
└── unit/
    └── domain/
        ├── conftest.py        # ← 픽스처가 정의된 곳
        └── artist/
            └── test_service.py  # ← 픽스처를 사용할 수 있는 곳
```

### 커버리지가 낮은 경우

**문제**: 커버리지가 80% 미만

**해결**:
1. 커버되지 않은 라인 확인:
   ```bash
   pytest --cov=backend/domain --cov-report=term-missing
   ```
2. 커버되지 않은 라인에 테스트 추가
3. 또는 정말 테스트 불가능한 경우 코드를 제외로 표시:
   ```python
   if condition:  # pragma: no cover
       raise NotImplementedError("Future feature")
   ```

---

## 모범 사례

1. **동작당 하나의 테스트**: 각 테스트는 하나의 특정 동작을 검증해야 함
2. **Arrange-Act-Assert**: 명확한 설정, 실행, 검증으로 테스트 구조화
3. **설명적인 이름**: 테스트 이름이 테스트 내용을 설명해야 함
   - ✅ `test_update_artist_raises_error_when_host_name_exists`
   - ❌ `test_update_artist_2`
4. **팩토리 사용**: 테스트 데이터를 수동으로 생성하지 말 것
5. **모두 비동기**: 모든 서비스 메서드는 비동기이므로 테스트도 마찬가지
6. **깔끔한 Mock**: 테스트 간 mock 초기화 (pytest가 픽스처로 자동 처리)

---

## 예시

다음 파일에서 예시 확인:
- `tests/test_factories.py` - 팩토리 사용 예시
- `tests/unit/domain/test_fixtures.py` - 픽스처 사용 예시
- `dev/active/backend-service-testing/backend-service-testing-context.md` - 상세 패턴

---

**질문이 있으신가요?** `dev/active/backend-service-testing/`의 계획을 확인하거나 팀에 문의하세요!
