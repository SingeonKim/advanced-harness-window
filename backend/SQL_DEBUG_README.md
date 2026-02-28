# SQL 쿼리 디버깅

이 백엔드는 디버깅 목적의 SQL 쿼리 로깅을 지원합니다. 활성화 시 SQLAlchemy가 실행하는 모든 SQL 쿼리가 파라미터와 함께 콘솔에 출력됩니다.

## SQL 쿼리 로깅 활성화 방법

### 방법 1: 환경 변수 사용

서버 시작 전 `SQL_DEBUG` 환경 변수를 `true`로 설정합니다:

```bash
export SQL_DEBUG=true
uvicorn backend.main:app --reload
```

### 방법 2: 디버그 스크립트 사용

제공된 디버그 스크립트를 실행합니다:

```bash
cd backend
./debug_sql.sh
```

### 방법 3: Docker 또는 Docker Compose에서

Docker 설정에 환경 변수를 추가합니다:

```yaml
environment:
  - SQL_DEBUG=true
```

## 로깅되는 내용

SQL 디버깅이 활성화되면 다음을 확인할 수 있습니다:

1. **원시 SQL 쿼리** - 실제로 실행되는 SQL 구문
2. **쿼리 파라미터** - 파라미터화된 쿼리에 전달되는 값
3. **연결 풀 활동** - 연결 체크아웃/체크인 이벤트
4. **트랜잭션 경계** - BEGIN, COMMIT, ROLLBACK 구문

## 출력 예시

```
INFO:sqlalchemy.engine.Engine:SELECT product.id, product.title, product.price
FROM product
WHERE product.site_id = %(site_id_1)s
LIMIT %(param_1)s
INFO:sqlalchemy.engine.Engine:[generated in 0.00034s] {'site_id_1': UUID('8b497d88-decd-46b0-8674-16de3f3674c4'), 'param_1': 20}
```

## 성능 참고사항

⚠️ **경고**: SQL 쿼리 로깅은 성능에 상당한 영향을 미치고 대량의 출력을 생성합니다. 개발/디버깅 환경에서만 사용하고, 프로덕션에서는 절대 사용하지 마세요.

## SQL 쿼리 로깅 비활성화

SQL 쿼리 로깅을 비활성화하려면:
- 환경 변수 해제: `unset SQL_DEBUG`
- false로 설정: `export SQL_DEBUG=false`
- 설정하지 않기 (기본값은 비활성화)

## 문제 해결

SQL 쿼리가 보이지 않는 경우:
1. 환경 변수가 올바르게 설정되어 있는지 확인
2. 올바른 로그 출력(stdout/stderr)을 보고 있는지 확인
3. 백엔드가 실제로 데이터베이스 쿼리를 실행하는지 확인
4. 환경 변수 설정 후 서버를 재시작했는지 확인
