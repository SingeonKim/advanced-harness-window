# 테스팅 아키텍처

## 개요

FastAPI 백엔드의 테스팅 아키텍처는 애플리케이션과 동일한 계층적 접근 방식을 따릅니다: Repository → Service → Router. 각 계층에는 특정 테스팅 관심사와 전략이 있습니다.

## 3계층 테스팅 전략

### 1. Repository 계층 테스팅

**테스트 대상:**
- 데이터베이스 쿼리 (SELECT, INSERT, UPDATE, DELETE)
- 필터링 및 정렬 로직
- 복잡한 조인과 관계
- 트랜잭션 처리
- 에러 처리 (unique constraint, foreign key)

**테스팅 접근 방식:**
- AsyncSession 모킹
- SQL 쿼리 구성 확인
- 쿼리 결과 매핑 테스트
- 데이터 접근 로직에 집중

**예시:**
```python
@pytest.mark.asyncio
async def test_artist_repository_find_by_name():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = Artist(
        id="1", name="Test Artist"
    )
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    result = await repository.find_by_name("Test Artist")

    # Assert
    assert result is not None
    assert result.name == "Test Artist"
    mock_session.execute.assert_awaited_once()
```

---

### 2. Service 계층 테스팅

**테스트 대상:**
- 비즈니스 로직과 도메인 규칙
- 서비스 오케스트레이션 (여러 repository 호출)
- 데이터 변환 (model → DTO 변환)
- 에러 처리와 유효성 검사
- 트랜잭션 관리

**테스팅 접근 방식:**
- repository 의존성 모킹
- 데이터베이스가 아닌 비즈니스 로직에 집중
- 다양한 시나리오와 엣지 케이스 테스트
- DTO 생성 확인

**예시:**
```python
@pytest.mark.asyncio
async def test_artist_service_create_with_duplicate_name():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()
    mock_repo.find_by_name = AsyncMock(return_value=existing_artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act & Assert
    with pytest.raises(ConflictError, match="Artist already exists"):
        await service.create_artist(ArtistRequestDto(name="Existing"))
```

---

### 3. Router 계층 테스팅

**테스트 대상:**
- HTTP 요청/응답 처리
- 요청 유효성 검사 (Pydantic)
- 응답 직렬화
- 인증/권한
- HTTP 상태 코드
- 에러 응답 포맷

**테스팅 접근 방식:**
- FastAPI TestClient 사용
- service 의존성 모킹
- 완전한 요청/응답 사이클 테스트
- API 계약 확인

**예시:**
```python
def test_create_artist_endpoint_success(client, mocker):
    # Arrange
    mock_service = mocker.patch("backend.api.v1.routers.artist.ArtistService")
    mock_service.return_value.create_artist = AsyncMock(
        return_value=ArtistResponseDto(id="1", name="Test")
    )

    # Act
    response = client.post(
        "/api/v1/artists",
        json={"name": "Test", "bio": "Bio"}
    )

    # Assert
    assert response.status_code == 201
    assert response.json()["name"] == "Test"
```

---

## 테스트 격리 원칙

### 1. 독립적인 테스트

각 테스트는 다른 테스트에 의존하지 않고 독립적으로 실행되어야 합니다:

```python
# 독립적인 테스트 (권장)
@pytest.mark.asyncio
async def test_create_artist():
    artist = await service.create_artist(data)
    assert artist.id is not None

@pytest.mark.asyncio
async def test_get_artist():
    # 이 테스트를 위해 직접 아티스트 생성
    artist = await service.create_artist(data)
    result = await service.get_artist(artist.id)
    assert result.id == artist.id

# 실행 순서에 의존하는 테스트 (금지)
artist_id = None

@pytest.mark.asyncio
async def test_create_artist():
    global artist_id
    artist = await service.create_artist(data)
    artist_id = artist.id  # 공유 상태!

@pytest.mark.asyncio
async def test_get_artist():
    global artist_id
    result = await service.get_artist(artist_id)  # 이전 테스트에 의존!
```

### 2. 계층 경계에서 모킹

계층 사이의 경계에서 의존성을 모킹합니다:

```python
# Service 계층 테스팅
@pytest.mark.asyncio
async def test_service_logic():
    # repository (아래 계층) 모킹
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo  # mock 주입

    # 격리된 상태에서 service 로직 테스트
    result = await service.get_artist("id")
    assert result is not None
```

### 3. 재사용 가능한 설정에 Fixture 사용

공통 테스트 설정을 위한 fixture 생성:

```python
@pytest.fixture
def sample_artist():
    return Artist(id="1", name="Test", bio="Bio")

@pytest.fixture
def mock_artist_repository(sample_artist):
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=sample_artist)
    mock_repo.create = AsyncMock(return_value=sample_artist)
    return mock_repo

# 테스트에서 사용
@pytest.mark.asyncio
async def test_with_fixtures(mock_artist_repository, sample_artist):
    service = ArtistService(mock_session)
    service._repository = mock_artist_repository

    result = await service.get_artist("1")
    assert result.id == sample_artist.id
```

---

## 테스트 구성

### 디렉토리 구조

```
tests/
  conftest.py                # 전역 fixture
  unit/                      # Unit 테스트 (격리됨)
    domain/
      artist/
        test_artist_model.py
        test_artist_repository.py
        test_artist_service.py
      artwork/
      auth/
    middleware/
      test_error_handler.py
    utils/
      test_helpers.py

  integration/               # Integration 테스트 (다중 계층)
    test_artist_api.py       # 완전한 API 플로우 테스트
    test_auth_flow.py
    test_artwork_creation.py
```

### 테스트 파일 네이밍

- **Unit 테스트**: `test_{module_name}.py`
- **Integration 테스트**: `test_{feature}_api.py` 또는 `test_{feature}_flow.py`
- **테스트 함수**: `test_{what}_{when}_{expected}`

### 테스트 클래스 구성

클래스를 사용하여 관련 테스트 그룹화:

```python
class TestArtistRepository:
    """ArtistRepository 테스트."""

    @pytest.mark.asyncio
    async def test_get_by_id_success(self):
        """성공적인 조회 테스트."""
        pass

    @pytest.mark.asyncio
    async def test_get_by_id_not_found(self):
        """아티스트를 찾지 못했을 때 테스트."""
        pass

    @pytest.mark.asyncio
    async def test_create_artist_success(self):
        """성공적인 생성 테스트."""
        pass

class TestArtistService:
    """ArtistService 테스트."""
    pass
```

---

## 테스트 데이터 관리

### 1. 팩토리 또는 빌더 사용

테스트 데이터를 위한 헬퍼 생성:

```python
def create_artist(
    id: str = "test-id",
    name: str = "Test Artist",
    bio: str | None = None
) -> Artist:
    """테스트 아티스트 생성을 위한 팩토리 함수."""
    return Artist(id=id, name=name, bio=bio)

# 테스트에서 사용
def test_something():
    artist = create_artist(name="Custom Name")
    assert artist.name == "Custom Name"
```

### 2. Fixture 파라미터화

파라미터화된 fixture로 다양한 시나리오 테스트:

```python
@pytest.fixture(params=[
    ("valid-name", True),
    ("", False),
    ("a" * 256, False),  # 너무 긴 이름
])
def artist_name_scenario(request):
    name, is_valid = request.param
    return name, is_valid

def test_artist_name_validation(artist_name_scenario):
    name, is_valid = artist_name_scenario
    if is_valid:
        artist = Artist(id="1", name=name)
        assert artist.name == name
    else:
        with pytest.raises(ValidationError):
            Artist(id="1", name=name)
```

---

## 데이터베이스 테스팅 전략

### 전략 1: 전부 모킹 (Unit 테스트)

```python
@pytest.mark.asyncio
async def test_repository_with_mocks():
    mock_session = AsyncMock(spec=AsyncSession)
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = artist
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)
    result = await repository.get_by_id("id")
    assert result is not None
```

**장점:**
- 빠른 실행
- 데이터베이스 설정 불필요
- 데이터베이스 변경으로부터 격리됨

**단점:**
- 실제 SQL 테스트 안 됨
- 데이터베이스 특정 문제를 잡지 못할 수 있음

### 전략 2: 인메모리 데이터베이스 (Integration 테스트)

```python
@pytest.fixture
async def test_db_session():
    """테스트용 인메모리 SQLite 데이터베이스 생성."""
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        echo=True
    )

    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

    await engine.dispose()

@pytest.mark.asyncio
async def test_with_real_database(test_db_session):
    repository = ArtistRepository(test_db_session)
    artist = await repository.create(Artist(id="1", name="Test"))

    result = await repository.get_by_id("1")
    assert result.id == "1"
```

**장점:**
- 실제 데이터베이스 작업 테스트
- SQL 에러 감지
- 관계와 제약 조건 확인

**단점:**
- 모킹보다 느림
- SQLite가 PostgreSQL과 다를 수 있음
- 테스트 사이에 정리 필요

### 전략 3: 트랜잭션 롤백 (정리)

```python
@pytest.fixture
async def db_session_with_rollback(test_db_session):
    """테스트 후 롤백하는 세션."""
    async with test_db_session.begin():
        yield test_db_session
        await test_db_session.rollback()
```

---

## 공통 패턴

### 패턴: 에러 처리 테스팅

```python
@pytest.mark.asyncio
async def test_service_handles_not_found():
    # Arrange
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(return_value=None)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act & Assert
    with pytest.raises(NotFoundError, match="Artist not found"):
        await service.get_artist("nonexistent-id")
```

### 패턴: 유효성 검사 테스팅

```python
def test_request_dto_validation():
    # 유효한 데이터
    dto = ArtistRequestDto(name="Valid Name", bio="Bio")
    assert dto.name == "Valid Name"

    # 유효하지 않은 데이터 - 빈 이름
    with pytest.raises(ValidationError):
        ArtistRequestDto(name="", bio="Bio")

    # 유효하지 않은 데이터 - 이름이 너무 긴 경우
    with pytest.raises(ValidationError):
        ArtistRequestDto(name="a" * 256, bio="Bio")
```

### 패턴: Async Context Manager 테스팅

```python
@pytest.mark.asyncio
async def test_transaction_manager():
    mock_session = AsyncMock()

    async with transaction_manager(mock_session):
        # 무언가 수행
        pass

    # commit이 호출되었는지 확인
    mock_session.commit.assert_awaited_once()
```

---

## 모범 사례 요약

1. **각 계층 독립적으로 테스트**: 의존성 모킹, 계층별 로직에 집중
2. **설명적인 이름 사용**: 테스트 이름으로 무엇, 언제, 예상 결과 설명
3. **AAA 패턴 따르기**: 명확한 테스트 구조를 위한 Arrange, Act, Assert
4. **경계에서 모킹**: 테스팅 중인 것의 바로 아래 계층 모킹
5. **테스트를 빠르게 유지**: Unit 테스트는 밀리초 안에 실행되어야 함
6. **에러 경로 테스트**: 정상 경로만 테스트하지 않기
7. **Fixture 사용**: 공통 설정에 pytest fixture 재사용
8. **격리 유지**: 각 테스트는 독립적이어야 함
9. **논리적으로 구성**: 관련 테스트 그룹화, 명확한 디렉토리 구조 사용
10. **커버리지에 집중**: 80%+ 목표이지만 핵심 비즈니스 로직 우선

---

## 관련 리소스

- [unit-testing.md](unit-testing.md) - 상세한 unit 테스팅 패턴
- [integration-testing.md](integration-testing.md) - Integration 테스트 전략
- [mocking-fixtures.md](mocking-fixtures.md) - 고급 모킹 및 fixture
