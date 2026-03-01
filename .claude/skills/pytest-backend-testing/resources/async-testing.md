# Async 테스팅

## 개요

FastAPI와 SQLModel은 async/await를 광범위하게 사용합니다. 비동기 코드를 테스트하려면 특별한 고려사항과 도구가 필요하며, 주로 pytest-asyncio를 사용합니다.

## pytest-asyncio 설정

### 설정 방법

프로젝트에는 이미 `conftest.py`에 pytest-asyncio가 설정되어 있습니다:

```python
# conftest.py
import pytest

# pytest-asyncio를 auto 모드로 설정
pytest_plugins = ("pytest_asyncio",)
```

### Auto 모드

auto 모드를 활성화하면 pytest-asyncio가 자동으로 async 테스트 함수를 처리합니다. 여전히 `@pytest.mark.asyncio`로 마킹해야 합니다:

```python
@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_function()
    assert result == expected
```

---

## Async 함수 테스팅

### 기본 Async 테스트

```python
import pytest

@pytest.mark.asyncio
async def test_get_artist_by_id():
    # Arrange
    artist_id = "test-id"

    # Act
    result = await artist_service.get_artist(artist_id)

    # Assert
    assert result.id == artist_id
```

### AsyncMock으로 Async 테스팅

```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_repository_get_by_id():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = Artist(id="1")
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    result = await repository.get_by_id("1")

    # Assert
    assert result.id == "1"
    mock_session.execute.assert_awaited_once()
```

---

## AsyncMock vs MagicMock

### 각각의 사용 시점

```python
# Async 함수/메서드에는 AsyncMock 사용
async_func = AsyncMock(return_value="result")
result = await async_func()

# 동기 속성/어트리뷰트에는 MagicMock 사용
mock_obj = MagicMock()
mock_obj.property = "value"
value = mock_obj.property  # await 불필요

# 복잡한 객체에는 조합하여 사용
mock_session = AsyncMock(spec=AsyncSession)
mock_session.execute = AsyncMock(return_value=mock_result)  # 비동기 메서드
mock_session.closed = False  # 동기 속성
```

### 일반적인 Async 패턴

```python
from unittest.mock import AsyncMock

# 1. 단순 async 함수
mock_func = AsyncMock(return_value="result")
result = await mock_func()
assert result == "result"

# 2. 예외를 발생시키는 async 함수
mock_func = AsyncMock(side_effect=ValueError("error"))
with pytest.raises(ValueError):
    await mock_func()

# 3. 여러 번 호출되는 async 함수
mock_func = AsyncMock(side_effect=["first", "second", "third"])
assert await mock_func() == "first"
assert await mock_func() == "second"
assert await mock_func() == "third"

# 4. await 여부 검증
mock_func = AsyncMock()
await mock_func("arg1", "arg2")
mock_func.assert_awaited_once_with("arg1", "arg2")
```

---

## Async Context Manager 테스팅

### 기본 Context Manager 테스트

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_session():
    session = AsyncSession(engine)
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()

@pytest.mark.asyncio
async def test_db_session_context_manager():
    # Act
    async with get_db_session() as session:
        # 세션 사용
        assert session is not None
        # 여기서 세션이 닫히지 않아야 함
        assert not session.closed

    # context 종료 후 세션이 닫혀야 함
    # (구현에 따라 다름)
```

### Async Context Manager 모킹

```python
from unittest.mock import AsyncMock, MagicMock

@pytest.mark.asyncio
async def test_with_mocked_context_manager():
    # async context manager를 지원하는 mock 생성
    mock_session = AsyncMock()
    mock_session.__aenter__ = AsyncMock(return_value=mock_session)
    mock_session.__aexit__ = AsyncMock(return_value=None)

    # 사용
    async with mock_session as session:
        await session.execute("query")

    # 검증
    mock_session.__aenter__.assert_awaited_once()
    mock_session.__aexit__.assert_awaited_once()
```

---

## Async 데이터베이스 작업 테스팅

### SQLModel 쿼리 모킹

```python
from sqlmodel import select
from sqlmodel.ext.asyncio.session import AsyncSession

@pytest.mark.asyncio
async def test_repository_find_all():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)

    # 쿼리 결과 모킹
    mock_result = MagicMock()
    mock_result.scalars.return_value.all.return_value = [
        Artist(id="1", name="Artist 1"),
        Artist(id="2", name="Artist 2"),
    ]
    mock_session.execute = AsyncMock(return_value=mock_result)

    repository = ArtistRepository(mock_session)

    # Act
    results = await repository.find_all()

    # Assert
    assert len(results) == 2
    assert results[0].id == "1"
    assert results[1].id == "2"
    mock_session.execute.assert_awaited_once()
```

### 트랜잭션 관리 테스팅

```python
@pytest.mark.asyncio
async def test_service_transaction_commit():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_session.commit = AsyncMock()
    mock_session.rollback = AsyncMock()

    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(return_value=artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act
    await service.create_artist(dto)

    # Assert - commit이 호출되었는지 확인
    mock_session.commit.assert_awaited_once()
    mock_session.rollback.assert_not_awaited()

@pytest.mark.asyncio
async def test_service_transaction_rollback_on_error():
    # Arrange
    mock_session = AsyncMock(spec=AsyncSession)
    mock_session.commit = AsyncMock()
    mock_session.rollback = AsyncMock()

    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(side_effect=Exception("DB Error"))

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act & Assert
    with pytest.raises(Exception):
        await service.create_artist(dto)

    # rollback이 호출되었는지 확인
    mock_session.rollback.assert_awaited_once()
    mock_session.commit.assert_not_awaited()
```

---

## 동시 Async 작업 테스팅

### 여러 Async 호출 테스팅

```python
import asyncio

@pytest.mark.asyncio
async def test_concurrent_artist_creation():
    # Arrange
    mock_session = AsyncMock()
    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(side_effect=lambda x: x)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act - 여러 아티스트를 동시에 생성
    artists = [
        ArtistRequestDto(name=f"Artist {i}", bio=f"Bio {i}")
        for i in range(5)
    ]

    results = await asyncio.gather(*[
        service.create_artist(artist) for artist in artists
    ])

    # Assert
    assert len(results) == 5
    assert mock_repo.create.await_count == 5
```

### asyncio.gather 테스팅

```python
@pytest.mark.asyncio
async def test_gather_multiple_queries():
    # Arrange
    mock_repo = AsyncMock()
    mock_repo.get_by_id = AsyncMock(side_effect=[
        Artist(id="1", name="Artist 1"),
        Artist(id="2", name="Artist 2"),
        Artist(id="3", name="Artist 3"),
    ])

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act
    results = await asyncio.gather(
        service.get_artist("1"),
        service.get_artist("2"),
        service.get_artist("3"),
    )

    # Assert
    assert len(results) == 3
    assert results[0].id == "1"
    assert results[1].id == "2"
    assert results[2].id == "3"
```

---

## Async Generator 테스팅

### 기본 Async Generator 테스트

```python
async def fetch_artists_stream():
    """아티스트를 yield하는 async generator."""
    for i in range(5):
        await asyncio.sleep(0)  # async 작업 시뮬레이션
        yield Artist(id=str(i), name=f"Artist {i}")

@pytest.mark.asyncio
async def test_async_generator():
    # Act
    artists = []
    async for artist in fetch_artists_stream():
        artists.append(artist)

    # Assert
    assert len(artists) == 5
    assert artists[0].id == "0"
    assert artists[4].id == "4"
```

### Async Generator 모킹

```python
from unittest.mock import AsyncMock

async def mock_async_generator():
    """모킹용 async generator 헬퍼."""
    yield Artist(id="1", name="Artist 1")
    yield Artist(id="2", name="Artist 2")

@pytest.mark.asyncio
async def test_with_mocked_async_generator():
    # Arrange
    mock_stream = mock_async_generator()

    # Act
    artists = []
    async for artist in mock_stream:
        artists.append(artist)

    # Assert
    assert len(artists) == 2
```

---

## Async 타임아웃 테스팅

### asyncio.timeout으로 테스팅

```python
@pytest.mark.asyncio
async def test_operation_completes_within_timeout():
    # Arrange
    async def slow_operation():
        await asyncio.sleep(0.1)
        return "result"

    # Act & Assert - 1초 이내에 완료되어야 함
    async with asyncio.timeout(1.0):
        result = await slow_operation()
        assert result == "result"

@pytest.mark.asyncio
async def test_operation_times_out():
    # Arrange
    async def very_slow_operation():
        await asyncio.sleep(10)
        return "result"

    # Act & Assert - 타임아웃이 발생해야 함
    with pytest.raises(asyncio.TimeoutError):
        async with asyncio.timeout(0.1):
            await very_slow_operation()
```

---

## 일반적인 Async 테스팅 패턴

### 패턴 1: Async 재시도 로직 테스팅

```python
@pytest.mark.asyncio
async def test_retry_on_failure():
    # Arrange
    mock_func = AsyncMock(
        side_effect=[
            Exception("First failure"),
            Exception("Second failure"),
            "success"  # 세 번째 시도에서 성공
        ]
    )

    # 재시도 로직
    async def retry_operation(max_retries=3):
        for attempt in range(max_retries):
            try:
                return await mock_func()
            except Exception:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(0.1)

    # Act
    result = await retry_operation()

    # Assert
    assert result == "success"
    assert mock_func.await_count == 3
```

### 패턴 2: Async 캐싱 테스팅

```python
@pytest.mark.asyncio
async def test_cached_result_not_refetched():
    # Arrange
    call_count = 0

    async def expensive_operation():
        nonlocal call_count
        call_count += 1
        await asyncio.sleep(0.1)
        return "result"

    cache = {}

    async def get_with_cache(key):
        if key not in cache:
            cache[key] = await expensive_operation()
        return cache[key]

    # Act
    result1 = await get_with_cache("key1")
    result2 = await get_with_cache("key1")  # 캐시를 사용해야 함

    # Assert
    assert result1 == "result"
    assert result2 == "result"
    assert call_count == 1  # 한 번만 호출되어야 함
```

### 패턴 3: Async 이벤트 처리 테스팅

```python
@pytest.mark.asyncio
async def test_event_handler():
    # Arrange
    events = []

    async def event_handler(event):
        events.append(event)
        await asyncio.sleep(0)  # 비동기 처리 시뮬레이션

    # Act
    await event_handler({"type": "create", "id": "1"})
    await event_handler({"type": "update", "id": "2"})

    # Assert
    assert len(events) == 2
    assert events[0]["type"] == "create"
    assert events[1]["type"] == "update"
```

---

## Async 테스트 디버깅

### 일반적인 문제

**1. @pytest.mark.asyncio 누락:**
```python
# @데코레이터 누락
async def test_async_function():
    result = await some_async_function()
    assert result == expected
# Error: coroutine was never awaited

# 데코레이터 있음 (올바른 방법)
@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_function()
    assert result == expected
```

**2. async 함수에 MagicMock 사용:**
```python
# async 함수에 MagicMock 사용 (잘못된 방법)
mock_func = MagicMock(return_value="result")
result = await mock_func()  # Error!

# async 함수에 AsyncMock 사용 (올바른 방법)
mock_func = AsyncMock(return_value="result")
result = await mock_func()
```

**3. async 호출에 await 누락:**
```python
# await 누락 (잘못된 방법)
result = mock_func()  # coroutine을 반환하며, 실제 결과가 아님

# await 있음 (올바른 방법)
result = await mock_func()
```

### 디버그 팁

```python
# async 작업 타이밍 출력
import time

@pytest.mark.asyncio
async def test_with_timing():
    start = time.time()

    result = await some_async_operation()

    duration = time.time() - start
    print(f"Operation took {duration:.2f}s")

    assert result == expected
```

---

## Async 테스트 Fixture

### Async Fixture 생성

```python
import pytest
from sqlmodel.ext.asyncio.session import AsyncSession

@pytest.fixture
async def async_db_session():
    """데이터베이스 세션을 위한 async fixture."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")

    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

    await engine.dispose()

# 테스트에서 사용
@pytest.mark.asyncio
async def test_with_async_fixture(async_db_session):
    # async 세션 사용
    result = await async_db_session.execute(select(Artist))
    artists = result.scalars().all()
    assert isinstance(artists, list)
```

### Async Fixture 스코프

```python
# 함수 스코프 (기본값) - 각 테스트마다 새 fixture
@pytest.fixture
async def async_fixture():
    yield "value"

# 세션 스코프 - 모든 테스트에서 공유
@pytest.fixture(scope="session")
async def async_session_fixture():
    yield "shared_value"
```

---

## 모범 사례

1. **항상 @pytest.mark.asyncio 사용**: async 테스트 함수에 필수
2. **async 호출에 AsyncMock 사용**: 일반 MagicMock을 사용하지 말 것
3. **모든 async 호출에 await**: await 누락 = 버그
4. **동시 작업 테스트**: asyncio.gather 패턴 테스트
5. **async 경계에서 모킹**: async 의존성을 올바르게 모킹
6. **async 예외 처리**: async 코드의 에러 경로 테스트
7. **async 리소스 정리**: 세션/연결이 닫히도록 보장
8. **이벤트 루프 유의**: 테스트당 하나의 이벤트 루프 (pytest-asyncio가 처리)

---

## 체크리스트

async 코드 테스트 시:

- [ ] `@pytest.mark.asyncio` 데코레이터 사용
- [ ] async 함수에 `AsyncMock` 사용
- [ ] 모든 async 함수 호출에 await
- [ ] async 데이터베이스 세션 올바르게 모킹
- [ ] 트랜잭션 commit/rollback 테스트
- [ ] 해당하는 경우 동시 작업 테스트
- [ ] async context manager 올바르게 처리
- [ ] fixture에서 async 리소스 정리
- [ ] 타임아웃 시나리오 테스트
- [ ] `assert_awaited_*` 메서드로 async 호출 검증

---

## 관련 리소스

- [unit-testing.md](unit-testing.md) - 일반 unit 테스팅 패턴
- [mocking-fixtures.md](mocking-fixtures.md) - 고급 모킹 기법
- [integration-testing.md](integration-testing.md) - 실제 async 데이터베이스로 테스팅
