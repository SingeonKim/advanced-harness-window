# Little-Boy 백엔드

Little-Boy 애플리케이션의 백엔드 API 서비스입니다.

## 설치

```bash
# 가상 환경 생성
uv venv

# 가상 환경 활성화
source .venv/bin/activate

# 의존성 설치
uv pip install -e .

# 개발 서버 시작
uvicorn app.main:app --reload --port 28900
```

## 환경 변수

`.env.example`을 `.env`로 복사하고 설정합니다:

```bash
cp .env.example .env
```

## 프로젝트 구조

```
backend/
├── app/               # 애플리케이션 코드
│   ├── api/          # API 엔드포인트
│   ├── core/         # 핵심 설정
│   ├── db/           # 데이터베이스 모델 및 레포지토리
│   └── services/     # 비즈니스 로직 서비스
├── alembic/          # 데이터베이스 마이그레이션
├── infrastructure/   # 코드로서의 인프라
├── scripts/          # 유틸리티 스크립트
└── tests/           # 테스트 스위트
```

## API 문서

실행 후 다음 주소에서 API 문서를 확인할 수 있습니다:

- Swagger UI: http://localhost:8000/docs
