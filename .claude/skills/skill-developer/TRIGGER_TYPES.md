# 트리거 유형 - 완전한 가이드

Claude Code의 스킬 자동 활성화 시스템에서 스킬 트리거 설정을 위한 완전한 참조 가이드.

## 목차

- [키워드 트리거 (명시적)](#키워드-트리거-명시적)
- [의도 패턴 트리거 (암묵적)](#의도-패턴-트리거-암묵적)
- [파일 경로 트리거](#파일-경로-트리거)
- [콘텐츠 패턴 트리거](#콘텐츠-패턴-트리거)
- [모범 사례 요약](#모범-사례-요약)

---

## 키워드 트리거 (명시적)

### 작동 방식

사용자 프롬프트에서 대소문자를 구분하지 않는 부분 문자열 매칭.

### 사용 시

사용자가 주제를 명시적으로 언급할 때 주제 기반 활성화.

### 설정

```json
"promptTriggers": {
  "keywords": ["layout", "grid", "toolbar", "submission"]
}
```

### 예시

- 사용자 프롬프트: "how does the **layout** system work?"
- 매칭: "layout" 키워드
- 활성화: `project-catalog-developer`

### 모범 사례

- 구체적이고 명확한 용어 사용
- 일반적인 변형 포함 ("layout", "layout system", "grid layout")
- 너무 일반적인 단어 지양 ("system", "work", "create")
- 실제 프롬프트로 테스트

---

## 의도 패턴 트리거 (암묵적)

### 작동 방식

사용자가 주제를 명시적으로 언급하지 않더라도 의도를 감지하는 Regex 패턴 매칭.

### 사용 시

사용자가 특정 주제가 아닌 하고 싶은 것을 설명할 때 작업 기반 활성화.

### 설정

```json
"promptTriggers": {
  "intentPatterns": [
    "(create|add|implement).*?(feature|endpoint)",
    "(how does|explain).*?(layout|workflow)"
  ]
}
```

### 예시

**데이터베이스 작업:**
- 사용자 프롬프트: "add user tracking feature"
- 매칭: `(add).*?(feature)`
- 활성화: `database-verification`, `error-tracking`

**컴포넌트 생성:**
- 사용자 프롬프트: "create a dashboard widget"
- 매칭: `(create).*?(component)` (패턴에 component가 있는 경우)
- 활성화: `frontend-dev-guidelines`

### 모범 사례

- 일반적인 동작 동사 캡처: `(create|add|modify|build|implement)`
- 도메인 특화 명사 포함: `(feature|endpoint|component|workflow)`
- 비탐욕적 매칭 사용: `.*?` 대신 `.*`
- regex tester로 철저히 패턴 테스트 (https://regex101.com/)
- 패턴을 너무 광범위하게 만들지 말 것 (오탐 원인)
- 패턴을 너무 구체적으로 만들지 말 것 (미탐 원인)

### 일반적인 패턴 예시

```regex
# 데이터베이스 작업
(add|create|implement).*?(user|login|auth|feature)

# 설명
(how does|explain|what is|describe).*?

# 프론트엔드 작업
(create|add|make|build).*?(component|UI|page|modal|dialog)

# 오류 처리
(fix|handle|catch|debug).*?(error|exception|bug)

# 워크플로우 작업
(create|add|modify).*?(workflow|step|branch|condition)
```

---

## 파일 경로 트리거

### 작동 방식

편집 중인 파일 경로에 대한 Glob 패턴 매칭.

### 사용 시

프로젝트에서 파일 위치에 따른 도메인/영역별 활성화.

### 설정

```json
"fileTriggers": {
  "pathPatterns": [
    "frontend/src/**/*.tsx",
    "form/src/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### Glob 패턴 문법

- `**` = 임의 수의 디렉토리 (제로 포함)
- `*` = 디렉토리 이름 내 임의의 문자
- 예시:
  - `frontend/src/**/*.tsx` = frontend/src와 하위 디렉토리의 모든 .tsx 파일
  - `**/schema.prisma` = 프로젝트 어디서든 schema.prisma
  - `form/src/**/*.ts` = form/src 하위의 모든 .ts 파일

### 예시

- 편집 중인 파일: `frontend/src/components/Dashboard.tsx`
- 매칭: `frontend/src/**/*.tsx`
- 활성화: `frontend-dev-guidelines`

### 모범 사례

- 오탐 방지를 위해 구체적으로 작성
- 테스트 파일에 제외 사용: `**/*.test.ts`
- 하위 디렉토리 구조 고려
- 실제 파일 경로로 패턴 테스트
- 가능한 좁은 패턴 사용: `form/src/services/**` 대신 `form/**`

### 일반적인 경로 패턴

```glob
# 프론트엔드
frontend/src/**/*.tsx        # 모든 React 컴포넌트
frontend/src/**/*.ts         # 모든 TypeScript 파일
frontend/src/components/**   # 컴포넌트 디렉토리만

# 백엔드 서비스
form/src/**/*.ts            # Form 서비스
email/src/**/*.ts           # Email 서비스
users/src/**/*.ts           # Users 서비스

# 데이터베이스
**/schema.prisma            # Prisma 스키마 (어디서든)
**/migrations/**/*.sql      # 마이그레이션 파일
database/src/**/*.ts        # 데이터베이스 스크립트

# 워크플로우
form/src/workflow/**/*.ts              # 워크플로우 엔진
form/src/workflow-definitions/**/*.json # 워크플로우 정의

# 테스트 제외
**/*.test.ts                # TypeScript 테스트
**/*.test.tsx               # React 컴포넌트 테스트
**/*.spec.ts                # Spec 파일
```

---

## 콘텐츠 패턴 트리거

### 작동 방식

파일의 실제 내용(파일 안에 있는 것)에 대한 Regex 패턴 매칭.

### 사용 시

코드가 가져오거나 사용하는 것(Prisma, 컨트롤러, 특정 라이브러리)에 기반한 기술 특화 활성화.

### 설정

```json
"fileTriggers": {
  "contentPatterns": [
    "import.*[Pp]risma",
    "PrismaService",
    "\\.findMany\\(",
    "\\.create\\("
  ]
}
```

### 예시

**Prisma 감지:**
- 파일 포함: `import { PrismaService } from '@project/database'`
- 매칭: `import.*[Pp]risma`
- 활성화: `database-verification`

**컨트롤러 감지:**
- 파일 포함: `export class UserController {`
- 매칭: `export class.*Controller`
- 활성화: `error-tracking`

### 모범 사례

- 임포트 매칭: `import.*[Pp]risma` (대소문자 구분 없이 [Pp]로)
- 특수 regex 문자 이스케이프: `\\.findMany\\(` (`.findMany(` 아님)
- 패턴은 대소문자 구분 없는 플래그 사용
- 실제 파일 내용으로 테스트
- 잘못된 매칭을 피하기 위해 충분히 구체적인 패턴 작성

### 일반적인 콘텐츠 패턴

```regex
# Prisma/데이터베이스
import.*[Pp]risma                # Prisma 임포트
PrismaService                    # PrismaService 사용
prisma\.                         # prisma.something
\.findMany\(                     # Prisma 쿼리 메서드
\.create\(
\.update\(
\.delete\(

# 컨트롤러/라우트
export class.*Controller         # 컨트롤러 클래스
router\.                         # Express 라우터
app\.(get|post|put|delete|patch) # Express 앱 라우트

# 오류 처리
try\s*\{                        # Try 블록
catch\s*\(                      # Catch 블록
throw new                        # Throw 문

# React/컴포넌트
export.*React\.FC               # React 함수형 컴포넌트
export default function.*       # 기본 함수 익스포트
useState|useEffect              # React 훅
```

---

## 모범 사례 요약

### 해야 할 것:
- 구체적이고 명확한 키워드 사용
- 실제 예시로 모든 패턴 테스트
- 일반적인 변형 포함
- 비탐욕적 regex 사용: `.*?`
- 콘텐츠 패턴에서 특수 문자 이스케이프
- 테스트 파일에 제외 추가
- 파일 경로 패턴을 좁고 구체적으로 작성

### 하지 말아야 할 것:
- 너무 일반적인 키워드 사용 ("system", "work")
- 의도 패턴을 너무 광범위하게 만들기 (오탐)
- 패턴을 너무 구체적으로 만들기 (미탐)
- regex tester 테스트 잊기 (https://regex101.com/)
- 탐욕적 regex 사용: `.*?` 대신 `.*`
- 파일 경로에서 너무 광범위하게 매칭

### 트리거 테스트

**키워드/의도 트리거 테스트:**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**파일 경로/콘텐츠 트리거 테스트:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/test/file.ts"}
}
EOF
```

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 스킬 가이드
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 완전한 skill-rules.json 스키마
- [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md) - 바로 사용 가능한 패턴 라이브러리
