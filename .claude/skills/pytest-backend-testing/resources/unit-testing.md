# Unit 테스팅

## 개요

Unit 테스트는 격리된 상태에서 개별 함수, 메서드 또는 클래스를 테스트하는 데 초점을 맞춥니다. 빠르고, 독립적이며, 단일 기능 단위를 테스트해야 합니다.

## Unit 테스트 특징

### 빠름
- 밀리초 단위로 실행
- 데이터베이스 연결 없음
- 네트워크 호출 없음
- 파일 I/O 없음

### 격리됨
- 한 번에 한 가지만 테스트
- 모든 의존성 모킹
- 테스트 간 공유 상태 없음
- 테스트 실행 순서에 독립적

### 집중됨
- 단일 책임 테스트
- 명확한 Arrange-Act-Assert 구조
- 테스트당 하나의 assertion 개념

---

## AAA 패턴 (Arrange-Act-Assert)

### 구조

모든 unit 테스트는 AAA 패턴을 따라야 합니다:

```python
@pytest.mark.asyncio
async def test_create_artist_success():
    # Arrange - 테스트 데이터와 mock 설정
    mock_session = AsyncMock(spec=AsyncSession)
    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(return_value=artist_model)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    request_dto = ArtistRequestDto(name="Test", bio="Bio")

    # Act - 테스트할 코드 실행
    result = await service.create_artist(request_dto)

    # Assert - 결과 확인
    assert result.name == "Test"
    assert result.bio == "Bio"
    mock_repo.create.assert_awaited_once()
```

### 장점
- **가독성**: 명확한 구조로 테스트를 쉽게 이해
- **유지보수성**: 수정과 디버깅이 용이
- **문서화**: 테스트가 사용 예시로 활용됨

---

## Repository 계층 테스팅

### 기본 Repository 테스트

```python
from unittest.mock import AsyncMock, MagicMock
from sqlmodel.ext.asyncio.session import AsyncSession

@pytest.mark.asyncio
async def test_get_by_id_returns_artist():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    expected_artist = Artist(id="test-id", name="Test Artist")

    # execute 결과 모킹
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = expected_artist
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    result = await repository.get_by_id("test-id")

    # Assert
    assert result is not None
    assert result.id == "test-id"
    assert result.name == "Test Artist"
    mock_session.execute.assert_awaited_once()
```

### 쿼리 구성 테스팅

```python
@pytest.mark.asyncio
async def test_find_by_name_constructs_correct_query():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = None
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    await repository.find_by_name("Test Artist")

    # Assert
    # execute가 select 문으로 호출되었는지 확인
    mock_session.execute.assert_awaited_once()
    call_args = mock_session.execute.call_args[0][0]
    # 여기서 쿼리 구조를 확인할 수 있음
    assert str(call_args).lower().__contains__("select")
```

### 에러 케이스 테스팅

```python
@pytest.mark.asyncio
async def test_get_by_id_returns_none_when_not_found():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = None
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    result = await repository.get_by_id("nonexistent-id")

    # Assert
    assert result is None
```

---

## Service 계층 테스팅

### 기본 Service 테스트

```python
@pytest.mark.asyncio
async def test_get_artist_success():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()

    artist_model = Artist(id="1", name="Test", bio="Bio")
    mock_repo.get_by_id = AsyncMock(return_value=artist_model)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act
    result = await service.get_artist("1")

    # Assert
    assert result.id == "1"
    assert result.name == "Test"
    mock_repo.get_by_id.assert_awaited_once_with("1")
```

### 비즈니스 로직 테스팅

```python
@pytest.mark.asyncio
async def test_create_artist_validates_unique_name():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()

    # 동일한 이름의 기존 아티스트 시뮬레이션
    existing_artist = Artist(id="existing", name="Existing Artist")
    mock_repo.find_by_name = AsyncMock(return_value=existing_artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    request_dto = ArtistRequestDto(name="Existing Artist", bio="Bio")

    # Act & Assert
    with pytest.raises(ConflictError, match="Artist.*already exists"):
        await service.create_artist(request_dto)

    # create가 호출되지 않았는지 확인
    mock_repo.create.assert_not_awaited()
```

### 데이터 변환 테스팅

```python
@pytest.mark.asyncio
async def test_get_artist_returns_dto():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()

    artist_model = Artist(id="1", name="Test", bio="Bio")
    mock_repo.get_by_id = AsyncMock(return_value=artist_model)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act
    result = await service.get_artist("1")

    # Assert
    # 모델이 아닌 DTO인지 확인
    from backend.dtos.artist import ArtistResponseDto
    assert isinstance(result, ArtistResponseDto)
    assert result.id == artist_model.id
    assert result.name == artist_model.name
```

### 에러 처리 테스팅

```python
@pytest.mark.asyncio
async def test_get_artist_raises_not_found_error():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=None)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act & Assert
    with pytest.raises(NotFoundError) as exc_info:
        await service.get_artist("nonexistent")

    assert "not found" in str(exc_info.value).lower()
```

---

## 모델과 DTO 테스팅

### 모델 생성 테스팅

```python
def test_artist_model_creation():
    # Act
    artist = Artist(
        id="test-id",
        name="Test Artist",
        bio="Test bio"
    )

    # Assert
    assert artist.id == "test-id"
    assert artist.name == "Test Artist"
    assert artist.bio == "Test bio"
```

### DTO 유효성 검사 테스팅

```python
from pydantic import ValidationError

def test_artist_request_dto_validates_name():
    # 유효한 이름
    dto = ArtistRequestDto(name="Valid Name", bio="Bio")
    assert dto.name == "Valid Name"

    # 빈 이름은 실패해야 함
    with pytest.raises(ValidationError) as exc_info:
        ArtistRequestDto(name="", bio="Bio")

    errors = exc_info.value.errors()
    assert any(e["loc"] == ("name",) for e in errors)

def test_artist_request_dto_validates_name_length():
    # 이름이 너무 긴 경우 (> 255자)
    long_name = "a" * 256

    with pytest.raises(ValidationError) as exc_info:
        ArtistRequestDto(name=long_name, bio="Bio")

    errors = exc_info.value.errors()
    assert any("max_length" in str(e) for e in errors)
```

### 커스텀 유효성 검사 테스팅

```python
def test_artist_dto_custom_validator():
    # Arrange
    class ArtistDto(BaseModel):
        name: str

        @field_validator('name')
        def name_must_not_be_whitespace(cls, v):
            if not v.strip():
                raise ValueError('이름은 공백으로만 구성될 수 없습니다')
            return v.strip()

    # 유효한 이름
    dto = ArtistDto(name="  Test  ")
    assert dto.name == "Test"  # 트리밍됨

    # 공백만 있는 이름은 실패해야 함
    with pytest.raises(ValidationError):
        ArtistDto(name="   ")
```

---

## 유틸리티 함수 테스팅

### 순수 함수

```python
from backend.utils.slug import create_slug

def test_create_slug_converts_to_lowercase():
    assert create_slug("Hello World") == "hello-world"

def test_create_slug_replaces_spaces():
    assert create_slug("Hello   World") == "hello-world"

def test_create_slug_removes_special_chars():
    assert create_slug("Hello@World!") == "helloworld"

def test_create_slug_handles_unicode():
    assert create_slug("Café") == "cafe"
```

### 사이드 이펙트가 있는 함수

```python
from unittest.mock import patch, mock_open

def test_write_file():
    # Arrange
    mock_file = mock_open()

    # Act
    with patch("builtins.open", mock_file):
        write_data_to_file("test.txt", "data")

    # Assert
    mock_file.assert_called_once_with("test.txt", "w")
    mock_file().write.assert_called_once_with("data")
```

---

## 테스트 조직화 패턴

### 테스트 클래스 사용

클래스로 관련 테스트 그룹화:

```python
class TestArtistRepository:
    """ArtistRepository 테스트 스위트."""

    @pytest.fixture
    def mock_session(self):
        """모킹된 세션 fixture, 모든 테스트에서 재사용."""
        return AsyncMock(spec=AsyncSession)

    @pytest.fixture
    def repository(self, mock_session):
        """repository 인스턴스 fixture."""
        return ArtistRepository(mock_session)

    @pytest.mark.asyncio
    async def test_get_by_id_success(self, repository, mock_session):
        # Arrange
        mock_result = MagicMock()
        mock_result.scalar_one_or_none.return_value = Artist(id="1")
        mock_session.execute = AsyncMock(return_value=mock_result)

        # Act
        result = await repository.get_by_id("1")

        # Assert
        assert result.id == "1"

    @pytest.mark.asyncio
    async def test_get_by_id_not_found(self, repository, mock_session):
        # 유사한 테스트...
        pass
```

### 파라미터화된 테스트

하나의 테스트 함수로 여러 시나리오 테스트:

```python
@pytest.mark.parametrize("input_name,expected_slug", [
    ("Hello World", "hello-world"),
    ("UPPERCASE", "uppercase"),
    ("special@chars!", "specialchars"),
    ("  spaces  ", "spaces"),
])
def test_create_slug_various_inputs(input_name, expected_slug):
    assert create_slug(input_name) == expected_slug
```

### 공유 Fixture

conftest.py에서 재사용 가능한 fixture 생성:

```python
# conftest.py
import pytest

@pytest.fixture
def sample_artist():
    return Artist(id="test-id", name="Test Artist", bio="Bio")

@pytest.fixture
def sample_artist_dto():
    return ArtistRequestDto(name="Test Artist", bio="Bio")

# test_artist_service.py
def test_with_shared_fixture(sample_artist):
    assert sample_artist.name == "Test Artist"
```

---

## 공통 모킹 패턴

### Async 함수 모킹

```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_function():
    # async mock 생성
    mock_func = AsyncMock(return_value="result")

    # 호출
    result = await mock_func("arg")

    # Assert
    assert result == "result"
    mock_func.assert_awaited_once_with("arg")
```

### 클래스 모킹

```python
from unittest.mock import MagicMock

def test_mock_class():
    # 클래스 모킹
    MockClass = MagicMock()
    mock_instance = MockClass.return_value

    # mock 인스턴스 설정
    mock_instance.method.return_value = "result"

    # 사용
    instance = MockClass()
    result = instance.method("arg")

    # Assert
    assert result == "result"
    mock_instance.method.assert_called_once_with("arg")
```

### 속성 모킹

```python
def test_mock_attributes():
    # 속성이 있는 mock 생성
    mock_obj = MagicMock()
    mock_obj.attribute = "value"
    mock_obj.method.return_value = "result"

    # 사용
    assert mock_obj.attribute == "value"
    assert mock_obj.method() == "result"
```

---

## 모범 사례

### 1. 테스트당 하나의 assertion 개념

```python
# 한 가지를 테스트 (권장)
def test_create_slug_converts_to_lowercase():
    assert create_slug("HELLO") == "hello"

def test_create_slug_replaces_spaces():
    assert create_slug("hello world") == "hello-world"

# 여러 가지를 테스트 (금지)
def test_create_slug():
    assert create_slug("HELLO") == "hello"
    assert create_slug("hello world") == "hello-world"
    assert create_slug("hello@world") == "helloworld"
```

### 2. 설명적인 테스트 이름

```python
# 명확한 이름 (권장)
def test_get_artist_when_not_found_raises_not_found_error():
    pass

def test_create_artist_with_duplicate_name_raises_conflict_error():
    pass

# 모호한 이름 (금지)
def test_get_artist():
    pass

def test_error():
    pass
```

### 3. 구현 세부사항 테스트하지 않기

```python
# 동작 테스트 (권장)
@pytest.mark.asyncio
async def test_create_artist_saves_to_database():
    # Act
    result = await service.create_artist(dto)

    # Assert - 저장되었는지 확인 (동작)
    mock_repo.create.assert_awaited_once()
    assert result.id is not None

# 구현 테스트 (금지)
@pytest.mark.asyncio
async def test_create_artist_calls_repository_create_method():
    # 어떻게 하는지에 너무 집중, 무엇을 하는지 아님
    pass
```

### 4. 에러 케이스 테스트

```python
# 성공과 실패 모두 테스트
@pytest.mark.asyncio
async def test_get_artist_success():
    mock_repo.get_by_id = AsyncMock(return_value=artist)
    result = await service.get_artist("1")
    assert result is not None

@pytest.mark.asyncio
async def test_get_artist_not_found():
    mock_repo.get_by_id = AsyncMock(return_value=None)
    with pytest.raises(NotFoundError):
        await service.get_artist("nonexistent")
```

### 5. 테스트를 단순하게 유지

```python
# 단순하고 명확한 테스트 (권장)
@pytest.mark.asyncio
async def test_create_artist():
    mock_repo.create = AsyncMock(return_value=artist)
    result = await service.create_artist(dto)
    assert result.name == dto.name

# 너무 복잡한 테스트 (금지)
@pytest.mark.asyncio
async def test_create_artist_complex():
    # 너무 많은 설정, 다중 작업, 무엇을 테스트하는지 불명확
    for i in range(10):
        dto = create_dto(f"Artist {i}")
        result = await service.create_artist(dto)
        if i % 2 == 0:
            assert result.name.startswith("Artist")
        # ... 더 복잡한 로직
```

---

## 공통 함정

### 1. 테스트 격리하지 않기

```python
# 테스트 간 공유 상태 (금지)
global_artist = None

def test_create():
    global global_artist
    global_artist = create_artist()

def test_get():
    # test_create가 먼저 실행되어야 함에 의존!
    assert global_artist is not None
```

### 2. 너무 많이 테스트하기

```python
# 프레임워크 코드 테스트 (금지)
def test_pydantic_validation():
    # Pydantic이 유효성 검사를 한다는 것을 테스트하지 마세요
    # 당신의 유효성 검사 로직을 테스트하세요
    with pytest.raises(ValidationError):
        ArtistDto(name=None)  # Pydantic이 처리, 당신의 코드가 아님
```

### 3. 테스트하는 것을 모킹하기

```python
# 테스트 중인 것을 모킹 (금지)
def test_service():
    mock_service = AsyncMock()
    mock_service.get_artist = AsyncMock(return_value=artist)

    # mock을 테스트하는 것, 실제 service가 아님!
    result = await mock_service.get_artist("1")
    assert result == artist
```

---

## 요약 체크리스트

unit 테스트 작성 시:

- [ ] 단일 기능 단위 테스트
- [ ] 모든 외부 의존성 모킹
- [ ] AAA 패턴 따르기
- [ ] 설명적인 테스트 이름 사용
- [ ] 성공 케이스와 에러 케이스 모두 테스트
- [ ] 테스트를 단순하고 집중적으로 유지
- [ ] 테스트가 빠른지 확인 (<100ms)
- [ ] 테스트를 독립적으로 만들기
- [ ] 구현 세부사항 테스트하지 않기
- [ ] 재사용 가능한 설정에 fixture 사용

---

## 관련 리소스

- [testing-architecture.md](testing-architecture.md) - 전체 테스팅 전략
- [mocking-fixtures.md](mocking-fixtures.md) - 고급 모킹 기술
- [async-testing.md](async-testing.md) - Async 특정 패턴
