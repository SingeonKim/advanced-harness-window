# Claude 통합 가이드

**CLAUDE CODE에게:** 사용자가 이 쇼케이스 저장소의 컴포넌트를 자신의 프로젝트에 통합해달라고 요청할 때, 다음 지침을 주의 깊게 따르세요.

---

## 개요

이 저장소는 Claude Code 인프라 컴포넌트의 **참고 라이브러리**입니다. 사용자는 특정 부분을 자신의 프로젝트에 통합하는 데 도움을 요청할 것입니다. 당신의 역할은:

1. **명확화 질문하기** - 프로젝트 구조에 대해 질문
2. **적절한 파일 복사**
3. **설정 커스터마이징** - 사용자 환경에 맞게
4. **통합 검증** - 올바르게 작동하는지 확인

**핵심 원칙:** 프로젝트 구조를 가정하기 전에 항상 먼저 물어보세요. 한 프로젝트에서 작동하는 것이 다른 프로젝트에서는 작동하지 않을 수 있습니다.

---

## 기술 스택 호환성 확인

**중요:** 스킬을 통합하기 전에, 사용자의 기술 스택이 스킬 요구사항과 일치하는지 확인하세요.

### 프론트엔드 스킬

**frontend-dev-guidelines 요구사항:**
- React (19+)
- MUI v5
- TypeScript

**통합 전 질문:**
"React와 MUI v5를 사용하시나요?"

**아니오인 경우:**
```
frontend-dev-guidelines 스킬은 React + MUI v7 전용으로 설계되었습니다. 다음 중 선택하세요:
1. 이것을 템플릿으로 사용하여 [사용자 스택]에 맞는 유사한 스킬 생성
2. 프레임워크 독립적인 패턴 추출 (파일 구성, 성능 등)
3. 관련이 없다면 이 스킬 건너뛰기

어떤 방법을 선호하시나요?
```

### 백엔드 스킬

**backend-dev-guidelines 요구사항:**
- Node.js/Express
- TypeScript
- Prisma ORM
- Sentry

**통합 전 질문:**
"Node.js와 Express, Prisma를 사용하시나요?"

**아니오인 경우:**
```
backend-dev-guidelines 스킬은 Express/Prisma용으로 설계되었습니다. 다음 중 선택하세요:
1. 이것을 템플릿으로 사용하여 [사용자 스택]에 맞는 유사한 가이드라인 생성
2. 아키텍처 패턴 추출 (계층형 아키텍처는 모든 프레임워크에 적용 가능)
3. 이 스킬 건너뛰기

어떤 방법을 선호하시나요?
```

### 기술 독립적인 스킬

다음 스킬들은 **어떤 기술 스택에서도** 작동합니다:
- ✅ **skill-developer** - 메타 스킬, 기술 요구사항 없음
- ✅ **route-tester** - JWT 쿠키 인증만 필요 (프레임워크 독립적)
- ✅ **error-tracking** - Sentry는 대부분의 스택과 호환

---

## 일반 통합 패턴

사용자가 **"[컴포넌트]를 내 프로젝트에 추가해줘"** 라고 할 때:

1. 컴포넌트 유형 파악 (스킬/훅/에이전트/명령어)
2. **기술 스택 호환성 확인** (프론트엔드/백엔드 스킬의 경우)
3. 프로젝트 구조 파악
4. 파일 복사 또는 스택에 맞게 적용
5. 사용자 환경에 맞게 커스터마이징
6. 통합 검증
7. 다음 단계 안내

---

## 스킬 통합

### 단계별 프로세스

**사용자가 스킬을 요청할 때** (예: "backend-dev-guidelines 추가해줘"):

#### 1. 프로젝트 파악

**다음 질문들을 하세요:**
- "프로젝트 구조가 어떻게 되나요? 단일 앱, monorepo, 또는 멀티 서비스?"
- "[백엔드/프론트엔드] 코드는 어디에 있나요?"
- "어떤 프레임워크/기술을 사용하시나요?"

#### 2. 스킬 복사

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. skill-rules.json 처리

**존재 여부 확인:**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**없는 경우:**
- 쇼케이스에서 템플릿 복사
- 사용자가 원하지 않는 스킬 제거
- 프로젝트에 맞게 커스터마이징

**있는 경우:**
- 현재 skill-rules.json 읽기
- 새 스킬 항목 추가
- 기존 스킬이 깨지지 않도록 주의하여 병합

#### 4. 경로 패턴 커스터마이징

**중요:** skill-rules.json의 `pathPatterns`를 사용자 구조에 맞게 업데이트:

**예시 - Monorepo 사용자:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "packages/api/src/**/*.ts",
        "packages/server/src/**/*.ts",
        "apps/backend/**/*.ts"
      ]
    }
  }
}
```

**예시 - 단일 백엔드 사용자:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.ts",
        "backend/**/*.ts"
      ]
    }
  }
}
```

**확실하지 않을 때 사용할 수 있는 일반적인 패턴:**
```json
{
  "pathPatterns": [
    "**/*.ts",          // 모든 TypeScript 파일
    "src/**/*.ts",      // 일반적인 src 디렉토리
    "backend/**/*.ts"   // 일반적인 backend 디렉토리
  ]
}
```

#### 5. 통합 검증

```bash
# 스킬이 복사되었는지 확인
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# skill-rules.json 문법 유효성 검사
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**사용자에게 알리기:** "[사용자의 백엔드 경로]에서 파일을 편집해보세요. 스킬이 활성화되어야 합니다."

---

### 스킬별 참고사항

#### backend-dev-guidelines
- **기술 요구사항:** Node.js/Express, Prisma, TypeScript, Sentry
- **질문:** "Express와 Prisma를 사용하시나요?" "백엔드 코드는 어디에 있나요?"
- **다른 스택인 경우:** 이것을 템플릿으로 사용하여 적용 제안
- **커스터마이징:** pathPatterns
- **경로 예시:** `api/`, `server/`, `backend/`, `services/*/src/`
- **적용 팁:** 아키텍처 패턴 (Routes→Controllers→Services)은 대부분의 프레임워크에 전용 가능

#### frontend-dev-guidelines
- **기술 요구사항:** React 18+, MUI v7, TanStack Query/Router, TypeScript
- **질문:** "React와 MUI v7을 사용하시나요?" "프론트엔드 코드는 어디에 있나요?"
- **다른 스택인 경우:** 적용된 버전 생성 제안 (Vue, Angular 등)
- **커스터마이징:** pathPatterns + 모든 프레임워크 특정 예시
- **경로 예시:** `frontend/`, `client/`, `web/`, `apps/web/src/`
- **적용 팁:** 파일 구성과 성능 패턴은 전용 가능하지만, 컴포넌트 코드는 불가

#### route-tester
- **기술 요구사항:** JWT 쿠키 기반 인증 (프레임워크 독립적)
- **질문:** "JWT 쿠키 기반 인증을 사용하시나요?"
- **아니오인 경우:** "이 스킬은 JWT 쿠키용입니다. [사용자 인증 방식]에 맞게 적용할까요, 아니면 건너뛸까요?"
- **커스터마이징:** 서비스 URL, 인증 패턴
- **호환:** JWT 쿠키를 사용하는 모든 백엔드 프레임워크

#### error-tracking
- **기술 요구사항:** Sentry (대부분의 백엔드와 호환)
- **질문:** "Sentry를 사용하시나요?" "백엔드 코드는 어디에 있나요?"
- **Sentry 없는 경우:** "[사용자의 에러 추적 도구]를 위한 템플릿으로 사용하실래요?"
- **커스터마이징:** pathPatterns
- **적용 팁:** 에러 추적 철학은 다른 도구에도 전용 가능 (Rollbar, Bugsnag 등)

#### skill-developer
- **기술 요구사항:** 없음!
- **그대로 복사** - 메타 스킬, 완전히 범용적, 어떤 기술 스택에서든 스킬 생성을 가르침

---

## 다른 기술 스택에 맞게 스킬 적용

사용자의 기술 스택이 스킬 요구사항과 다를 경우 선택지가 있습니다:

### 옵션 1: 기존 스킬 적용 (권장)

**사용 시점:** 사용자가 다른 기술로 유사한 가이드라인을 원할 때

**프로세스:**
1. **스킬을 시작점으로 복사:**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **변경이 필요한 부분 파악:**
   - 프레임워크 특정 코드 예시 (React → Vue)
   - 라이브러리 API (MUI → Vuetify/PrimeVue)
   - Import 구문
   - 컴포넌트 패턴

3. **전용 가능한 부분 유지:**
   - 파일 구성 원칙
   - 성능 최적화 전략
   - TypeScript 표준
   - 일반적인 모범 사례

4. **예시를 체계적으로 교체:**
   - 사용자에게 스택에 맞는 동등한 패턴 요청
   - 코드 예시를 사용자 프레임워크로 업데이트
   - 전체 구조와 섹션 유지

5. **스킬 이름 및 트리거 업데이트:**
   - 적절하게 스킬 이름 변경
   - 사용자 스택에 맞게 skill-rules.json 트리거 업데이트
   - 활성화 테스트

**예시 - Vue용 frontend-dev-guidelines 적용:**
```
React 스킬 구조를 기반으로 vue-dev-guidelines를 생성합니다:
- React.FC → Vue defineComponent 교체
- useSuspenseQuery → Vue composables 교체
- MUI 컴포넌트 → [사용자 컴포넌트 라이브러리] 교체
- 유지: 파일 구성, 성능 패턴, TypeScript 가이드라인

몇 분 정도 걸립니다. 진행할까요?
```

### 옵션 2: 프레임워크 독립적인 패턴 추출

**사용 시점:** 스택이 매우 다르지만 핵심 원칙은 적용될 때

**프로세스:**
1. 기존 스킬 읽기
2. 프레임워크 독립적인 패턴 파악:
   - 계층형 아키텍처 (백엔드)
   - 파일 구성 전략
   - 성능 최적화 원칙
   - 테스팅 전략
   - 에러 처리 철학

3. 해당 패턴만으로 새 스킬 생성
4. 사용자가 나중에 프레임워크 특정 예시 추가 가능

**예시:**
```
backend-dev-guidelines는 Express를 사용하지만, 계층형 아키텍처
(Routes → Controllers → Services → Repositories)는 Django에서도 작동합니다.

다음 내용으로 스킬을 생성할 수 있습니다:
- 계층형 아키텍처 패턴
- 관심사 분리 원칙
- 에러 처리 모범 사례
- 테스팅 전략

그런 다음 패턴이 확립되면 Django 특정 예시를 추가할 수 있습니다.
```

### 옵션 3: 참고용으로만 사용

**사용 시점:** 적용하기에 너무 다르지만 사용자가 영감을 원할 때

**프로세스:**
1. 사용자가 기존 스킬 탐색
2. 처음부터 새 스킬 생성 지원
3. 기존 스킬 구조를 템플릿으로 활용
4. 모듈식 패턴 따르기 (메인 + 리소스 파일)

### 기술 스택 간 전용 가능한 것들

**아키텍처 & 구성:**
- ✅ 계층형 아키텍처 (Routes/Controllers/Services 패턴)
- ✅ 관심사 분리
- ✅ 파일 구성 전략 (features/ 패턴)
- ✅ 점진적 공개 (메인 + 리소스 파일)
- ✅ 데이터 접근을 위한 Repository 패턴

**개발 관행:**
- ✅ 에러 처리 철학
- ✅ 입력 유효성 검사의 중요성
- ✅ 테스팅 전략
- ✅ 성능 최적화 원칙
- ✅ TypeScript 모범 사례

**프레임워크 특정 코드:**
- ❌ React hooks → Vue/Angular로 전용 불가
- ❌ MUI 컴포넌트 → 다른 컴포넌트 라이브러리
- ❌ Prisma 쿼리 → 다른 ORM 문법
- ❌ Express 미들웨어 → 다른 프레임워크 패턴
- ❌ 라우팅 구현 → 프레임워크 특정

### 적용 vs 건너뛰기 권장 시점

**적용 권장:**
- 사용자가 스택에 맞는 유사한 가이드라인을 원할 때
- 핵심 패턴이 적용될 때 (계층형 아키텍처 등)
- 사용자가 프레임워크 특정 예시를 도울 시간이 있을 때

**건너뛰기 권장:**
- 스택이 완전히 다를 때
- 사용자가 해당 패턴을 필요로 하지 않을 때
- 적용하는 데 너무 오래 걸릴 때
- 사용자가 처음부터 만들기를 선호할 때

---

## 훅 통합

### 필수 훅 (항상 안전하게 복사 가능)

#### skill-activation-prompt (UserPromptSubmit)

**목적:** 사용자 프롬프트 기반으로 스킬 자동 제안

**통합 (커스터마이징 불필요):**

```bash
# 두 파일 모두 복사
cp showcase/.claude/hooks/skill-activation-prompt.sh \
   $CLAUDE_PROJECT_DIR/.claude/hooks/
cp showcase/.claude/hooks/skill-activation-prompt.ts \
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 실행 권한 부여
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# 필요한 경우 의존성 설치
if [ -f "showcase/.claude/hooks/package.json" ]; then
  cp showcase/.claude/hooks/package.json \
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
fi
```

**settings.json에 추가:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**이 훅은 완전히 범용적** - 어디서든 작동, 커스터마이징 불필요!

#### post-tool-use-tracker (PostToolUse)

**목적:** 컨텍스트 관리를 위한 파일 변경 추적

**통합 (커스터마이징 불필요):**

```bash
# 파일 복사
cp showcase/.claude/hooks/post-tool-use-tracker.sh \
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 실행 권한 부여
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**settings.json에 추가:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**이 훅은 완전히 범용적** - 프로젝트 구조를 자동 감지!

---

### 선택적 훅 (대규모 커스터마이징 필요)

#### tsc-check.sh 및 trigger-build-resolver.sh (Stop 훅)

⚠️ **경고:** 이 훅들은 특정 멀티 서비스 monorepo 구조에 맞게 설정되어 있습니다.

**통합 전 질문:**
1. "여러 TypeScript 서비스가 있는 monorepo인가요?"
2. "서비스 디렉토리 이름은 무엇인가요?"
3. "tsconfig.json 파일은 어디에 있나요?"

**단순 프로젝트 (단일 서비스):**
- **건너뛰기를 권장**
- 단일 서비스 프로젝트에서는 과한 설정
- 대신 수동으로 `tsc --noEmit` 실행 가능

**복잡한 프로젝트 (멀티 서비스 monorepo):**

1. 파일 복사
2. **반드시 편집** tsc-check.sh - 다음 섹션 찾기:
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. 사용자의 실제 서비스 이름으로 교체:
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← 사용자 서비스
        echo "$repo"
        return 0
        ;;
esac
```

4. settings.json에 추가하기 전 수동 테스트:
```bash
./.claude/hooks/tsc-check.sh
```

**중요:** 이 훅이 실패하면 Stop 이벤트를 차단합니다. 설정이 작동하는 경우에만 추가하세요.

---

## 에이전트 통합

**에이전트는 독립적** - 가장 쉽게 통합 가능!

### 표준 에이전트 통합

```bash
# 에이전트 파일 복사
cp showcase/.claude/agents/[agent-name].md \
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**끝입니다!** 에이전트는 즉시 작동하며, 별도 설정 불필요.

### 하드코딩된 경로 확인

일부 에이전트가 경로를 참조할 수 있습니다. **복사 전 에이전트 파일을 읽고 다음을 확인하세요:**

- `~/git/old-project/` → `$CLAUDE_PROJECT_DIR` 또는 `.`로 변경 필요
- `/root/git/project/` → `$CLAUDE_PROJECT_DIR` 또는 `.`로 변경 필요
- 하드코딩된 스크린샷 경로 → 사용자에게 원하는 저장 위치 확인

**발견 시 업데이트:**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### 에이전트별 참고사항

**auth-route-tester / auth-route-debugger:**
- 프로젝트에 JWT 쿠키 기반 인증 필요
- 질문: "인증에 JWT 쿠키를 사용하시나요?"
- 아니오인 경우: "이 에이전트는 JWT 쿠키 인증용입니다. 건너뛸까요, 아니면 적용할까요?"

**frontend-error-fixer:**
- 스크린샷 경로 참조 가능
- 질문: "스크린샷을 어디에 저장할까요?"

**다른 모든 에이전트:**
- 그대로 복사, 완전히 범용적

---

## 슬래시 명령어 통합

```bash
# 명령어 파일 복사
cp showcase/.claude/commands/[command].md \
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### 경로 커스터마이징

명령어가 개발 문서 경로를 참조할 수 있습니다. **확인 후 업데이트:**

**dev-docs 및 dev-docs-update:**
- `dev/active/` 경로 참조 확인
- 질문: "개발 문서를 어디에 저장하고 싶으신가요?"
- 명령어 파일의 경로 업데이트

**route-research-for-testing:**
- 서비스 경로 참조 가능
- API 구조에 대해 질문

---

## 일반 패턴 & 모범 사례

### 패턴: 프로젝트 구조 파악

**가정하지 마세요:**
- ❌ "blog-api 서비스에 추가하겠습니다"
- ❌ "frontend 디렉토리로 설정하겠습니다"

**물어보세요:**
- ✅ "프로젝트 구조가 어떻게 되나요? Monorepo 또는 단일 앱?"
- ✅ "백엔드 코드는 어디에 있나요?"
- ✅ "워크스페이스를 사용하거나 여러 서비스가 있나요?"

### 패턴: skill-rules.json 커스터마이징

**워크스페이스가 있는 monorepo 사용자:**
```json
{
  "pathPatterns": [
    "packages/*/src/**/*.ts",
    "apps/*/src/**/*.tsx"
  ]
}
```

**Nx monorepo 사용자:**
```json
{
  "pathPatterns": [
    "apps/api/src/**/*.ts",
    "libs/*/src/**/*.ts"
  ]
}
```

**단순 구조 사용자:**
```json
{
  "pathPatterns": [
    "src/**/*.ts",
    "backend/**/*.ts"
  ]
}
```

### 패턴: settings.json 통합

**쇼케이스의 settings.json을 그대로 복사하지 마세요!**

대신, 필요한 섹션을 **추출하여 병합**하세요:

1. 기존 settings.json 읽기
2. 원하는 훅 설정 추가
3. 기존 설정 유지

**병합 예시:**
```json
{
  // ... 기존 설정 ...
  "hooks": {
    // ... 기존 훅 ...
    "UserPromptSubmit": [  // ← 이 섹션 추가
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 검증 체크리스트

통합 후 **다음 항목들을 검증하세요:**

```bash
# 1. 훅이 실행 가능한지 확인
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# 표시되어야 함: -rwxr-xr-x

# 2. skill-rules.json이 유효한 JSON인지 확인
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# 오류 없이 파싱되어야 함

# 3. 훅 의존성 설치 확인 (TypeScript 훅인 경우)
ls $CLAUDE_PROJECT_DIR/.claude/hooks/node_modules/
# package.json이 있다면 패키지 표시되어야 함

# 4. Settings.json이 유효한 JSON인지 확인
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# 오류 없이 파싱되어야 함
```

**그런 다음 사용자에게 테스트 요청:**
- "[관련 경로]에서 파일을 편집해보세요 - 스킬이 활성화되어야 합니다"
- "[주제]에 대해 질문해보세요 - 스킬을 제안해야 합니다"

---

## 피해야 할 일반적인 실수

### ❌ 하지 마세요: settings.json을 그대로 복사
**이유:** Stop 훅이 존재하지 않는 서비스를 참조함
**대신:** UserPromptSubmit과 PostToolUse 섹션만 추출

### ❌ 하지 마세요: 예시 서비스 이름 그대로 유지
**이유:** 사용자는 blog-api, auth-service 등이 없음
**대신:** 실제 구조를 묻고 업데이트

### ❌ 하지 마세요: 훅 실행 권한 설정 빠뜨리기
**이유:** 실행 권한 없으면 훅이 실행되지 않음
**대신:** 복사 후 항상 `chmod +x` 실행

### ❌ 하지 마세요: Monorepo 구조 가정
**이유:** 대부분의 프로젝트는 단일 서비스
**대신:** 먼저 물어보고 커스터마이징

### ❌ 하지 마세요: 모든 스킬을 한 번에 추가
**이유:** 압도적이며 모두 관련이 없을 수 있음
**대신:** 어떤 스킬이 업무에 관련이 있는지 물어보기

### ❌ 하지 마세요: Stop 훅을 테스트 없이 복사
**이유:** 실패하면 Stop 이벤트를 차단할 수 있음
**대신:** 먼저 수동 테스트, 작동하는 경우에만 추가

---

## 통합 대화 예시

### 예시 1: 백엔드 스킬 추가

**사용자:** "backend-dev-guidelines 스킬을 내 프로젝트에 추가해줘"

**Claude의 응답:**
```
backend-dev-guidelines 스킬을 추가하겠습니다. 먼저 프로젝트 구조를 이해해야 합니다:

1. 이것이 여러 서비스가 있는 monorepo인가요, 아니면 단일 백엔드 앱인가요?
2. 백엔드 코드는 어디에 있나요? (예: src/, backend/, server/)
3. Express.js를 사용하시나요?
```

**사용자:** "단일 앱이고 코드는 src/api/에 있어요, Express 써요"

**Claude가 해야 할 일:**
1. 스킬 디렉토리 복사
2. skill-rules.json에 pathPattern 업데이트: `"src/api/**/*.ts"`
3. 올바르게 복사됐는지 확인
4. 응답:
```
✅ backend-dev-guidelines 스킬 추가 완료!

설정:
- src/api/의 파일 편집 시 스킬 활성화
- "controller", "service", "route", "API" 키워드로 활성화

테스트: src/api/의 파일을 편집해보세요. 스킬 제안이 표시되어야 합니다.
```

### 예시 2: 훅 설정

**사용자:** "스킬 활성화 훅을 설정해줘"

**Claude의 응답:**
```
스킬 자동 활성화 시스템을 설정하겠습니다. 두 가지 훅이 필요합니다:
1. skill-activation-prompt - 프롬프트 기반으로 스킬 제안
2. post-tool-use-tracker - 파일 변경 추적

이미 .claude/settings.json 파일이 있나요, 아니면 새로 만들까요?
```

**사용자:** "settings.json 있어요"

**Claude가 해야 할 일:**
1. settings.json 읽기
2. 두 훅 파일 복사
3. hooks 디렉토리에서 npm install 실행
4. 기존 설정 유지하며 훅 설정 추가
5. 훅 실행 권한 설정
6. 응답:
```
✅ 스킬 활성화 훅 설치 완료!

작업 내용:
- skill-activation-prompt 훅 복사
- post-tool-use-tracker 훅 복사
- 의존성 설치
- settings.json에 추가

테스트: "백엔드 개발 도움"이라고 물어보면 backend-dev-guidelines 스킬을 제안해야 합니다.
```

### 예시 3: 에이전트 추가

**사용자:** "code-architecture-reviewer 에이전트 추가해줘"

**Claude의 응답:**
```
code-architecture-reviewer 에이전트는 코드의 아키텍처 문제와 모범 사례를 검토합니다.
이 에이전트는 독립적으로 작동하며, 별도 설정이 필요 없습니다.

지금 .claude/agents/ 디렉토리에 복사하겠습니다.
```

**그런 다음:**
1. 에이전트 파일 복사
2. 응답:
```
✅ code-architecture-reviewer 에이전트 추가 완료!

Task 도구로 실행:
- subagent_type: "code-architecture-reviewer"
- prompt: 검토할 코드 설명

에이전트가 코드를 검토하고 아키텍처 피드백을 제공합니다.
```

### 예시 4: 기술 스택 불일치

**사용자:** "내 Vue 프로젝트에 frontend-dev-guidelines 스킬 추가해줘"

**Claude의 응답:**
```
Vue를 사용하시는군요. frontend-dev-guidelines 스킬은 React + MUI v7 전용으로 설계되었습니다. 선택지가 있습니다:

옵션 1 (권장): React 스킬을 템플릿으로 사용하여 vue-dev-guidelines를 생성합니다. 다음을 적용합니다:
- React.FC → Vue defineComponent
- useSuspenseQuery → Vue composables
- MUI 예시 → 사용자 컴포넌트 라이브러리
- 유지: 파일 구성, 성능 원칙, TypeScript 표준

옵션 2: 프레임워크 독립적인 패턴만 추출 (파일 구성, 성능, 테스팅).
React 특정 코드 제외.

옵션 3: 건너뛰고 처음부터 Vue 가이드라인 생성 지원.

어떤 방법을 선호하시나요?
```

**사용자:** "옵션 1이 좋아요, Vuetify 써요"

**Claude가 해야 할 일:**
1. frontend-dev-guidelines를 vue-dev-guidelines로 복사
2. 각 리소스 파일 검토
3. React 패턴을 Vue Composition API로 교체
4. MUI 컴포넌트를 Vuetify로 교체
5. Vue 특정 트리거(.vue 파일)로 skill-rules.json 업데이트
6. 응답:
```
✅ Vue 3 + Vuetify에 맞게 적용된 vue-dev-guidelines 스킬 생성 완료!

변경 사항:
- React.FC 컴포넌트 → Vue defineComponent
- MUI v7 예시 → Vuetify 컴포넌트
- TanStack Query → 사용자 데이터 가져오기 방식
- 파일 패턴 → .vue 파일

유지된 내용:
- 파일 구성 (features/ 패턴)
- 성능 최적화 전략
- TypeScript 표준
- 로딩/에러 상태 원칙

.vue 파일을 편집해보세요 - 스킬이 활성화되어야 합니다.
```

---

## 빠른 참조 표

### 커스터마이징이 필요한 것?

| 컴포넌트 | 기술 요구사항 | 커스터마이징 | 물어볼 것 |
|-----------|------------------|--------------|-------------|
| **skill-developer** | 없음 | ✅ 없음 | 그대로 복사 |
| **backend-dev-guidelines** | Express/Prisma/Node | ⚠️ 경로 + 기술 확인 | "Express/Prisma 사용?" "백엔드 위치?" |
| **frontend-dev-guidelines** | React/MUI v7 | ⚠️⚠️ 경로 + 프레임워크 | "React/MUI v7 사용?" "프론트엔드 위치?" |
| **route-tester** | JWT 쿠키 | ⚠️ 인증 + 경로 | "JWT 쿠키 인증?" |
| **error-tracking** | Sentry | ⚠️ 경로 | "Sentry 사용?" "백엔드 위치?" |
| **skill-activation-prompt** | ✅ 없음 | 그대로 복사 | |
| **post-tool-use-tracker** | ✅ 없음 | 그대로 복사 | |
| **tsc-check** | ⚠️⚠️⚠️ 대규모 | "Monorepo 또는 단일 서비스?" |
| **모든 에이전트** | ✅ 최소 | 경로 확인 |
| **모든 명령어** | ⚠️ 경로 | "개발 문서 어디에?" |

### 건너뛰기를 권장하는 경우

| 컴포넌트 | 건너뛰기 조건 |
|-----------|-----------|
| **tsc-check 훅** | 단일 서비스 프로젝트 또는 다른 빌드 설정 |
| **route-tester** | JWT 쿠키 인증을 사용하지 않는 경우 |
| **frontend-dev-guidelines** | React + MUI를 사용하지 않는 경우 |
| **auth 에이전트** | JWT 쿠키 인증을 사용하지 않는 경우 |

---

## Claude를 위한 최종 팁

**사용자가 "모두 추가해줘"라고 할 때:**
- 필수품부터 시작: 스킬 활성화 훅 + 관련 스킬 1-2개
- 스킬 5개 + 에이전트 10개로 압도하지 말 것
- 실제로 무엇이 필요한지 물어보기

**무언가 작동하지 않을 때:**
- 검증 체크리스트 확인
- 경로가 구조와 일치하는지 확인
- 훅 수동 테스트
- JSON 문법 오류 확인

**사용자가 확신이 없을 때:**
- 스킬 활성화 훅만으로 시작하기 권장
- 백엔드 또는 프론트엔드 스킬 하나 추가 (사용하는 것)
- 나중에 필요에 따라 더 추가

**항상 하는 일 설명:**
- 실행 중인 명령어 보여주기
- 질문하는 이유 설명
- 통합 후 명확한 다음 단계 제공

---

**기억하세요:** 이것은 참고 라이브러리이지 작동하는 애플리케이션이 아닙니다. 당신의 역할은 사용자가 **자신의 특정 프로젝트 구조**에 맞게 컴포넌트를 선택하고 적용하도록 돕는 것입니다.
