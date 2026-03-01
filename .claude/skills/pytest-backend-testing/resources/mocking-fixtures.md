# 모킹 & Fixture

## 개요

모킹과 fixture는 유지보수가 쉽고, 빠르고, 격리된 테스트를 작성하는 데 필수적입니다. 이 가이드는 pytest fixture, unittest.mock, 그리고 FastAPI 애플리케이션을 위한 고급 모킹 패턴을 다룹니다.

## Pytest Fixture

### 기본 Fixture

Fixture는 재사용 가능한 테스트 데이터와 설정을 제공합니다:

```python
import pytest

@pytest.fixture
def sample_artist():
    """샘플 아티스트를 제공하는 fixture."""
    return Artist(
        id="test-id",
        name="Test Artist",
        bio="Test bio"
    )

# 테스트에서 사용
def test_artist_name(sample_artist):
    assert sample_artist.name == "Test Artist"
```

### Fixture 스코프

fixture의 생존 기간을 제어합니다:

```python
# 함수 스코프 (기본값) - 각 테스트마다 새 fixture
@pytest.fixture
def function_fixture():
    print("Setup")
    yield "value"
    print("Teardown")

# 클래스 스코프 - 테스트 클래스 전체에서 공유
@pytest.fixture(scope="class")
def class_fixture():
    return "shared_value"

# 모듈 스코프 - 모듈 전체에서 공유
@pytest.fixture(scope="module")
def module_fixture():
    return "module_value"

# 세션 스코프 - 전체 테스트 세션에서 공유
@pytest.fixture(scope="session")
def session_fixture():
    return "session_value"
```

### Fixture Teardown

정리를 위해 `yield` 사용:

```python
@pytest.fixture
def database_connection():
    # 설정
    conn = create_connection()

    yield conn  # 테스트에 제공

    # Teardown (테스트 후 실행)
    conn.close()

@pytest.fixture
async def async_session():
    # 비동기 설정
    session = AsyncSession(engine)

    yield session

    # 비동기 teardown
    await session.close()
```

### Fixture 의존성

Fixture가 다른 fixture를 사용할 수 있습니다:

```python
@pytest.fixture
def mock_session():
    return AsyncMock(spec=AsyncSession)

@pytest.fixture
def mock_repository(mock_session):
    return ArtistRepository(mock_session)

@pytest.fixture
def artist_service(mock_session, mock_repository):
    service = ArtistService(mock_session)
    service._repository = mock_repository
    return service

# 테스트에서 사용
@pytest.mark.asyncio
async def test_with_service(artist_service):
    result = await artist_service.get_artist("1")
    # 완전히 설정된 service를 사용하여 테스트
```

---

## Fixture 구성

### conftest.py 구조

```python
# tests/conftest.py - 전역 fixture
import pytest

@pytest.fixture
def mock_session():
    """전역 mock 세션 fixture."""
    return AsyncMock(spec=AsyncSession)

# tests/unit/domain/artist/conftest.py - 도메인 특화 fixture
@pytest.fixture
def sample_artist():
    """아티스트 특화 fixture."""
    return Artist(id="1", name="Test")

@pytest.fixture
def sample_artist_dto():
    """아티스트 DTO fixture."""
    return ArtistRequestDto(name="Test", bio="Bio")
```

### Autouse Fixture

자동으로 실행되는 fixture:

```python
@pytest.fixture(autouse=True)
def setup_logging():
    """모든 테스트 전에 자동으로 실행됩니다."""
    import logging
    logging.basicConfig(level=logging.DEBUG)

@pytest.fixture(autouse=True)
async def reset_database():
    """모든 async 테스트 전에 실행됩니다."""
    # 데이터베이스 상태 초기화
    yield
    # 테스트 후 정리
```

---

## unittest.mock 기본

### MagicMock

동기 코드용:

```python
from unittest.mock import MagicMock

# mock 생성
mock_obj = MagicMock()

# 반환값 설정
mock_obj.method.return_value = "result"

# 사용
result = mock_obj.method("arg")

# 검증
assert result == "result"
mock_obj.method.assert_called_once_with("arg")
```

### AsyncMock

비동기 코드용:

```python
from unittest.mock import AsyncMock

# async mock 생성
mock_func = AsyncMock(return_value="result")

# 사용
result = await mock_func("arg")

# 검증
assert result == "result"
mock_func.assert_awaited_once_with("arg")
```

### Mock 설정

```python
# 반환값
mock.method.return_value = "value"

# Side effect (각 호출마다 다른 값)
mock.method.side_effect = ["first", "second", "third"]

# 예외 발생
mock.method.side_effect = ValueError("error")

# 커스텀 함수
mock.method.side_effect = lambda x: x * 2
```

---

## 모킹 패턴

### 패턴 1: 데이터베이스 세션 모킹

```python
@pytest.fixture
def mock_db_session():
    """SQLModel async 세션 모킹."""
    mock_session = AsyncMock(spec=AsyncSession)

    # 공통 메서드 설정
    mock_session.execute = AsyncMock()
    mock_session.commit = AsyncMock()
    mock_session.rollback = AsyncMock()
    mock_session.close = AsyncMock()
    mock_session.refresh = AsyncMock()

    # 속성
    mock_session.closed = False

    return mock_session

@pytest.mark.asyncio
async def test_with_mock_session(mock_db_session):
    repository = ArtistRepository(mock_db_session)

    # execute가 결과를 반환하도록 설정
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = Artist(id="1")
    mock_db_session.execute.return_value = mock_result

    # 테스트
    result = await repository.get_by_id("1")

    assert result.id == "1"
    mock_db_session.execute.assert_awaited_once()
```

### 패턴 2: Repository 모킹

```python
@pytest.fixture
def mock_artist_repository():
    """아티스트 repository 모킹."""
    mock_repo = AsyncMock(spec=ArtistRepository)

    # 메서드 설정
    mock_repo.get_by_id = AsyncMock(return_value=None)
    mock_repo.find_all = AsyncMock(return_value=[])
    mock_repo.create = AsyncMock()
    mock_repo.update = AsyncMock()
    mock_repo.delete = AsyncMock()

    return mock_repo

@pytest.mark.asyncio
async def test_service_with_mock_repo(mock_artist_repository):
    # 이 테스트를 위한 특정 동작 설정
    artist = Artist(id="1", name="Test")
    mock_artist_repository.get_by_id.return_value = artist

    # mock과 함께 service 생성
    service = ArtistService(mock_session)
    service._repository = mock_artist_repository

    # 테스트
    result = await service.get_artist("1")

    assert result.id == "1"
    mock_artist_repository.get_by_id.assert_awaited_once_with("1")
```

### 패턴 3: 외부 서비스 모킹

```python
@pytest.fixture
def mock_s3_client():
    """boto3 S3 클라이언트 모킹."""
    mock_client = MagicMock()

    # S3 작업 설정
    mock_client.upload_file = MagicMock(return_value=None)
    mock_client.generate_presigned_url = MagicMock(
        return_value="https://example.com/presigned"
    )

    return mock_client

def test_upload_to_s3(mock_s3_client, mocker):
    # 우리 mock을 반환하도록 boto3 패치
    mocker.patch("boto3.client", return_value=mock_s3_client)

    # S3를 사용하는 코드 테스트
    upload_file_to_s3("test.jpg", "bucket")

    # 검증
    mock_s3_client.upload_file.assert_called_once()
```

---

## pytest-mock을 사용한 고급 모킹

### mocker Fixture 사용

pytest-mock은 더 깔끔한 모킹을 위한 `mocker` fixture를 제공합니다:

```python
def test_with_mocker(mocker):
    # 함수 패치
    mock_func = mocker.patch("backend.utils.helpers.some_function")
    mock_func.return_value = "mocked"

    # 사용
    result = some_function()

    assert result == "mocked"
    mock_func.assert_called_once()

@pytest.mark.asyncio
async def test_async_with_mocker(mocker):
    # async 함수 패치
    mock_func = mocker.patch(
        "backend.domain.artist.service.ArtistService.get_artist",
        new_callable=AsyncMock
    )
    mock_func.return_value = ArtistResponseDto(id="1", name="Test")

    # 테스트
    result = await service.get_artist("1")

    assert result.id == "1"
```

### 객체 패치

```python
def test_patch_object(mocker):
    # 객체의 메서드 패치
    service = ArtistService(mock_session)

    mocker.patch.object(
        service._repository,
        "get_by_id",
        return_value=Artist(id="1")
    )

    # 테스트가 패치된 메서드 사용
    result = service.get_artist("1")
```

### 클래스 패치

```python
def test_patch_class(mocker):
    # 전체 클래스 패치
    MockRepository = mocker.patch(
        "backend.domain.artist.repository.ArtistRepository"
    )

    # mock 인스턴스 설정
    mock_instance = MockRepository.return_value
    mock_instance.get_by_id = AsyncMock(return_value=Artist(id="1"))

    # ArtistRepository를 생성하는 코드는 mock을 받게 됨
    service = ArtistService(mock_session)  # MockRepository 사용

    # 검증
    MockRepository.assert_called_once()
```

---

## 데이터베이스 결과 모킹

### SQLModel 쿼리 결과 모킹

```python
@pytest.mark.asyncio
async def test_repository_query():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)

    # mock 결과 생성
    mock_result = MagicMock()

    # 단일 결과: scalar_one_or_none()
    mock_result.scalar_one_or_none.return_value = Artist(id="1", name="Test")

    # 여러 결과: scalars().all()
    mock_scalars = MagicMock()
    mock_scalars.all.return_value = [
        Artist(id="1", name="Artist 1"),
        Artist(id="2", name="Artist 2"),
    ]
    mock_result.scalars.return_value = mock_scalars

    # session.execute가 mock 결과를 반환하도록 설정
    mock_session.execute = AsyncMock(return_value=mock_result)

    # Act
    repository = ArtistRepository(mock_session)
    result = await repository.get_by_id("1")

    # Assert
    assert result.id == "1"
```

### 다양한 쿼리 유형 모킹

```python
# 1. 단일 항목 조회
mock_result = MagicMock()
mock_result.scalar_one_or_none.return_value = artist
mock_session.execute.return_value = mock_result

# 2. 전체 항목 조회
mock_result = MagicMock()
mock_scalars = MagicMock()
mock_scalars.all.return_value = [artist1, artist2]
mock_result.scalars.return_value = mock_scalars
mock_session.execute.return_value = mock_result

# 3. 첫 번째 항목 조회
mock_result = MagicMock()
mock_scalars = MagicMock()
mock_scalars.first.return_value = artist
mock_result.scalars.return_value = mock_scalars
mock_session.execute.return_value = mock_result

# 4. 개수 조회
mock_result = MagicMock()
mock_result.scalar.return_value = 42
mock_session.execute.return_value = mock_result
```

---

## Mock 검증

### 호출 검증

```python
# 호출 여부
mock.method.assert_called()

# 한 번 호출
mock.method.assert_called_once()

# 특정 인수로 호출
mock.method.assert_called_with("arg1", "arg2")
mock.method.assert_called_once_with("arg1", "arg2")

# 임의 인수로 호출
mock.method.assert_called()

# 호출 안 됨
mock.method.assert_not_called()

# 호출 횟수
assert mock.method.call_count == 3

# Async 버전
await mock.async_method("arg")
mock.async_method.assert_awaited()
mock.async_method.assert_awaited_once()
mock.async_method.assert_awaited_with("arg")
```

### 호출 인수 검사

```python
# 모든 호출 가져오기
calls = mock.method.call_args_list

# 가장 최근 호출 가져오기
args, kwargs = mock.method.call_args
assert args == ("arg1",)
assert kwargs == {"key": "value"}

# 특정 호출 확인
mock.method.assert_any_call("arg1")
mock.method.assert_has_calls([
    call("first"),
    call("second"),
])
```

---

## Fixture 파라미터화

### 여러 시나리오 테스팅

```python
@pytest.fixture(params=[
    ("valid-name", True),
    ("", False),
    (None, False),
])
def name_scenario(request):
    name, is_valid = request.param
    return name, is_valid

def test_name_validation(name_scenario):
    name, is_valid = name_scenario

    if is_valid:
        artist = Artist(id="1", name=name)
        assert artist.name == name
    else:
        with pytest.raises(ValidationError):
            Artist(id="1", name=name)
```

### ID가 있는 파라미터화 Fixture

```python
@pytest.fixture(
    params=[
        ("artist1", "bio1"),
        ("artist2", "bio2"),
    ],
    ids=["scenario-1", "scenario-2"]
)
def artist_data(request):
    name, bio = request.param
    return {"name": name, "bio": bio}

def test_with_named_scenarios(artist_data):
    # 설명적인 ID로 두 번 실행됨
    assert artist_data["name"]
```

---

## 모킹 모범 사례

### 1. 올바른 계층에서 모킹

```python
# service 테스팅 시 repository 모킹 (올바른 방법)
@pytest.mark.asyncio
async def test_service():
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo  # 경계에서 모킹

    result = await service.get_artist("1")

# 너무 깊은 계층까지 모킹 (잘못된 방법)
@pytest.mark.asyncio
async def test_service():
    mock_db = AsyncMock()  # 데이터베이스 내부를 모킹
    # 구현에 너무 결합되어 있음
```

### 2. 테스트 대상은 모킹하지 않기

```python
# 테스트 중인 service를 모킹 (잘못된 방법)
def test_service():
    mock_service = AsyncMock()
    mock_service.get_artist = AsyncMock(return_value=artist)

    # mock을 테스트하는 것이지, 실제 service를 테스트하는 게 아님!
    result = await mock_service.get_artist("1")

# 실제 service 테스트, 의존성만 모킹 (올바른 방법)
@pytest.mark.asyncio
async def test_service():
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=artist)

    service = ArtistService(mock_session)  # 실제 service
    service._repository = mock_repo  # 의존성 모킹

    result = await service.get_artist("1")
```

### 3. 타입 안전성을 위해 spec 사용

```python
# spec을 사용하면 잘못된 사용을 방지 (올바른 방법)
mock_session = AsyncMock(spec=AsyncSession)
mock_session.nonexistent_method()  # AttributeError 발생

# spec 없이는 아무거나 허용 (잘못된 방법)
mock_session = AsyncMock()
mock_session.nonexistent_method()  # 허용되지만, 잘못된 사용
```

### 4. 테스트 간 Mock 초기화

```python
@pytest.fixture
def mock_service():
    mock = AsyncMock()
    yield mock
    # 테스트 간에 자동으로 초기화됨

# 또는 수동으로 초기화
def test_1(mock_service):
    await mock_service.method()
    mock_service.method.assert_awaited_once()

def test_2(mock_service):
    # mock_service가 새로 시작되어 call count가 0임
    await mock_service.method()
    mock_service.method.assert_awaited_once()
```

---

## 일반적인 함정

### 1. Mock 반환값 설정 누락

```python
# Mock이 기본적으로 MagicMock을 반환 (잘못된 방법)
mock_repo = AsyncMock()
result = await mock_repo.get_by_id("1")
# result가 Artist가 아닌 MagicMock!

# 반환값 명시적으로 설정 (올바른 방법)
mock_repo = AsyncMock()
mock_repo.get_by_id = AsyncMock(return_value=Artist(id="1"))
result = await mock_repo.get_by_id("1")
# result가 Artist
```

### 2. AsyncMock에 await 누락

```python
# await 누락 (잘못된 방법)
mock_func = AsyncMock(return_value="result")
result = mock_func()  # coroutine을 반환!

# await 있음 (올바른 방법)
result = await mock_func()  # "result"를 반환
```

### 3. 잘못된 위치에서 모킹

```python
# 파일: backend/domain/artist/service.py
from backend.domain.artist.repository import ArtistRepository

class ArtistService:
    def __init__(self):
        self.repo = ArtistRepository()

# 정의된 위치 패치 (잘못된 방법)
mocker.patch("backend.domain.artist.repository.ArtistRepository")

# 사용되는 위치 패치 (올바른 방법)
mocker.patch("backend.domain.artist.service.ArtistRepository")
```

---

## 체크리스트

Mock과 fixture 사용 시:

- [ ] 재사용 가능한 테스트 데이터에 fixture 사용
- [ ] conftest.py에 fixture 구성
- [ ] 적절한 fixture 스코프 사용
- [ ] 계층 경계에서 모킹
- [ ] async 함수에 AsyncMock 사용
- [ ] 타입 안전성을 위해 spec 파라미터 사용
- [ ] Mock 반환값을 명시적으로 설정
- [ ] assert_* 메서드로 mock 호출 검증
- [ ] 테스트 대상은 모킹하지 않기
- [ ] 테스트 간 mock 초기화
- [ ] 더 깔끔한 패치를 위해 pytest-mock 사용

---

## 관련 리소스

- [unit-testing.md](unit-testing.md) - Unit 테스팅 패턴
- [async-testing.md](async-testing.md) - Async 특화 모킹
- [testing-architecture.md](testing-architecture.md) - 전체 테스팅 전략
