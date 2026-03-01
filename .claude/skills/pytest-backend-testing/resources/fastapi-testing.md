# FastAPI 테스팅

## 개요

FastAPI는 TestClient 클래스와 의존성 주입 시스템을 통해 뛰어난 테스팅 지원을 제공합니다. 이 가이드는 FastAPI 특화 테스팅 패턴, 요청/응답 테스팅, 그리고 의존성 재정의를 다룹니다.

## TestClient 기본

### TestClient 설정

```python
from fastapi.testclient import TestClient
from backend.main import create_application

@pytest.fixture
def client():
    """FastAPI 테스트 클라이언트."""
    app = create_application()
    return TestClient(app)

# 테스트에서 사용
def test_health_endpoint(client):
    response = client.get("/health")
    assert response.status_code == 200
```

### TestClient vs httpx

TestClient는 동기 방식 (httpx를 래핑) - 테스팅에 이상적:

```python
# TestClient로 동기 테스트 (올바른 방법)
def test_endpoint(client):
    response = client.get("/api/v1/artists")
    assert response.status_code == 200

# TestClient에서 async는 불필요 (잘못된 방법)
@pytest.mark.asyncio
async def test_endpoint(client):  # 불필요한 async
    response = client.get("/api/v1/artists")
```

---

## HTTP 메서드 테스팅

### GET 요청

```python
def test_get_artist(client):
    # 단순 GET
    response = client.get("/api/v1/artists/123")

    assert response.status_code == 200
    data = response.json()
    assert data["id"] == "123"

def test_get_with_query_params(client):
    # 쿼리 파라미터가 있는 GET
    response = client.get(
        "/api/v1/artists",
        params={"page": 1, "limit": 10, "sort": "name"}
    )

    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) <= 10
```

### POST 요청

```python
def test_create_artist(client):
    # JSON 바디가 있는 POST
    response = client.post(
        "/api/v1/artists",
        json={"name": "New Artist", "bio": "Artist bio"}
    )

    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "New Artist"
    assert "id" in data

def test_create_with_headers(client):
    # 커스텀 헤더가 있는 POST
    response = client.post(
        "/api/v1/artists",
        json={"name": "Artist", "bio": "Bio"},
        headers={"Content-Type": "application/json"}
    )

    assert response.status_code == 201
```

### PUT 및 PATCH 요청

```python
def test_update_artist_put(client):
    # PUT (전체 업데이트)
    response = client.put(
        "/api/v1/artists/123",
        json={"name": "Updated", "bio": "New bio"}
    )

    assert response.status_code == 200
    assert response.json()["name"] == "Updated"

def test_update_artist_patch(client):
    # PATCH (부분 업데이트)
    response = client.patch(
        "/api/v1/artists/123",
        json={"bio": "Updated bio only"}
    )

    assert response.status_code == 200
```

### DELETE 요청

```python
def test_delete_artist(client):
    response = client.delete("/api/v1/artists/123")

    assert response.status_code == 204
    assert response.content == b""

def test_delete_nonexistent(client):
    response = client.delete("/api/v1/artists/nonexistent")

    assert response.status_code == 404
```

---

## 요청 유효성 검사 테스팅

### Pydantic 유효성 검사 에러

```python
def test_create_artist_invalid_data(client):
    # 빈 이름 (유효성 검사 실패해야 함)
    response = client.post(
        "/api/v1/artists",
        json={"name": "", "bio": "Bio"}
    )

    assert response.status_code == 422  # 처리 불가 엔티티
    data = response.json()
    assert "detail" in data
    assert any(
        error["loc"] == ["body", "name"]
        for error in data["detail"]
    )

def test_create_artist_missing_required_field(client):
    # 필수 필드 누락
    response = client.post(
        "/api/v1/artists",
        json={"bio": "Bio"}  # 'name' 누락
    )

    assert response.status_code == 422
    errors = response.json()["detail"]
    assert any(
        error["loc"] == ["body", "name"]
        for error in errors
    )
```

### 커스텀 유효성 검사

```python
def test_artist_name_length_validation(client):
    # 이름이 너무 긴 경우
    long_name = "a" * 256
    response = client.post(
        "/api/v1/artists",
        json={"name": long_name, "bio": "Bio"}
    )

    assert response.status_code == 422

def test_artist_email_format_validation(client):
    # 이메일 형식이 잘못된 경우
    response = client.post(
        "/api/v1/artists",
        json={
            "name": "Artist",
            "email": "invalid-email",
            "bio": "Bio"
        }
    )

    assert response.status_code == 422
```

---

## 응답 형식 테스팅

### JSON 응답

```python
def test_response_format(client):
    response = client.get("/api/v1/artists/123")

    # 상태 코드
    assert response.status_code == 200

    # Content-Type
    assert response.headers["content-type"] == "application/json"

    # JSON 데이터
    data = response.json()
    assert isinstance(data, dict)
    assert "id" in data
    assert "name" in data
    assert "bio" in data

def test_list_response_format(client):
    response = client.get("/api/v1/artists")

    data = response.json()
    assert isinstance(data, list)
    assert all(isinstance(item, dict) for item in data)
```

### 페이지네이션 응답

```python
def test_paginated_response(client):
    response = client.get("/api/v1/artists?page=1&limit=10")

    assert response.status_code == 200
    data = response.json()

    # 페이지네이션 구조 확인
    assert "items" in data
    assert "total" in data
    assert "page" in data
    assert "limit" in data
    assert len(data["items"]) <= 10
```

---

## 의존성 재정의

### 데이터베이스 세션 재정의

```python
from fastapi import FastAPI
from backend.db.orm import get_read_session_dependency

@pytest.fixture
def client_with_test_db(test_db):
    """데이터베이스 의존성이 재정의된 클라이언트."""
    app = create_application()

    # 의존성 재정의
    async def override_get_db():
        yield test_db

    app.dependency_overrides[get_read_session_dependency] = override_get_db

    client = TestClient(app)

    yield client

    # 재정의 초기화
    app.dependency_overrides.clear()

# 테스트에서 사용
def test_with_test_db(client_with_test_db):
    # 실제 데이터베이스 대신 테스트 데이터베이스 사용
    response = client_with_test_db.get("/api/v1/artists")
    assert response.status_code == 200
```

### 인증 재정의

```python
from backend.api.dependencies import get_current_user

@pytest.fixture
def authenticated_client():
    """인증이 우회된 클라이언트."""
    app = create_application()

    # mock 사용자
    mock_user = User(id="test-user", email="test@test.com")

    # 인증 의존성 재정의
    async def override_auth():
        return mock_user

    app.dependency_overrides[get_current_user] = override_auth

    client = TestClient(app)

    yield client

    app.dependency_overrides.clear()

def test_protected_route(authenticated_client):
    # 인증 우회
    response = authenticated_client.get("/api/v1/protected")
    assert response.status_code == 200
```

### 외부 서비스 재정의

```python
from backend.services.s3 import get_s3_client

@pytest.fixture
def client_with_mock_s3():
    """S3 서비스가 모킹된 클라이언트."""
    app = create_application()

    # S3 클라이언트 모킹
    mock_s3 = MagicMock()
    mock_s3.upload_file.return_value = "https://example.com/file.jpg"

    def override_s3():
        return mock_s3

    app.dependency_overrides[get_s3_client] = override_s3

    client = TestClient(app)

    yield client

    app.dependency_overrides.clear()

def test_file_upload(client_with_mock_s3):
    # S3가 모킹됨
    response = client_with_mock_s3.post(
        "/api/v1/upload",
        files={"file": ("test.jpg", b"content", "image/jpeg")}
    )
    assert response.status_code == 200
```

---

## 인증 테스팅

### JWT 토큰 인증

```python
import jwt
from datetime import datetime, timedelta

@pytest.fixture
def auth_token():
    """테스팅용 유효한 JWT 토큰 생성."""
    payload = {
        "sub": "test-user-id",
        "email": "test@test.com",
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    token = jwt.encode(payload, "secret-key", algorithm="HS256")
    return token

def test_protected_endpoint_without_token(client):
    response = client.get("/api/v1/protected")
    assert response.status_code == 401

def test_protected_endpoint_with_token(client, auth_token):
    response = client.get(
        "/api/v1/protected",
        headers={"Authorization": f"Bearer {auth_token}"}
    )
    assert response.status_code == 200

def test_expired_token(client):
    # 만료된 토큰 생성
    payload = {
        "sub": "user-id",
        "exp": datetime.utcnow() - timedelta(hours=1)  # 만료됨
    }
    expired_token = jwt.encode(payload, "secret-key", algorithm="HS256")

    response = client.get(
        "/api/v1/protected",
        headers={"Authorization": f"Bearer {expired_token}"}
    )
    assert response.status_code == 401
```

### 쿠키 기반 인증

```python
def test_login_sets_cookie(client):
    # 로그인
    response = client.post(
        "/api/v1/auth/login",
        json={"email": "test@test.com", "password": "password"}
    )

    assert response.status_code == 200

    # 쿠키가 설정되었는지 확인
    assert "session" in response.cookies

def test_authenticated_request_with_cookie(client):
    # 먼저 로그인
    login_response = client.post(
        "/api/v1/auth/login",
        json={"email": "test@test.com", "password": "password"}
    )

    # 이후 요청에 쿠키 사용
    response = client.get(
        "/api/v1/protected",
        cookies=login_response.cookies
    )

    assert response.status_code == 200
```

---

## 에러 처리 테스팅

### 404 Not Found

```python
def test_get_nonexistent_artist(client):
    response = client.get("/api/v1/artists/nonexistent-id")

    assert response.status_code == 404
    data = response.json()
    assert "detail" in data
    assert "not found" in data["detail"].lower()
```

### 409 Conflict

```python
def test_create_duplicate_artist(client):
    # 첫 번째 아티스트 생성
    client.post(
        "/api/v1/artists",
        json={"name": "Artist", "email": "test@test.com"}
    )

    # 중복 생성 시도
    response = client.post(
        "/api/v1/artists",
        json={"name": "Artist 2", "email": "test@test.com"}
    )

    assert response.status_code == 409
    assert "already exists" in response.json()["detail"].lower()
```

### 500 Internal Server Error

```python
def test_internal_server_error_handling(client, mocker):
    # 예외를 발생시키도록 service 모킹
    mocker.patch(
        "backend.domain.artist.service.ArtistService.get_artist",
        side_effect=Exception("Database error")
    )

    response = client.get("/api/v1/artists/123")

    assert response.status_code == 500
    data = response.json()
    assert "detail" in data
```

---

## 파일 업로드 테스팅

### 단일 파일 업로드

```python
def test_upload_file(client):
    # 테스트 파일 생성
    file_content = b"fake image content"

    response = client.post(
        "/api/v1/upload",
        files={"file": ("test.jpg", file_content, "image/jpeg")}
    )

    assert response.status_code == 200
    data = response.json()
    assert "url" in data
```

### 다중 파일 업로드

```python
def test_upload_multiple_files(client):
    files = [
        ("files", ("file1.jpg", b"content1", "image/jpeg")),
        ("files", ("file2.jpg", b"content2", "image/jpeg")),
    ]

    response = client.post("/api/v1/upload/multiple", files=files)

    assert response.status_code == 200
    data = response.json()
    assert len(data["urls"]) == 2
```

### 파일 크기 유효성 검사

```python
def test_upload_file_too_large(client):
    # 제한보다 큰 파일 생성 (예: 10MB)
    large_file = b"x" * (11 * 1024 * 1024)

    response = client.post(
        "/api/v1/upload",
        files={"file": ("large.jpg", large_file, "image/jpeg")}
    )

    assert response.status_code == 413  # Payload Too Large
```

---

## 백그라운드 태스크 테스팅

### 백그라운드 태스크 실행 검증

```python
from unittest.mock import MagicMock, patch

def test_endpoint_triggers_background_task(client):
    with patch("backend.tasks.send_email") as mock_send_email:
        response = client.post(
            "/api/v1/artists",
            json={"name": "Artist", "email": "test@test.com"}
        )

        assert response.status_code == 201

        # 백그라운드 태스크가 호출되었는지 확인
        mock_send_email.assert_called_once()
        assert mock_send_email.call_args[0][0] == "test@test.com"
```

---

## 스트리밍 응답 테스팅

### Server-Sent Events (SSE)

```python
def test_sse_endpoint(client):
    with client.stream("GET", "/api/v1/stream") as response:
        assert response.status_code == 200
        assert response.headers["content-type"] == "text/event-stream"

        # 이벤트 읽기
        events = []
        for line in response.iter_lines():
            if line.startswith("data:"):
                events.append(line[5:])

        assert len(events) > 0
```

---

## WebSocket 엔드포인트 테스팅

### WebSocket 테스팅

```python
def test_websocket(client):
    with client.websocket_connect("/ws") as websocket:
        # 메시지 전송
        websocket.send_text("Hello")

        # 응답 수신
        data = websocket.receive_text()

        assert data == "Hello back"

def test_websocket_json(client):
    with client.websocket_connect("/ws/json") as websocket:
        # JSON 전송
        websocket.send_json({"type": "ping"})

        # JSON 수신
        data = websocket.receive_json()

        assert data["type"] == "pong"
```

---

## 성능 테스팅

### 응답 시간

```python
import time

def test_response_time(client):
    start = time.time()

    response = client.get("/api/v1/artists")

    duration = time.time() - start

    assert response.status_code == 200
    assert duration < 0.1  # 100ms 이내에 응답해야 함
```

### 부하 테스팅 (단순화)

```python
@pytest.mark.performance
def test_endpoint_under_load(client):
    """엔드포인트가 여러 요청을 처리하는지 테스트."""
    results = []

    # 100개 요청 전송
    for _ in range(100):
        response = client.get("/api/v1/artists")
        results.append(response.status_code)

    # 모두 성공해야 함
    assert all(status == 200 for status in results)
```

---

## 모범 사례

### 1. 공통 설정에 Fixture 사용

```python
@pytest.fixture
def sample_artist(client):
    """테스팅용 샘플 아티스트 생성."""
    response = client.post(
        "/api/v1/artists",
        json={"name": "Test Artist", "bio": "Bio"}
    )
    return response.json()

def test_update_artist(client, sample_artist):
    artist_id = sample_artist["id"]

    response = client.put(
        f"/api/v1/artists/{artist_id}",
        json={"name": "Updated", "bio": "New bio"}
    )

    assert response.status_code == 200
```

### 2. 성공과 에러 케이스 모두 테스트

```python
def test_create_artist_success(client):
    response = client.post(
        "/api/v1/artists",
        json={"name": "Artist", "bio": "Bio"}
    )
    assert response.status_code == 201

def test_create_artist_validation_error(client):
    response = client.post(
        "/api/v1/artists",
        json={"name": "", "bio": "Bio"}
    )
    assert response.status_code == 422

def test_create_artist_duplicate_error(client):
    # 첫 번째 생성
    client.post("/api/v1/artists", json={"name": "Artist", "bio": "Bio"})

    # 중복 시도
    response = client.post(
        "/api/v1/artists",
        json={"name": "Artist", "bio": "Bio"}
    )
    assert response.status_code == 409
```

### 3. 응답 구조 검증

```python
def test_artist_response_structure(client):
    response = client.get("/api/v1/artists/123")

    data = response.json()

    # 모든 필수 필드가 있는지 확인
    assert "id" in data
    assert "name" in data
    assert "bio" in data
    assert "created_at" in data

    # 타입 검증
    assert isinstance(data["id"], str)
    assert isinstance(data["name"], str)
```

---

## 체크리스트

FastAPI 엔드포인트 테스팅 시:

- [ ] HTTP 테스팅에 TestClient 사용
- [ ] 모든 HTTP 메서드 테스트 (GET, POST, PUT, PATCH, DELETE)
- [ ] 상태 코드 검증
- [ ] 요청 유효성 검사 테스트 (422 에러)
- [ ] 인증 및 권한 테스트
- [ ] 의존성 재정의로 모킹
- [ ] 에러 응답 테스트 (404, 409, 500)
- [ ] 응답 구조와 타입 검증
- [ ] 해당하는 경우 파일 업로드 테스트
- [ ] 해당하는 경우 페이지네이션 테스트
- [ ] 외부 서비스 모킹
- [ ] 성공과 실패 경로 모두 테스트

---

## 관련 리소스

- [integration-testing.md](integration-testing.md) - 전체 워크플로우 테스팅
- [unit-testing.md](unit-testing.md) - 격리된 컴포넌트 테스팅
- [testing-architecture.md](testing-architecture.md) - 전체 테스팅 전략
