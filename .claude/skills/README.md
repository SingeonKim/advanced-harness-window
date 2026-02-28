# 스킬 (Skills)

컨텍스트 기반으로 자동 활성화되는 Claude Code용 프로덕션 검증 스킬입니다.

---

## 스킬이란?

스킬은 Claude가 필요할 때 로드하는 모듈형 지식 베이스입니다. 다음을 제공합니다:
- 도메인 특화 가이드라인
- 모범 사례
- 코드 예시
- 피해야 할 안티패턴

**문제:** 스킬은 기본적으로 자동으로 활성화되지 않습니다.

**해결:** 이 쇼케이스에는 스킬을 활성화하는 훅 + 설정이 포함되어 있습니다.

---

## 사용 가능한 스킬

### skill-developer (메타 스킬)
**목적:** Claude Code 스킬 생성 및 관리

**파일:** 7개 리소스 파일 (총 426줄)

**사용 시점:**
- 새 스킬 생성 시
- 스킬 구조 이해 시
- skill-rules.json 작업 시
- 스킬 활성화 디버깅 시

**커스터마이징:** ✅ 없음 - 그대로 복사

**[스킬 보기 →](skill-developer/)**

---

### backend-dev-guidelines
**목적:** Node.js/Express/TypeScript 개발 패턴

**파일:** 12개 리소스 파일 (메인 + 리소스 304줄)

**내용:**
- 계층형 아키텍처 (Routes → Controllers → Services → Repositories)
- BaseController 패턴
- Prisma 데이터베이스 접근
- Sentry 에러 추적
- Zod 유효성 검사
- UnifiedConfig 패턴
- 의존성 주입
- 테스팅 전략

**사용 시점:**
- API 라우트 생성/수정 시
- 컨트롤러 또는 서비스 빌드 시
- Prisma 데이터베이스 작업 시
- 에러 추적 설정 시

**커스터마이징:** ⚠️ skill-rules.json의 `pathPatterns`를 백엔드 디렉토리에 맞게 업데이트

**pathPatterns 예시:**
```json
{
  "pathPatterns": [
    "src/api/**/*.ts",       // src/api가 있는 단일 앱
    "backend/**/*.ts",       // 백엔드 디렉토리
    "services/*/src/**/*.ts" // 멀티 서비스 monorepo
  ]
}
```

**[스킬 보기 →](backend-dev-guidelines/)**

---

### frontend-dev-guidelines
**목적:** React/TypeScript/MUI v7 개발 패턴

**파일:** 11개 리소스 파일 (메인 + 리소스 398줄)

**내용:**
- 모던 React 패턴 (Suspense, lazy loading)
- 데이터 가져오기를 위한 useSuspenseQuery
- MUI v7 스타일링 (`size={{}}` prop을 사용하는 Grid)
- TanStack Router
- 파일 구성 (features/ 패턴)
- 성능 최적화
- TypeScript 모범 사례

**사용 시점:**
- React 컴포넌트 생성 시
- TanStack Query로 데이터 가져올 때
- MUI v7로 스타일링 시
- 라우팅 설정 시

**커스터마이징:** ⚠️ `pathPatterns` 업데이트 + React/MUI 사용 여부 확인

**pathPatterns 예시:**
```json
{
  "pathPatterns": [
    "src/**/*.tsx",          // 단일 React 앱
    "frontend/src/**/*.tsx", // 프론트엔드 디렉토리
    "apps/web/**/*.tsx"      // Monorepo 웹 앱
  ]
}
```

**참고:** 이 스킬은 MUI v6→v7 비호환성을 방지하기 위한 **가드레일** (enforcement: "block")로 설정되어 있습니다.

**[스킬 보기 →](frontend-dev-guidelines/)**

---

### route-tester
**목적:** JWT 쿠키 인증으로 인증된 API 라우트 테스팅

**파일:** 1개 메인 파일 (389줄)

**내용:**
- JWT 쿠키 기반 인증 테스팅
- test-auth-route.js 스크립트 패턴
- 쿠키 인증이 있는 cURL
- 인증 문제 디버깅
- POST/PUT/DELETE 작업 테스팅

**사용 시점:**
- API 엔드포인트 테스팅 시
- 인증 디버깅 시
- 라우트 기능 검증 시

**커스터마이징:** ⚠️ JWT 쿠키 인증 설정 필요

**먼저 질문:** "JWT 쿠키 기반 인증을 사용하시나요?"
- 예: 복사하고 서비스 URL 커스터마이징
- 아니오: 건너뛰거나 인증 방법에 맞게 적용

**[스킬 보기 →](route-tester/)**

---

### error-tracking
**목적:** Sentry 에러 추적 및 모니터링 패턴

**파일:** 1개 메인 파일 (약 250줄)

**내용:**
- Sentry v8 초기화
- 에러 캡처 패턴
- 브레드크럼과 사용자 컨텍스트
- 성능 모니터링
- Express 및 React와의 통합

**사용 시점:**
- 에러 추적 설정 시
- 예외 캡처 시
- 에러 컨텍스트 추가 시
- 프로덕션 문제 디버깅 시

**커스터마이징:** ⚠️ 백엔드에 맞게 `pathPatterns` 업데이트

**[스킬 보기 →](error-tracking/)**

---

## 프로젝트에 스킬 추가하는 방법

### 빠른 통합

**Claude Code를 위한 안내:**
```
사용자: "내 프로젝트에 backend-dev-guidelines 스킬을 추가해줘"

Claude가 해야 할 일:
1. 프로젝트 구조에 대해 질문
2. 스킬 디렉토리 복사
3. 사용자 경로로 skill-rules.json 업데이트
4. 통합 검증
```

전체 지침은 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md) 참고.

### 수동 통합

**Step 1: 스킬 디렉토리 복사**
```bash
cp -r claude-code-infrastructure-showcase/.claude/skills/backend-dev-guidelines \
      your-project/.claude/skills/
```

**Step 2: skill-rules.json 업데이트**

없다면 생성:
```bash
cp claude-code-infrastructure-showcase/.claude/skills/skill-rules.json \
   your-project/.claude/skills/
```

그런 다음 프로젝트에 맞게 `pathPatterns` 커스터마이징:
```json
{
  "skills": {
    "backend-dev-guidelines": {
      "fileTriggers": {
        "pathPatterns": [
          "YOUR_BACKEND_PATH/**/*.ts"  // ← 이것을 업데이트!
        ]
      }
    }
  }
}
```

**Step 3: 테스트**
- 백엔드 디렉토리의 파일 편집
- 스킬이 자동으로 활성화되어야 함

---

## skill-rules.json 설정

### 기능

다음을 기반으로 스킬 활성화 시점을 정의합니다:
- 사용자 프롬프트의 **키워드** ("backend", "API", "route")
- **의도 패턴** (사용자 의도와 정규식 매칭)
- **파일 경로 패턴** (백엔드 파일 편집)
- **내용 패턴** (코드에 Prisma 쿼리 포함)

### 설정 형식

```json
{
  "skill-name": {
    "type": "domain" | "guardrail",
    "enforcement": "suggest" | "block",
    "priority": "high" | "medium" | "low",
    "promptTriggers": {
      "keywords": ["키워드", "목록"],
      "intentPatterns": ["정규식 패턴"]
    },
    "fileTriggers": {
      "pathPatterns": ["path/to/files/**/*.ts"],
      "contentPatterns": ["import.*Prisma"]
    }
  }
}
```

### 실행 레벨

- **suggest**: 스킬이 제안으로 표시, 차단하지 않음
- **block**: 진행하기 전에 스킬을 사용해야 함 (가드레일)

**"block" 사용 시:**
- 파괴적인 변경 방지 (MUI v6→v7)
- 중요한 데이터베이스 작업
- 보안 민감 코드

**"suggest" 사용 시:**
- 일반적인 모범 사례
- 도메인 가이드
- 코드 구성

---

## 자체 스킬 만들기

**skill-developer** 스킬에서 다음에 대한 완전한 가이드 참고:
- 스킬 YAML 프론트매터 구조
- 리소스 파일 구성
- 트리거 패턴 설계
- 스킬 활성화 테스팅

**빠른 템플릿:**
```markdown
---
name: my-skill
description: 이 스킬이 하는 일
---

# 스킬 제목

## 목적
[이 스킬이 존재하는 이유]

## 이 스킬을 사용하는 경우
[자동 활성화 시나리오]

## 빠른 참조
[주요 패턴과 예시]

## 리소스 파일
- [topic-1.md](resources/topic-1.md)
- [topic-2.md](resources/topic-2.md)
```

---

## 문제 해결

### 스킬이 활성화되지 않는 경우

**확인:**
1. `.claude/skills/`에 스킬 디렉토리가 있나요?
2. `skill-rules.json`에 스킬이 나열되어 있나요?
3. `pathPatterns`가 파일과 매칭되나요?
4. 훅이 설치되고 작동 중인가요?
5. settings.json이 올바르게 설정되어 있나요?

**디버그:**
```bash
# 스킬 존재 확인
ls -la .claude/skills/

# skill-rules.json 유효성 검사
cat .claude/skills/skill-rules.json | jq .

# 훅이 실행 가능한지 확인
ls -la .claude/hooks/*.sh

# 훅 수동 테스트
./.claude/hooks/skill-activation-prompt.sh
```

### 스킬이 너무 자주 활성화되는 경우

skill-rules.json 업데이트:
- 키워드를 더 구체적으로 만들기
- `pathPatterns` 범위 좁히기
- `intentPatterns` 특수성 높이기

### 스킬이 전혀 활성화되지 않는 경우

skill-rules.json 업데이트:
- 키워드 추가
- `pathPatterns` 범위 넓히기
- `intentPatterns` 추가

---

## Claude Code를 위한 안내

**사용자를 위해 스킬을 통합할 때:**

1. **먼저 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md) 읽기**
2. 프로젝트 구조에 대해 질문
3. skill-rules.json의 `pathPatterns` 커스터마이징
4. 스킬 파일에 하드코딩된 경로가 없는지 확인
5. 통합 후 활성화 테스트

**일반적인 실수:**
- 예시 경로 그대로 유지 (blog-api/, frontend/)
- monorepo vs 단일 앱 여부를 묻지 않음
- skill-rules.json을 커스터마이징 없이 복사

---

## 다음 단계

1. **간단하게 시작:** 업무에 맞는 스킬 하나 추가
2. **활성화 확인:** 관련 파일 편집, 스킬이 제안되어야 함
3. **더 추가:** 첫 스킬이 작동하면 다른 것도 추가
4. **커스터마이징:** 워크플로우에 맞게 트리거 조정

**질문이 있으신가요?** 포괄적인 통합 지침은 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md) 참고.
