# Integration 테스팅

## 개요

Integration 테스트는 여러 컴포넌트가 함께 올바르게 동작하는지 검증합니다. 개별 단위를 격리하는 unit 테스트와 달리, integration 테스트는 계층 간, 서비스 간, 외부 시스템과의 상호작용을 테스트합니다.

**중요:** Integration 테스트에는 반드시 전용 테스트 데이터베이스(PostgreSQL)를 사용해야 합니다. 프로덕션 데이터베이스를 절대 사용하지 마세요.
- 로컬: docker-compose를 사용해 포트 5433에서 PostgreSQL 테스트 데이터베이스 실행
- CI: GitHub Actions에서 포트 5432에 PostgreSQL 서비스 컨테이너 자동 제공

## Integration vs Unit 테스트

### 비교

| 항목              | Unit 테스트              | Integration 테스트        |
| ----------------- | ------------------------ | ------------------------- |
| **범위**          | 단일 함수/클래스         | 여러 컴포넌트             |
| **의존성**        | 전부 모킹                | 일부/없음 모킹            |
| **속도**          | 빠름 (밀리초)            | 느림 (초)                 |
| **데이터베이스**  | 모킹                     | PostgreSQL 테스트 DB      |
| **네트워크**      | 모킹                     | 실제 서비스 사용 가능     |
| **집중 대상**     | 로직 정확성              | 컴포넌트 상호작용         |
| **수량**          | 많음 (수백 개)           | 적음 (수십 개)            |

### 각 테스트 사용 시점

```python
# Unit 테스트 - 모킹된 repository로 service 로직 테스트
@pytest.mark.asyncio
async def test_create_artist_service_unit():
    mock_repo = AsyncMock()
    mock_repo.create = AsyncMock(return_value=artist)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    result = await service.create_artist(dto)
    assert result.name == dto.name

# Integration 테스트 - service + repository + 데이터베이스 테스트
@pytest.mark.asyncio
async def test_create_artist_service_integration(test_db):
    # 실제 repository, 실제 데이터베이스
    session = test_db
    service = ArtistService(session)

    result = await service.create_artist(dto)

    # 데이터베이스에서 검증
    saved = await session.get(Artist, result.id)
    assert saved.name == dto.name
```

---

## Integration 테스트 구조

### 디렉토리 구성

```
tests/
  unit/                     # Unit 테스트
    domain/
      artist/
        test_artist_repository.py
        test_artist_service.py

  integration/              # Integration 테스트
    test_artist_api.py      # API 엔드포인트 테스트
    test_artist_flow.py     # 전체 워크플로우 테스트
    test_auth_flow.py
    conftest.py             # Integration fixture
```

---

## 데이터베이스 Integration 테스팅

### 권장 전략: PostgreSQL 테스트 데이터베이스

**Integration 테스트에는 실제 PostgreSQL 테스트 데이터베이스를 사용하세요.** 프로덕션과의 호환성을 보장하고, PostgreSQL 특유의 기능(ARRAY, JSONB 등)을 테스트할 수 있습니다.

#### 테스트 데이터베이스 설정

**로컬 개발 (docker-compose):**
```bash
cd backend
docker-compose up test-db  # 포트 5433에서 PostgreSQL 시작
```

**GitHub Actions CI:**
워크플로우가 자동으로 포트 5432에 PostgreSQL 서비스 컨테이너를 제공합니다.

#### 구현:

```python
import os
import pytest
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine
from sqlmodel import SQLModel
from sqlmodel.ext.asyncio.session import AsyncSession

# 환경 변수에서 테스트 데이터베이스 URL 가져오기, 없으면 기본값 사용
TEST_DATABASE_URL = os.getenv(
    "TEST_DATABASE_URL",
    "postgresql+asyncpg://test_user:test_password@localhost:5433/test_qwarty"
)

@pytest.fixture(scope="session")
async def test_engine() -> AsyncEngine:
    """
    PostgreSQL 테스트 데이터베이스 엔진.

    로컬: postgresql+asyncpg://test_user:test_password@localhost:5433/test_qwarty
    CI:   postgresql+asyncpg://test_user:test_password@localhost:5432/test_qwarty
    """
    engine = create_async_engine(
        TEST_DATABASE_URL,
        echo=False,
        future=True,
        pool_pre_ping=True,  # 사용 전 연결 상태 확인
    )

    # 모든 테이블 생성
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

    yield engine

    # 정리 - 모든 테스트 완료 후 테이블 삭제
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.drop_all)

    await engine.dispose()

@pytest.fixture(scope="function")
async def db_session(test_engine: AsyncEngine):
    """
    자동 롤백이 있는 데이터베이스 세션.

    각 테스트는 새 세션을 받으며, PostgreSQL 트랜잭션 롤백을 사용해
    모든 변경사항을 롤백하므로 테스트 격리가 보장됩니다.
    """
    connection = await test_engine.connect()
    transaction = await connection.begin()

    async with AsyncSession(bind=connection, expire_on_commit=False) as session:
        yield session

    # 트랜잭션 롤백 (정리 - 모든 변경사항 되돌리기)
    await transaction.rollback()
    await connection.close()

# 테스트에서 사용
@pytest.mark.asyncio
async def test_with_postgresql(db_session):
    repository = ArtistRepository(db_session)

    # 아티스트 생성
    artist = Artist(id="1", name="Test", bio="Bio")
    created = await repository.create(artist)

    # 저장 확인
    retrieved = await repository.get_by_id("1")
    assert retrieved.name == "Test"
    # 테스트 후 트랜잭션 롤백됨
```

### 테스트 실행

**로컬 (docker-compose 사용):**
```bash
# 테스트 데이터베이스 시작
cd backend
docker-compose up -d test-db

# Integration 테스트 실행
pytest tests/integration/ -v

# 테스트 데이터베이스 중지
docker-compose down
```

**CI (GitHub Actions):**
```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
      POSTGRES_DB: test_qwarty
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5

steps:
  - name: Run integration tests
    env:
      TEST_DATABASE_URL: postgresql+asyncpg://test_user:test_password@localhost:5432/test_qwarty
    run: pytest tests/integration/ -v
```

---

## API 라우트 테스팅 (FastAPI)

### TestClient 사용

```python
from fastapi.testclient import TestClient
from backend.main import create_application

@pytest.fixture
def client():
    """FastAPI 테스트 클라이언트."""
    app = create_application()
    return TestClient(app)

def test_create_artist_endpoint(client):
    # Act
    response = client.post(
        "/api/v1/artists",
        json={"name": "Test Artist", "bio": "Test bio"}
    )

    # Assert
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test Artist"
    assert "id" in data

def test_get_artist_endpoint(client):
    # Arrange - 먼저 아티스트 생성
    create_response = client.post(
        "/api/v1/artists",
        json={"name": "Test", "bio": "Bio"}
    )
    artist_id = create_response.json()["id"]

    # Act
    response = client.get(f"/api/v1/artists/{artist_id}")

    # Assert
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == artist_id
    assert data["name"] == "Test"
```

### 인증이 있는 테스팅

```python
@pytest.fixture
def authenticated_client(client):
    """인증 토큰이 있는 클라이언트."""
    # 토큰 획득을 위해 로그인
    response = client.post(
        "/api/v1/auth/login",
        json={"email": "test@test.com", "password": "password"}
    )
    token = response.json()["access_token"]

    # 클라이언트에 토큰 추가
    client.headers = {"Authorization": f"Bearer {token}"}
    return client

def test_protected_endpoint(authenticated_client):
    response = authenticated_client.get("/api/v1/artists/me")
    assert response.status_code == 200
```

### 에러 응답 테스팅

```python
def test_create_artist_with_invalid_data(client):
    # 빈 이름
    response = client.post(
        "/api/v1/artists",
        json={"name": "", "bio": "Bio"}
    )

    assert response.status_code == 422  # 유효성 검사 에러
    data = response.json()
    assert "detail" in data

def test_get_nonexistent_artist(client):
    response = client.get("/api/v1/artists/nonexistent-id")

    assert response.status_code == 404
    data = response.json()
    assert "not found" in data["detail"].lower()
```

---

## 전체 워크플로우 테스팅

### 완전한 사용자 흐름

```python
@pytest.mark.asyncio
async def test_artist_creation_workflow(client, test_db):
    """완전한 아티스트 생성 워크플로우 테스트."""
    # 1. 아티스트 생성
    create_response = client.post(
        "/api/v1/artists",
        json={"name": "New Artist", "bio": "Artist bio"}
    )
    assert create_response.status_code == 201
    artist_id = create_response.json()["id"]

    # 2. 아티스트 조회
    get_response = client.get(f"/api/v1/artists/{artist_id}")
    assert get_response.status_code == 200
    assert get_response.json()["name"] == "New Artist"

    # 3. 아티스트 업데이트
    update_response = client.put(
        f"/api/v1/artists/{artist_id}",
        json={"name": "Updated Artist", "bio": "New bio"}
    )
    assert update_response.status_code == 200
    assert update_response.json()["name"] == "Updated Artist"

    # 4. 데이터베이스에서 검증
    artist = await test_db.get(Artist, artist_id)
    assert artist.name == "Updated Artist"

    # 5. 아티스트 삭제
    delete_response = client.delete(f"/api/v1/artists/{artist_id}")
    assert delete_response.status_code == 204

    # 6. 삭제 확인
    get_deleted = client.get(f"/api/v1/artists/{artist_id}")
    assert get_deleted.status_code == 404
```

### 멀티 서비스 Integration

```python
@pytest.mark.asyncio
async def test_artist_artwork_relationship(client, test_db):
    """artist와 artwork 서비스 간 integration 테스트."""
    # 1. 아티스트 생성
    artist_response = client.post(
        "/api/v1/artists",
        json={"name": "Artist", "bio": "Bio"}
    )
    artist_id = artist_response.json()["id"]

    # 2. 아티스트의 작품 생성
    artwork_response = client.post(
        "/api/v1/artworks",
        json={
            "title": "Artwork",
            "artist_id": artist_id,
            "price": 1000
        }
    )
    assert artwork_response.status_code == 201
    artwork_id = artwork_response.json()["id"]

    # 3. 아티스트의 작품 목록 조회
    artworks_response = client.get(
        f"/api/v1/artists/{artist_id}/artworks"
    )
    assert artworks_response.status_code == 200
    artworks = artworks_response.json()
    assert len(artworks) == 1
    assert artworks[0]["id"] == artwork_id

    # 4. 아티스트 삭제 시도 (작품이 있으므로 실패해야 함)
    delete_response = client.delete(f"/api/v1/artists/{artist_id}")
    assert delete_response.status_code == 409  # Conflict
```

---

## 외부 서비스 테스팅

### 외부 API 모킹

```python
import respx
from httpx import Response

@pytest.mark.asyncio
@respx.mock
async def test_fetch_external_artist_data():
    """외부 API integration 테스트 (모킹됨)."""
    # 외부 API 모킹
    respx.get("https://api.example.com/artists/123").mock(
        return_value=Response(
            200,
            json={"id": "123", "name": "External Artist"}
        )
    )

    # 외부 API를 호출하는 서비스 테스트
    service = ExternalArtistService()
    result = await service.fetch_artist("123")

    assert result["name"] == "External Artist"
```

### S3 Integration 테스팅

```python
from moto import mock_s3
import boto3

@pytest.fixture
def s3_bucket():
    """테스팅용 모킹 S3 버킷."""
    with mock_s3():
        # 테스트 버킷 생성
        s3 = boto3.client("s3", region_name="us-east-1")
        s3.create_bucket(Bucket="test-bucket")

        yield s3

def test_upload_to_s3(s3_bucket):
    """S3 업로드 integration 테스트."""
    from backend.utils.s3 import upload_file

    # 파일 업로드
    upload_file("test.jpg", "test-bucket", "uploads/test.jpg")

    # 파일이 업로드되었는지 확인
    response = s3_bucket.get_object(
        Bucket="test-bucket",
        Key="uploads/test.jpg"
    )
    assert response["Body"].read() == b"file content"
```

---

## 데이터베이스 테스팅 패턴

### 트랜잭션 테스팅

```python
@pytest.mark.asyncio
async def test_transaction_rollback_on_error(test_db):
    """에러 발생 시 트랜잭션 롤백 테스트."""
    service = ArtistService(test_db)

    # 실패할 아티스트 생성 시도
    try:
        async with test_db.begin():
            await service.create_artist(dto)
            # 에러 시뮬레이션
            raise Exception("의도적인 에러")
    except Exception:
        pass

    # 아티스트가 저장되지 않았는지 확인
    artists = await service.get_all_artists()
    assert len(artists) == 0
```

### 관계 테스팅

```python
@pytest.mark.asyncio
async def test_artist_artworks_relationship(test_db):
    """artist와 artwork 간 데이터베이스 관계 테스트."""
    # 아티스트 생성
    artist = Artist(id="1", name="Test")
    test_db.add(artist)
    await test_db.commit()

    # 작품 생성
    artwork1 = Artwork(id="a1", title="Work 1", artist_id="1")
    artwork2 = Artwork(id="a2", title="Work 2", artist_id="1")
    test_db.add(artwork1)
    test_db.add(artwork2)
    await test_db.commit()

    # 관계 로드를 위해 refresh
    await test_db.refresh(artist)

    # 관계 검증
    assert len(artist.artworks) == 2
    assert artist.artworks[0].title == "Work 1"
```

### Unique 제약 조건 테스팅

```python
@pytest.mark.asyncio
async def test_unique_constraint_violation(test_db):
    """데이터베이스 unique 제약 조건 테스트."""
    # 첫 번째 아티스트 생성
    artist1 = Artist(id="1", name="Artist", email="test@test.com")
    test_db.add(artist1)
    await test_db.commit()

    # 동일한 이메일로 두 번째 아티스트 생성 시도
    artist2 = Artist(id="2", name="Artist 2", email="test@test.com")
    test_db.add(artist2)

    # 무결성 에러가 발생해야 함
    with pytest.raises(IntegrityError):
        await test_db.commit()
```

---

## 미들웨어 테스팅

### 에러 핸들러 미들웨어

```python
def test_error_handler_middleware(client):
    """에러 핸들러 미들웨어 동작 테스트."""
    # 에러 트리거
    response = client.get("/api/v1/artists/trigger-error")

    # 에러 응답 형식 검증
    assert response.status_code == 500
    data = response.json()
    assert "detail" in data
    assert "error" in data["detail"].lower()
```

### 인증 미들웨어

```python
def test_authentication_middleware(client):
    """인증 미들웨어 테스트."""
    # 토큰 없이
    response = client.get("/api/v1/protected")
    assert response.status_code == 401

    # 잘못된 토큰으로
    client.headers = {"Authorization": "Bearer invalid"}
    response = client.get("/api/v1/protected")
    assert response.status_code == 401

    # 유효한 토큰으로
    client.headers = {"Authorization": f"Bearer {valid_token}"}
    response = client.get("/api/v1/protected")
    assert response.status_code == 200
```

---

## 성능 Integration 테스트

### 쿼리 성능 테스팅

```python
@pytest.mark.performance
@pytest.mark.asyncio
async def test_artist_list_performance(test_db):
    """아티스트 목록 쿼리 성능 테스트."""
    # 많은 아티스트 생성
    for i in range(1000):
        artist = Artist(id=f"id-{i}", name=f"Artist {i}")
        test_db.add(artist)
    await test_db.commit()

    # 쿼리 시간 측정
    import time
    start = time.time()

    repository = ArtistRepository(test_db)
    artists = await repository.find_all(limit=100)

    duration = time.time() - start

    # 성능 검증
    assert len(artists) == 100
    assert duration < 0.5  # 500ms 이내에 완료되어야 함
```

---

## 모범 사례

### 1. 테스트 독립성 유지

```python
# 각 테스트는 자체 데이터를 생성
@pytest.mark.asyncio
async def test_1(test_db):
    artist = Artist(id="1", name="Test")
    test_db.add(artist)
    await test_db.commit()
    # 이 아티스트로 테스트

@pytest.mark.asyncio
async def test_2(test_db):
    # 자체 아티스트 생성, test_1에 의존하지 않음
    artist = Artist(id="2", name="Test 2")
    test_db.add(artist)
    await test_db.commit()
```

### 2. 테스트 데이터에 팩토리 사용

```python
from datetime import datetime

def create_artist(**kwargs):
    """테스트 아티스트 생성을 위한 팩토리."""
    defaults = {
        "id": f"artist-{datetime.now().timestamp()}",
        "name": "Test Artist",
        "bio": "Test bio",
    }
    defaults.update(kwargs)
    return Artist(**defaults)

# 테스트에서 사용
@pytest.mark.asyncio
async def test_with_factory(test_db):
    artist = create_artist(name="Custom Name")
    test_db.add(artist)
    await test_db.commit()
```

### 3. 테스트 후 정리

```python
@pytest.fixture
async def test_db():
    """자동 정리가 있는 데이터베이스."""
    # 설정
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

    # 정리 - 테스트 후 자동으로 실행됨
    await engine.dispose()
```

---

## 체크리스트

Integration 테스트 작성 시:

- [ ] 개별 단위가 아닌 컴포넌트 상호작용 테스트
- [ ] 실제 데이터베이스 사용 (인메모리 또는 테스트 인스턴스)
- [ ] API에서 데이터베이스까지 전체 워크플로우 테스트
- [ ] 작업 후 데이터베이스 상태 검증
- [ ] 에러 시나리오와 롤백 테스트
- [ ] 외부 서비스만 모킹 (필요시)
- [ ] 정리를 통해 테스트 독립성 유지
- [ ] 테스트 데이터 생성에 팩토리 사용
- [ ] 인증 및 권한 테스트
- [ ] 관계 로딩과 제약 조건 검증

---

## 관련 리소스

- [testing-architecture.md](testing-architecture.md) - 전체 테스트 전략
- [unit-testing.md](unit-testing.md) - 보완적인 unit 테스트
- [fastapi-testing.md](fastapi-testing.md) - FastAPI 특화 패턴
