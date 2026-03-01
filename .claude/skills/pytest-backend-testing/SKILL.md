---
name: pytest-backend-testing
description: Comprehensive pytest testing guide for FastAPI backends. Covers unit testing, integration testing, async patterns, mocking, fixtures, coverage, and FastAPI-specific testing with TestClient. Use when writing or updating test code for backend services, repositories, or API routes.
---

# Pytest 백엔드 테스팅 가이드라인

## 목적

pytest, pytest-asyncio, FastAPI TestClient를 사용하여 FastAPI 백엔드 애플리케이션의 포괄적인 테스트를 작성하기 위한 완전한 가이드. async 테스팅, 적절한 모킹, 계층적 테스팅(repository → service → router), 높은 테스트 커버리지 달성을 강조합니다.

## 이 스킬을 사용하는 경우

- 백엔드 코드의 새 테스트 파일 작성
- repository, service, API route 테스팅
- 테스트 fixture 및 mock 설정
- 실패하는 테스트 디버깅
- 테스트 커버리지 개선
- pytest-asyncio를 사용한 async 테스트 작성
- 데이터베이스 작업 테스팅
- route 테스팅을 위한 FastAPI TestClient 사용

---

## 빠른 시작

### 새 테스트 파일 체크리스트

새 코드에 대한 테스트 작성 시 이 체크리스트를 따르세요:

- [ ] 테스트 파일 생성: `tests/unit/{domain}/test_{module}.py`
- [ ] pytest 및 pytest-asyncio import
- [ ] 필요한 fixture 설정 (session, client 등)
- [ ] async 테스트에 `@pytest.mark.asyncio` 사용
- [ ] AAA 패턴 따르기: Arrange, Act, Assert
- [ ] 외부 의존성 모킹
- [ ] 성공 케이스와 에러 케이스 모두 테스트
- [ ] 커버리지가 80% 임계값을 충족하는지 확인
- [ ] 설명적인 테스트 이름 사용: `test_<what>_<when>_<expected>`

### 테스트 커버리지 체크리스트

좋은 커버리지 확보를 위해 다음을 확인하세요:

- [ ] 모든 public 메서드/함수 테스트
- [ ] 에러 처리 및 예외 테스트
- [ ] 엣지 케이스와 경계 조건 테스트
- [ ] 유효성 검사 로직 테스트
- [ ] 외부 의존성 모킹 (데이터베이스, API)
- [ ] async/await 동작 확인
- [ ] `pytest --cov=backend --cov-report=term-missing` 실행
- [ ] 커버리지 보고서에서 누락 부분 확인
- [ ] 80%+ 커버리지 목표

---

## 프로젝트 테스팅 구조

qwarty 백엔드 테스팅 구조:

```
backend/
  tests/
    conftest.py              # 전역 fixture
    unit/
      domain/
        artist/
          test_artist_repository.py
          test_artist_service.py
        artwork/
        auth/
        ...
      middleware/
        test_error_handler.py
      utils/
        test_utils.py
    integration/             # 엔드-투-엔드 테스트
      test_artist_api.py
      test_auth_flow.py
```

---

## 공통 테스트 패턴 빠른 참조

### 기본 Async 테스트

```python
import pytest
from sqlmodel.ext.asyncio.session import AsyncSession

@pytest.mark.asyncio
async def test_get_artist_by_id(db_session: AsyncSession):
    # Arrange
    artist_id = "test-artist-id"

    # Act
    result = await repository.get_by_id(artist_id)

    # Assert
    assert result is not None
    assert result.id == artist_id
```

### 데이터베이스 세션 모킹

```python
from unittest.mock import AsyncMock, MagicMock

@pytest.mark.asyncio
async def test_create_artist_success():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_session.execute = AsyncMock()
    mock_session.commit = AsyncMock()

    # Act
    service = ArtistService(mock_session)
    result = await service.create_artist(data)

    # Assert
    assert mock_session.commit.called
```

### FastAPI Route 테스팅

```python
from fastapi.testclient import TestClient
from backend.main import create_application

@pytest.fixture
def client():
    app = create_application()
    return TestClient(app)

def test_get_artist_endpoint(client):
    # Act
    response = client.get("/api/v1/artists/test-id")

    # Assert
    assert response.status_code == 200
    assert response.json()["id"] == "test-id"
```

---

## 테스트 구성 원칙

### 테스트 구조 (AAA 패턴)

1. **Arrange**: 테스트 데이터, mock, fixture 설정
2. **Act**: 테스트할 코드 실행
3. **Assert**: 예상 결과 확인

### 테스트 네이밍 컨벤션

```python
# 패턴: test_<what>_<when>_<expected>
def test_create_artist_with_valid_data_returns_artist()
def test_get_artist_when_not_found_raises_not_found_error()
def test_update_artist_with_duplicate_name_raises_conflict_error()
```

### 테스트 조직화

- **Unit 테스트**: 격리된 상태에서 개별 함수/메서드 테스트
- **Integration 테스트**: 여러 컴포넌트가 함께 동작하는 것 테스트
- **관련 테스트 그룹화**: 관련 기능에 테스트 클래스 사용

---

## 주제별 가이드

### 테스팅 아키텍처

**3계층 테스팅 전략:**
1. **Repository 계층**: 데이터베이스 쿼리, CRUD 작업 테스트
2. **Service 계층**: 비즈니스 로직, 오케스트레이션 테스트
3. **Router 계층**: API 엔드포인트, 요청/응답 처리 테스트

**핵심 개념:**
- 계층 경계에서 의존성 모킹
- 각 계층 독립적으로 테스트
- 엔드-투-엔드 플로우에 integration 테스트 사용
- 테스트 격리 유지

**[완전한 가이드: resources/testing-architecture.md](resources/testing-architecture.md)**

---

### Unit 테스팅

**Unit 테스트 모범 사례:**
- 단일 책임 테스트
- 외부 의존성 모킹
- 빠른 실행 (데이터베이스, 네트워크 없음)
- 독립적이고 격리됨
- 성공 경로와 실패 경로 모두 테스트

**Unit 테스트 패턴:**
```python
@pytest.mark.asyncio
async def test_artist_service_create():
    # repository 모킹
    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(return_value=artist_model)

    # service 로직 테스트
    service = ArtistService(mock_repo)
    result = await service.create_artist(data)

    assert result.name == data.name
```

**[완전한 가이드: resources/unit-testing.md](resources/unit-testing.md)**

---

### Integration 테스팅

**Integration 테스트 초점:**
- 여러 컴포넌트 함께 테스트
- 실제 데이터베이스 사용 (테스트 데이터베이스)
- 엔드-투-엔드 워크플로우 확인
- API 계약 테스트

**Integration 테스트 패턴:**
```python
@pytest.mark.asyncio
async def test_create_artist_flow(db_session, client):
    # 전체 플로우: API → Service → Repository → DB
    response = client.post("/api/v1/artists", json=artist_data)
    assert response.status_code == 201

    # 데이터베이스에서 확인
    artist = await db_session.get(Artist, response.json()["id"])
    assert artist is not None
```

**[완전한 가이드: resources/integration-testing.md](resources/integration-testing.md)**

---

### Async 테스팅

**Async 테스트 패턴:**
- `@pytest.mark.asyncio` 데코레이터 사용
- conftest.py에서 pytest-asyncio 설정
- AsyncMock으로 async 함수 모킹
- async context manager 테스트
- async 예외 처리

**Async Mock 패턴:**
```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_function():
    mock_func = AsyncMock(return_value="result")
    result = await mock_func()
    assert result == "result"
    mock_func.assert_awaited_once()
```

**[완전한 가이드: resources/async-testing.md](resources/async-testing.md)**

---

### Mocking & Fixtures

**모킹 전략:**
- 외부 의존성 모킹 (데이터베이스, API, S3)
- 재사용 가능한 테스트 데이터에 pytest fixture 사용
- 계층 경계에서 모킹
- 동기에는 MagicMock, 비동기에는 AsyncMock 사용

**Fixture 패턴:**
```python
import pytest

@pytest.fixture
def sample_artist():
    return Artist(
        id="test-id",
        name="Test Artist",
        bio="Test bio"
    )

@pytest.fixture
async def db_session():
    # 테스트 데이터베이스 세션 설정
    async with get_test_session() as session:
        yield session
        await session.rollback()
```

**[완전한 가이드: resources/mocking-fixtures.md](resources/mocking-fixtures.md)**

---

### 커버리지 모범 사례

**커버리지 전략:**
- 80%+ 커버리지 목표 (프로젝트 요구사항)
- 핵심 비즈니스 로직에 집중
- 에러 경로와 엣지 케이스 테스트
- 커버리지 보고서로 누락 부분 찾기
- 테스트 불가 코드 제외 (config, main.py)

**커버리지 명령어:**
```bash
# 커버리지와 함께 테스트 실행
pytest --cov=backend --cov-report=term-missing

# HTML 보고서 생성
pytest --cov=backend --cov-report=html

# 커버리지 임계값 확인
pytest --cov=backend --cov-fail-under=80
```

**[완전한 가이드: resources/coverage-best-practices.md](resources/coverage-best-practices.md)**

---

### FastAPI 테스팅

**FastAPI 테스트 패턴:**
- route 테스팅에 TestClient 사용
- 요청 유효성 검사 테스트
- 응답 직렬화 테스트
- 인증/권한 테스트
- 에러 처리 미들웨어 테스트

**TestClient 패턴:**
```python
from fastapi.testclient import TestClient

def test_create_artist_endpoint(client: TestClient):
    response = client.post(
        "/api/v1/artists",
        json={"name": "Artist", "bio": "Bio"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Artist"
```

**[완전한 가이드: resources/fastapi-testing.md](resources/fastapi-testing.md)**

---

## 네비게이션 가이드

| 필요한 작업 | 읽을 리소스 |
|------------|-----------|
| 테스트 구조 이해 | [testing-architecture.md](resources/testing-architecture.md) |
| Unit 테스트 작성 | [unit-testing.md](resources/unit-testing.md) |
| Integration 테스트 작성 | [integration-testing.md](resources/integration-testing.md) |
| Async 코드 테스트 | [async-testing.md](resources/async-testing.md) |
| Mock 및 fixture 사용 | [mocking-fixtures.md](resources/mocking-fixtures.md) |
| 커버리지 개선 | [coverage-best-practices.md](resources/coverage-best-practices.md) |
| FastAPI route 테스트 | [fastapi-testing.md](resources/fastapi-testing.md) |

---

## 핵심 원칙

1. **테스트 격리**: 각 테스트는 독립적으로 실행, 공유 상태 없음
2. **AAA 패턴**: 명확한 테스트 구조를 위한 Arrange, Act, Assert
3. **Async 테스팅**: async 코드에 pytest-asyncio 사용
4. **의존성 모킹**: 외부 시스템 모킹 (데이터베이스, API)
5. **계층별 테스팅**: 각 계층 (repository, service, router) 별도 테스트
6. **커버리지 목표**: 80%+ 커버리지 목표, 비즈니스 로직에 집중
7. **설명적인 이름**: 명확한 테스트 이름으로 무엇, 언제, 예상 결과 설명
8. **에러 테스팅**: 성공 경로와 실패 경로 모두 테스트
9. **빠른 테스트**: Unit 테스트는 빨라야 함 (실제 데이터베이스 없음)
10. **Fixture**: 재사용 가능한 테스트 데이터와 설정에 fixture 사용

---

## 빠른 참조: 테스트 템플릿

```python
"""Artist 도메인 테스트."""
import pytest
from unittest.mock import AsyncMock, MagicMock
from sqlmodel.ext.asyncio.session import AsyncSession

from backend.domain.artist.service import ArtistService
from backend.domain.artist.repository import ArtistRepository
from backend.domain.artist.model import Artist
from backend.dtos.artist import ArtistRequestDto
from backend.error import NotFoundError


@pytest.fixture
def sample_artist():
    """샘플 아티스트 데이터 fixture."""
    return Artist(
        id="test-artist-id",
        name="Test Artist",
        bio="Test bio"
    )


@pytest.fixture
def mock_session():
    """모킹된 데이터베이스 세션 fixture."""
    return AsyncMock(spec=AsyncSession)


class TestArtistRepository:
    """ArtistRepository 테스트 스위트."""

    @pytest.mark.asyncio
    async def test_get_by_id_success(self, mock_session, sample_artist):
        """get_by_id가 아티스트를 찾으면 반환하는지 테스트."""
        # Arrange
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = sample_artist
        mock_session.execute = AsyncMock(return_value=mock_result)

        repository = ArtistRepository(mock_session)

        # Act
        result = await repository.get_by_id("test-artist-id")

        # Assert
        assert result is not None
        assert result.id == sample_artist.id
        assert result.name == sample_artist.name

    @pytest.mark.asyncio
    async def test_get_by_id_not_found(self, mock_session):
        """get_by_id가 찾지 못하면 None을 반환하는지 테스트."""
        # Arrange
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = None
        mock_session.execute = AsyncMock(return_value=mock_result)

        repository = ArtistRepository(mock_session)

        # Act
        result = await repository.get_by_id("nonexistent-id")

        # Assert
        assert result is None


class TestArtistService:
    """ArtistService 테스트 스위트."""

    @pytest.mark.asyncio
    async def test_create_artist_success(self, mock_session, sample_artist):
        """create_artist가 아티스트를 생성하고 반환하는지 테스트."""
        # Arrange
        mock_repo = AsyncMock()
        mock_repo.create = AsyncMock(return_value=sample_artist)

        service = ArtistService(mock_session)
        service._repository = mock_repo

        request_dto = ArtistRequestDto(
            name="Test Artist",
            bio="Test bio"
        )

        # Act
        result = await service.create_artist(request_dto)

        # Assert
        assert result.name == request_dto.name
        mock_repo.create.assert_awaited_once()
```

---

## 현재 프로젝트 설정

qwarty 백엔드 테스트 설정:

**pytest.ini (pyproject.toml 내):**
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
filterwarnings = ["ignore::DeprecationWarning"]
markers = [
    "security: marks tests as security tests (SQL injection, etc.)",
    "performance: marks tests as performance benchmarks",
]
addopts = [
    "--cov=backend",
    "--cov-report=term-missing",
    "--cov-report=html",
    "--cov-fail-under=80",
]
```

**테스트 의존성:**
- pytest 8.4.2+
- pytest-asyncio 0.24.0+
- pytest-cov 6.0.0+

**커버리지 제외 항목:**
- 테스트 파일 자체 (`tests/*`)
- `__init__.py` 파일
- 메인 애플리케이션 진입점 (`backend/main.py`)
- 일부 router 및 특정 도메인 (pyproject.toml 참조)

---

## 관련 스킬

- **fastapi-backend-guidelines**: 테스팅 대상인 백엔드 개발 패턴
- **error-tracking**: 테스트할 에러 처리 패턴

---

**스킬 상태**: 최적의 컨텍스트 관리를 위한 점진적 로딩을 갖춘 모듈식 구조
