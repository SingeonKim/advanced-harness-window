# CLAUDE.md

이 파일은 이 저장소에서 코드 작업을 할 때 Claude Code(claude.ai/code)에게 안내를 제공합니다.

## Monorepo 구조

이것은 백엔드(FastAPI)와 프론트엔드(Next.js) 애플리케이션을 모두 포함하는 **monorepo**입니다:

- `backend/` - PostgreSQL을 사용하는 Python FastAPI 백엔드
- `frontend/` - TypeScript와 Tailwind CSS를 사용하는 Next.js 15 프론트엔드

## 백엔드 개발

### 사전 요구사항

- Python 3.12.3 (정확한 버전, `backend/pyproject.toml` 참고)
- Docker & Docker Compose

### 설치

```bash
cd backend
uv venv
source .venv/bin/activate
uv pip install -e .
uv pip install -e .[dev]  # 개발 의존성 설치 (black, isort, mypy, ruff)
```

### 백엔드 실행

```bash
# 개발 서버 (참고: 모듈은 app.main이 아닌 backend.main)
cd backend
uvicorn backend.main:app --reload --port 28080

# Docker Compose (프로덕션 유사 환경)
cd backend
docker-compose up
```

### 코드 품질

```bash
cd backend
black .                           # 코드 포맷팅
isort . --profile black          # import 정렬
ruff check --fix .               # 자동 수정과 함께 린팅
mypy .                           # 타입 체크
pre-commit run --all-files       # 모든 pre-commit 훅 실행
```

### 백엔드 아키텍처

**프레임워크:** SQLModel + SQLAlchemy를 사용하는 async/await 패턴의 FastAPI

**데이터베이스 레이어:**

- **읽기/쓰기 분리**: 읽기와 쓰기 작업에 별도의 데이터베이스 연결 사용
- `backend/db/orm.py`에서 이벤트 루프 기반의 엔진 및 sessionmaker 캐싱
- 세션 팩토리:
  - `get_write_session()` / `get_write_session_dependency()` - 쓰기 작업용
  - `get_read_session()` / `get_read_session_dependency()` - 읽기 작업용
- asyncpg 드라이버를 사용하는 PostgreSQL, 연결에 SSL 필수

**도메인 주도 설계(DDD):**

- `backend/domain/{entity}/` 디렉토리에 비즈니스 로직 구성
- 각 도메인 포함 요소: `model.py` (SQLModel), `service.py` (비즈니스 로직), `repository.py` (데이터 접근)
- 도메인: `user`, `auth`, `artist`, `artwork`, `admin`, `curai`, `exhibition`, `message`, `notification`, `subscription`, `shared`

**API 구조:**

- `/api/v1/` 접두사 하의 버전별 엔드포인트
- `backend/api/v1/routers/`의 라우터: `auth.py`, `artist.py`, `artwork.py`, `admin.py`, `curai.py`, `exhibition.py`, `message.py`, `notification.py`, `health.py`
- 요청/응답 유효성 검사를 위한 `backend/dtos/`의 DTO
- `backend/main.py`에서 `create_application()`을 통한 메인 앱 생성

**설정:**

- Pydantic BaseSettings를 사용하는 `backend/core/config.py`의 설정
- 필수 환경 변수: 데이터베이스 자격증명 (읽기/쓰기), JWT 설정
- 로컬 개발 및 프로덕션 도메인(qwarty.net)을 위한 CORS 설정

**배포:**

- Docker 이미지: `206404754787.dkr.ecr.ap-northeast-2.amazonaws.com/qwarty-backend:latest`
- AWS ECS 클러스터 `qwarty-backend-cluster`에 배포
- `main` 브랜치에 push 시 GitHub Actions를 통한 자동 배포

## 프론트엔드 개발

### 사전 요구사항

- Node.js 20+
- pnpm 패키지 매니저

### 설치

```bash
cd frontend
pnpm install
```

### 프론트엔드 실행

```bash
cd frontend
pnpm dev          # Turbopack을 사용한 개발 서버 (http://localhost:3000)
pnpm build        # Turbopack을 사용한 프로덕션 빌드
pnpm start        # 프로덕션 서버 시작
pnpm lint         # ESLint 실행
```

### 프론트엔드 아키텍처

**프레임워크:** React 19, TypeScript, Tailwind CSS 4를 사용하는 Next.js 15 (App Router)

**프로젝트 구조:**

- `src/app/` - Next.js App Router 페이지 및 API 라우트
  - 라우트 그룹: `/login`, `/sign-up`, `/artists`, `/artist`, `/account`, `/admin`, `/agent`, `/explore`, `/messages`, `/search`
  - API 라우트: `/api/upload` (S3 파일 업로드)
- `src/components/` - 기능별로 구성된 재사용 가능한 React 컴포넌트
- `src/lib/` - 핵심 유틸리티 및 설정
  - `api.ts` - 백엔드 통신을 위한 API 클라이언트
  - `serverAuth.ts` - 서버 사이드 인증 유틸리티
  - `s3Upload.ts` - 압축 기능이 포함된 AWS S3 업로드 유틸리티
  - `emailAuth.ts` - 이메일 인증 유틸리티
  - `firebase.ts` - Firebase 설정
  - `theme.ts` - MUI 테마 설정
- `src/hooks/` - 커스텀 React 훅
- `src/providers/` - React context 프로바이더
- `src/utils/` - 유틸리티 함수
- `src/types/` - TypeScript 타입 정의
- `src/const/` - 애플리케이션 상수
- `src/interfaces/` - TypeScript 인터페이스
- `src/locales/` - next-intl을 사용한 국제화(i18n)

**주요 기술:**

- **스타일링:** Tailwind CSS 4, MUI Material (컴포넌트), Emotion (CSS-in-JS)
- **상태 관리:** React 훅 및 context 프로바이더
- **인증:** 서버 사이드 검증을 사용한 JWT 토큰
- **파일 업로드:** 클라이언트 사이드 압축이 포함된 AWS S3
- **국제화:** next-intl을 사용한 i18n 지원
- **UI 컴포넌트:** MUI Material, Lucide React 아이콘

**설정:**

- `next.config.ts` - 원격 이미지 패턴(AWS S3) 및 Turbopack이 활성화된 Next.js 설정
- `tailwind.config.ts` - Tailwind CSS 4 설정
- `eslint.config.mjs` - TypeScript 지원이 포함된 ESLint 설정
- 필수 환경 변수: API 엔드포인트, Firebase 설정, AWS 자격증명, Kakao OAuth

## 개발 워크플로우

### 환경 파일

백엔드와 프론트엔드 모두 `.env` 파일이 필요합니다:

- `backend/.env` - 데이터베이스 자격증명 (읽기/쓰기), JWT 설정
- `frontend/.env` - API 엔드포인트, Firebase, AWS S3, Kakao OAuth 자격증명

### Git 워크플로우

- 메인 브랜치: `main` (보호됨, AWS ECS에 백엔드 자동 배포)
- 개발을 위한 feature 브랜치 생성
- 백엔드 배포는 `.github/workflows/deploy-real.yaml`을 통해 `main` push 시 자동 트리거
  - Docker 이미지 빌드 후 ECR에 push: `206404754787.dkr.ecr.ap-northeast-2.amazonaws.com/qwarty-backend:latest`
  - ECS 서비스 배포: `qwarty-backend-cluster` 클러스터의 `prod-apne2-qwarty-backend-svc`
  - 태스크 정의: `backend/prod-apne2-qwarty-backend-task-def.json`

### 주요 디자인 패턴

**백엔드:**

- 명확한 관심사 분리를 갖춘 도메인 주도 설계(DDD)
- 데이터 접근을 위한 Repository 패턴
- API 계약을 위한 DTO 패턴
- FastAPI의 `Depends()`를 통한 의존성 주입
- 적절한 세션 관리와 함께 전체적인 async/await 사용

**프론트엔드:**

- App Router를 사용한 서버 사이드 렌더링(SSR)
- 클라이언트/서버 컴포넌트 분리
- API 호출을 위한 Server Actions
- S3 업로드를 통한 이미지 최적화
- Tailwind CSS를 사용한 반응형 디자인

### 중요 참고사항

- **백엔드 모듈 경로:** uvicorn 실행 시 `app.main:app`이 아닌 `backend.main:app` 사용
- **데이터베이스 세션:** 항상 `backend/db/orm.py`의 적절한 읽기/쓰기 세션 팩토리 사용
  - 데이터를 수정하는 FastAPI 엔드포인트에는 `get_write_session_dependency()` 사용
  - 데이터만 읽는 FastAPI 엔드포인트에는 `get_read_session_dependency()` 사용
- **프론트엔드 API 호출:** `src/lib/api.ts`에 집중화
- **이미지 업로드:** S3 업로드는 클라이언트 직접 업로드를 위한 presigned POST URL 사용
  - 플로우: 클라이언트 → 백엔드 (`POST /api/v1/upload/presigned-url`) → 백엔드가 presigned POST 생성 → 클라이언트가 S3에 직접 업로드
  - 백엔드: `backend/utils/s3.py` - `generate_presigned_post()`가 필드 및 조건이 포함된 presigned POST 생성 (최대 50MB)
  - 프론트엔드: `src/lib/s3Upload.ts` - 압축(browser-image-compression), presigned URL 요청, S3 업로드 처리
  - 이미지는 업로드 전 클라이언트 사이드에서 WebP 포맷으로 압축됨 (사용하는 함수에 따라 선택적)
  - 썸네일 생성 지원: 원본과 압축된 썸네일을 병렬로 업로드
- **인증:** JWT 기반, `src/lib/serverAuth.ts`에서 서버 사이드 검증
- **Pre-commit 훅:** 백엔드는 `.pre-commit-config.yaml`을 통해 black, isort, ruff 및 기타 검사 사용

## 테스팅 & 성능 모니터링

### Chrome DevTools MCP를 활용한 브라우저 테스팅

**중요:** 프론트엔드 브라우저 테스팅 및 성능 측정에는 반드시 **chrome-devtools MCP**를 사용하세요. 테스팅을 위해 `pnpm dev`로 개발 서버를 수동으로 시작하지 마세요.

**사용 가능한 MCP 도구:**

- `mcp__chrome-devtools__navigate_page` - URL로 이동
- `mcp__chrome-devtools__take_snapshot` - 페이지 스냅샷 촬영 (구조)
- `mcp__chrome-devtools__take_screenshot` - 스크린샷 촬영
- `mcp__chrome-devtools__click` - 요소 클릭
- `mcp__chrome-devtools__fill` - 폼 입력 채우기
- `mcp__chrome-devtools__list_console_messages` - 콘솔 에러 확인
- `mcp__chrome-devtools__list_network_requests` - API 호출 모니터링
- `mcp__chrome-devtools__performance_start_trace` - 성능 기록 시작
- `mcp__chrome-devtools__performance_stop_trace` - 성능 기록 중지 및 분석

### 성능 테스팅 워크플로우

**Step 1: 백그라운드에서 개발 서버 시작**
chrome-devtools MCP가 이미 실행 중이라면 먼저 종료하세요.
chrome-devtools MCP를 사용하여 백그라운드에서 개발 서버를 시작하세요.

**Step 2: chrome-devtools MCP로 브라우저 테스트 실행**

```typescript
// 예시 테스트 플로우:
1. 페이지 이동: mcp__chrome-devtools__navigate_page({ url: "http://localhost:3000/ko" })
2. 스냅샷 촬영: mcp__chrome-devtools__take_snapshot({ verbose: false })
3. 콘솔 확인: mcp__chrome-devtools__list_console_messages()
4. 성능 추적 시작: mcp__chrome-devtools__performance_start_trace({ reload: true, autoStop: true })
5. 결과 분석: LCP, FCP, TTI, CLS 지표 검토
6. 스크린샷 촬영: mcp__chrome-devtools__take_screenshot({ fullPage: true })
```

**Step 3: Core Web Vitals 측정**

- **LCP (Largest Contentful Paint):** 목표 <2000ms
- **FCP (First Contentful Paint):** 목표 <1000ms
- **CLS (Cumulative Layout Shift):** 목표 <0.1
- **TTI (Time to Interactive):** 목표 <2500ms
- **TBT (Total Blocking Time):** 목표 <300ms

### 테스트 문서

**테스트 계획 및 보고서:**

- `frontend/tests/browser/` - 브라우저 테스트 문서
- `frontend/tests/browser/test-reports/` - 테스트 실행 보고서
- `frontend/docs/performance-baseline.md` - 성능 기준선 지표
- `frontend/docs/PERFORMANCE-DASHBOARD.md` - 성능 대시보드

**테스트 커버리지:**

- 홈 페이지 테스트 (8개)
- 아티스트 페이지 테스트 (10개)
- SearchBar 컴포넌트 테스트 (20개)
- 합계: 38개 자동화 테스트 케이스

### Lighthouse CI (자동화된 성능 회귀 테스팅)

**설정:** 프론트엔드 디렉토리의 `.lighthouserc.js`
**GitHub Actions:** `.github/workflows/lighthouse-ci.yaml`

**수동 Lighthouse CI 실행:**

```bash
cd frontend
pnpm build
npx lhci autorun
```

### 예시: 홈 페이지 테스팅

```bash
# 1. 백엔드가 실행 중인지 확인
cd backend
uvicorn backend.main:app --reload --port 28080

# 2. 프론트엔드 개발 서버 시작
cd frontend
pnpm dev

# 3. chrome-devtools MCP 도구를 사용하여:
# - http://localhost:3000/ko로 이동
# - 구조 확인을 위한 스냅샷 촬영
# - 콘솔 메시지 확인 (오류 0개 예상)
# - 리로드와 함께 성능 추적 시작
# - Core Web Vitals 검토
# - 문서화를 위한 스크린샷 촬영
# - 기준선과 비교 (frontend/docs/performance-baseline.md)
```

## AI 에이전트 시스템 (Curai)

**프레임워크:** 스트리밍 SSE 응답을 사용하는 Pydantic AI

**아키텍처:**

- 메시지 이력 영속성을 갖춘 스레드 기반 대화
- 사전 정의된 DB 도구를 사용한 에이전트 검색
