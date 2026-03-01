# 커버리지 모범 사례

## 개요

코드 커버리지는 테스트 중에 얼마나 많은 코드가 실행되는지를 측정합니다. 이 프로젝트는 80% 이상의 커버리지를 요구합니다. 이 가이드는 높은 커버리지를 효과적으로 달성하고 유지하는 방법을 다룹니다.

## 현재 프로젝트 설정

### pytest-cov 설정

`pyproject.toml`에서:

```toml
[tool.pytest.ini_options]
addopts = [
    "--cov=backend",
    "--cov-report=term-missing",
    "--cov-report=html",
    "--cov-fail-under=80",
]

[tool.coverage.run]
source = ["backend"]
omit = [
    "backend/__init__.py",
    "*/tests/*",
    "tests/*",
    "**/tests/*",
    "conftest.py",
    "*/conftest.py",
    "**/conftest.py",
    "backend/utils/pdf/*",
    "backend/utils/s3.py",
    "backend/api/v1/routers/*",
    "backend/main.py",
    "backend/domain/auth/*",
    "backend/domain/curai/*",
    "backend/domain/admin/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "raise AssertionError",
    "raise NotImplementedError",
]
```

---

## 커버리지 실행

### 기본 명령어

```bash
# 커버리지와 함께 테스트 실행
pytest --cov=backend

# 누락된 라인 표시
pytest --cov=backend --cov-report=term-missing

# HTML 보고서 생성
pytest --cov=backend --cov-report=html

# HTML 보고서 열기
open htmlcov/index.html  # macOS
# 또는
xdg-open htmlcov/index.html  # Linux

# 최소 임계값 확인
pytest --cov=backend --cov-fail-under=80
```

### 특정 모듈의 커버리지

```bash
# 특정 도메인의 커버리지
pytest --cov=backend.domain.artist tests/unit/domain/artist/

# 특정 파일의 커버리지
pytest --cov=backend.domain.artist.service tests/unit/domain/artist/test_artist_service.py
```

---

## 커버리지 보고서 이해

### 터미널 보고서

```
---------- coverage: platform darwin, python 3.12.3 ----------
Name                                    Stmts   Miss  Cover   Missing
---------------------------------------------------------------------
backend/domain/artist/model.py             12      0   100%
backend/domain/artist/repository.py        45      3    93%   78-80
backend/domain/artist/service.py           68      8    88%   45, 67-74
---------------------------------------------------------------------
TOTAL                                     125     11    91%
```

**보고서 읽기:**
- **Stmts**: 파일의 총 구문 수
- **Miss**: 테스트로 커버되지 않은 구문 수
- **Cover**: 커버된 비율
- **Missing**: 커버되지 않은 라인 번호

### HTML 보고서

HTML 보고서는 다음을 제공합니다:
- 커버된/커버되지 않은 코드의 시각적 하이라이팅
- 브랜치 커버리지 (if/else 경로)
- 라인별 상세 분석
- 정렬 가능한 열
- 검색 기능

---

## 커버리지를 위해 무엇을 테스트할 것인가

### 우선순위 1: 핵심 비즈니스 로직

```python
# 비즈니스 로직은 반드시 테스트해야 함
class ArtistService:
    async def create_artist(self, dto: ArtistRequestDto):
        # 유일성 검증 - 반드시 테스트
        existing = await self._repository.find_by_name(dto.name)
        if existing:
            raise ConflictError("Artist already exists")

        # 아티스트 생성 - 반드시 테스트
        artist = Artist(id=generate_id(), name=dto.name, bio=dto.bio)
        created = await self._repository.create(artist)

        # DTO 반환 - 반드시 테스트
        return ArtistResponseDto.from_model(created)
```

### 우선순위 2: 에러 처리

```python
# 에러 경로 테스트 필수
class ArtistService:
    async def get_artist(self, artist_id: str):
        artist = await self._repository.get_by_id(artist_id)

        # 에러 경로 - 반드시 테스트
        if not artist:
            raise NotFoundError(f"Artist {artist_id} not found")

        return ArtistResponseDto.from_model(artist)
```

### 우선순위 3: 엣지 케이스

```python
# 엣지 케이스 테스트
def validate_artist_name(name: str):
    # 빈 이름 - 테스트
    if not name.strip():
        raise ValueError("Name cannot be empty")

    # 너무 긴 이름 - 테스트
    if len(name) > 255:
        raise ValueError("Name too long")

    # 너무 짧은 이름 - 테스트
    if len(name) < 2:
        raise ValueError("Name too short")

    return name.strip()
```

### 낮은 우선순위: 단순 Getter/Setter

```python
# 단순 속성 접근 - 낮은 우선순위
@property
def full_name(self):
    return f"{self.first_name} {self.last_name}"

# 테스트할 수 있지만, 커버리지에 있어 중요하지 않음
```

---

## 커버리지 향상

### 커버되지 않은 라인 찾기

```bash
# 누락 보고서와 함께 실행
pytest --cov=backend --cov-report=term-missing

# "Missing" 열 확인
# 예시 출력:
# backend/domain/artist/service.py    88%   45, 67-74
```

### 커버되지 않은 코드 분석

```python
# 예시: 라인 67-74가 커버되지 않음
class ArtistService:
    async def update_artist(self, artist_id: str, dto: ArtistRequestDto):
        artist = await self._repository.get_by_id(artist_id)
        if not artist:
            raise NotFoundError("Artist not found")

        # 라인 67-74 커버되지 않음 - 테스트 누락!
        if dto.name != artist.name:
            existing = await self._repository.find_by_name(dto.name)
            if existing and existing.id != artist_id:
                raise ConflictError("Name already in use")

        artist.name = dto.name
        artist.bio = dto.bio
        await self._repository.update(artist)
        return ArtistResponseDto.from_model(artist)
```

### 커버되지 않은 코드에 대한 테스트 작성

```python
@pytest.mark.asyncio
async def test_update_artist_with_duplicate_name_raises_conflict():
    """커버되지 않은 라인 67-74에 대한 테스트."""
    # Arrange
    mock_repo = AsyncMock()

    # 업데이트되는 아티스트
    artist = Artist(id="1", name="Old Name", bio="Bio")
    mock_repo.get_by_id = AsyncMock(return_value=artist)

    # 새 이름을 가진 다른 아티스트
    existing = Artist(id="2", name="New Name", bio="Bio")
    mock_repo.find_by_name = AsyncMock(return_value=existing)

    service = ArtistService(mock_session)
    service._repository = mock_repo

    # Act & Assert - 라인 67-74 커버됨
    with pytest.raises(ConflictError, match="Name already in use"):
        await service.update_artist("1", ArtistRequestDto(name="New Name"))
```

---

## 커버리지 전략

### 전략 1: 모든 브랜치 테스트

```python
# 브랜치가 있는 코드
def calculate_discount(total: float, is_member: bool) -> float:
    if is_member:
        if total > 100:
            return total * 0.20  # 브랜치 1
        else:
            return total * 0.10  # 브랜치 2
    else:
        return 0  # 브랜치 3

# 모든 브랜치에 대한 테스트
def test_member_discount_high_total():
    assert calculate_discount(150, True) == 30  # 브랜치 1

def test_member_discount_low_total():
    assert calculate_discount(50, True) == 5  # 브랜치 2

def test_non_member_no_discount():
    assert calculate_discount(150, False) == 0  # 브랜치 3
```

### 전략 2: 예외 경로 테스트

```python
# 예외가 있는 코드
class ArtistService:
    async def delete_artist(self, artist_id: str):
        artist = await self._repository.get_by_id(artist_id)

        # 예외 경로
        if not artist:
            raise NotFoundError("Artist not found")

        # 작품 확인
        artworks = await self._artwork_repository.find_by_artist(artist_id)
        if artworks:
            raise ConflictError("Cannot delete artist with artworks")

        await self._repository.delete(artist_id)

# 예외 경로 테스트
@pytest.mark.asyncio
async def test_delete_artist_not_found():
    mock_repo.get_by_id = AsyncMock(return_value=None)
    with pytest.raises(NotFoundError):
        await service.delete_artist("1")

@pytest.mark.asyncio
async def test_delete_artist_with_artworks():
    mock_repo.get_by_id = AsyncMock(return_value=artist)
    mock_artwork_repo.find_by_artist = AsyncMock(return_value=[artwork])
    with pytest.raises(ConflictError):
        await service.delete_artist("1")
```

### 전략 3: 엣지 케이스 파라미터화

```python
@pytest.mark.parametrize("name,should_raise", [
    ("Valid Name", False),           # 일반 케이스
    ("", True),                       # 빈 값
    ("  ", True),                     # 공백
    ("a" * 256, True),                # 너무 긴 이름
    ("AB", False),                    # 최소 유효 길이
    ("A" * 255, False),               # 최대 유효 길이
])
def test_name_validation(name, should_raise):
    if should_raise:
        with pytest.raises(ValueError):
            validate_name(name)
    else:
        assert validate_name(name) == name.strip()
```

---

## 커버리지에서 코드 제외

### pragma: no cover 사용

```python
# 디버그/개발 코드 제외
def debug_print(msg: str):  # pragma: no cover
    """개발 중에만 사용됩니다."""
    print(f"DEBUG: {msg}")

# 추상 메서드 제외
class BaseRepository:
    def get_by_id(self, id: str):
        raise NotImplementedError  # pragma: no cover
```

### 설정 사용

`pyproject.toml`에 이미 설정되어 있음:

```toml
[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "raise AssertionError",
    "raise NotImplementedError",
]
```

### 전체 파일 제외

```toml
[tool.coverage.run]
omit = [
    "*/tests/*",
    "backend/main.py",  # 애플리케이션 진입점
    "backend/utils/pdf/*",  # 외부 라이브러리 래퍼
]
```

---

## 커버리지 안티패턴

### 커버리지를 위해 테스트를 작성하지 말 것

```python
# 커버리지만을 위한 테스트 (잘못된 방법)
def test_getter():
    artist = Artist(id="1", name="Test")
    assert artist.name == "Test"  # 의미 없는 테스트

# 실제 동작을 테스트하는 것 (올바른 방법)
@pytest.mark.asyncio
async def test_create_artist_validates_unique_name():
    # 비즈니스 로직 테스트, 단순 속성 접근이 아님
    with pytest.raises(ConflictError):
        await service.create_artist(duplicate_name_dto)
```

### 커버리지 비율에만 집중하지 말 것

```python
# 100% 커버리지이지만 부실한 테스트 (잘못된 방법)
def test_everything():
    # 모든 라인을 건드리는 하나의 거대한 테스트
    artist = create_artist()
    update_artist(artist)
    delete_artist(artist)
    # 커버리지는 달성하지만 동작을 제대로 테스트하지 않음

# 의미 있는 단언이 있는 집중된 테스트 (올바른 방법)
def test_create_artist():
    artist = create_artist(name="Test")
    assert artist.name == "Test"

def test_update_artist():
    updated = update_artist(artist, new_name="Updated")
    assert updated.name == "Updated"

def test_delete_artist():
    delete_artist(artist)
    with pytest.raises(NotFoundError):
        get_artist(artist.id)
```

### 구현 세부사항을 테스트하지 말 것

```python
# 어떻게 하는지를 테스트 (잘못된 방법)
@pytest.mark.asyncio
async def test_service_calls_repository():
    # 구현에 너무 집중
    await service.get_artist("1")
    mock_repo.get_by_id.assert_called_once()  # 호출만 검증

# 동작을 테스트 (올바른 방법)
@pytest.mark.asyncio
async def test_service_returns_artist():
    result = await service.get_artist("1")
    assert result.id == "1"  # 실제 동작 검증
    assert result.name == "Test"
```

---

## 커버리지 목표

### 현실적인 목표

```
전체: 80%+ (프로젝트 요구사항)

계층별:
- 비즈니스 로직 (Service): 90%+
- 데이터 접근 (Repository): 85%+
- 모델: 70%+ (단순 모델은 적어도 됨)
- 유틸리티: 90%+
- API 라우트: 60%+ (integration 테스트)

제외 대상:
- 설정 파일
- 메인 진입점
- __init__.py 파일
- 외부 라이브러리 래퍼
```

### 시간에 따른 커버리지 추적

```bash
# 커버리지 보고서 생성
pytest --cov=backend --cov-report=html

# 이전 실행과 비교
# htmlcov/를 git 또는 CI 아티팩트에 저장
# CI/CD에서 커버리지 추세 추적
```

---

## CI/CD에서 커버리지

### GitHub Actions 예시

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.12.3'

      - name: Install dependencies
        run: |
          pip install -e .
          pip install -e .[dev]

      - name: Run tests with coverage
        run: |
          pytest --cov=backend --cov-report=xml --cov-fail-under=80

      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          file: ./coverage.xml
```

---

## 모범 사례 요약

### 해야 할 것:

1. **비즈니스 로직에 집중**: 핵심 코드 테스트 우선
2. **에러 경로 테스트**: 정상 경로만 테스트하지 않기
3. **엣지 케이스 테스트**: 경계 조건, null 값 등
4. **커버리지로 공백 찾기**: 커버리지 보고서로 테스트할 것 파악
5. **의미 있는 테스트 작성**: 커버리지를 위해 테스트하지 않기
6. **테스트 불가능한 코드 제외**: pragma 또는 설정 사용
7. **추세 추적**: 시간에 따른 커버리지 모니터링

### 하지 말아야 할 것:

1. **100%에 집착**: 80%+면 훌륭, 품질에 집중
2. **사소한 코드 테스트**: Getter/setter는 가치가 적음
3. **프레임워크 코드 테스트**: Pydantic, FastAPI 등을 테스트하지 않기
4. **하나의 거대한 테스트 작성**: 테스트는 집중적으로
5. **실패하는 테스트 무시**: 깨진 테스트는 수정하거나 제거
6. **구현 테스트**: 방법이 아닌 동작 테스트

---

## 문제 해결

### 커버리지가 늘어나지 않는 경우

```bash
# 1. 테스트가 실제로 실행되는지 확인
pytest -v tests/unit/domain/artist/test_artist_service.py

# 2. 파일이 커버리지 범위 내에 있는지 확인
pytest --cov=backend.domain.artist tests/unit/domain/artist/

# 3. 파일이 제외되지 않았는지 확인
# pyproject.toml [tool.coverage.run] omit 설정 확인
```

### 커버리지가 예상보다 낮은 경우

```bash
# 1. 상세 보고서 생성
pytest --cov=backend --cov-report=html

# 2. HTML 보고서 열어 커버되지 않은 라인 찾기
open htmlcov/index.html

# 3. 빨간색으로 표시된 "Missing" 라인 확인
# 4. 해당 라인을 위한 테스트 작성
```

### CI에서 커버리지 실패

```bash
# 1. 로컬에서 동일한 명령어로 실행
pytest --cov=backend --cov-fail-under=80

# 2. CI에서 파일이 제외되었는지 확인
# CI 환경에 pyproject.toml이 있는지 확인

# 3. Python 버전이 일치하는지 확인
# CI는 개발과 동일한 Python 버전을 사용해야 함
```

---

## 체크리스트

좋은 커버리지 유지를 위해:

- [ ] 정기적으로 커버리지 보고서 실행
- [ ] 전체 80%+ 커버리지 목표
- [ ] 비즈니스 로직과 에러 처리 우선
- [ ] 모든 코드 브랜치 테스트 (if/else)
- [ ] HTML 보고서로 공백 찾기
- [ ] 테스트 불가능한 코드 적절히 제외
- [ ] 커버리지 수치보다 의미 있는 테스트에 집중
- [ ] 시간에 따른 커버리지 추세 추적
- [ ] CI/CD에 커버리지 확인 통합
- [ ] PR에서 커버리지 보고서 검토

---

## 관련 리소스

- [testing-architecture.md](testing-architecture.md) - 테스트할 것
- [unit-testing.md](unit-testing.md) - 효과적인 unit 테스트 작성
- [integration-testing.md](integration-testing.md) - 더 높은 수준의 커버리지
