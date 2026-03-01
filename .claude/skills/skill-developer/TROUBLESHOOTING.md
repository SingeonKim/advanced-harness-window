# 트러블슈팅 - 스킬 활성화 문제

스킬 활성화 문제를 위한 완전한 디버깅 가이드.

## 목차

- [스킬이 트리거되지 않음](#스킬이-트리거되지-않음)
  - [UserPromptSubmit이 제안하지 않음](#userpromptsubmit이-제안하지-않음)
  - [PreToolUse가 차단하지 않음](#pretooluse가-차단하지-않음)
- [오탐 (False Positives)](#오탐-false-positives)
- [훅이 실행되지 않음](#훅이-실행되지-않음)
- [성능 문제](#성능-문제)

---

## 스킬이 트리거되지 않음

### UserPromptSubmit이 제안하지 않음

**증상:** 질문을 해도 출력에 스킬 제안이 나타나지 않음.

**일반적인 원인:**

####  1. 키워드가 매칭되지 않음

**확인:**
- skill-rules.json의 `promptTriggers.keywords` 확인
- 해당 키워드가 실제로 프롬프트에 있는지 확인
- 대소문자를 구분하지 않는 부분 문자열 매칭임을 기억

**예시:**
```json
"keywords": ["layout", "grid"]
```
- "how does the layout work?" → 매칭 ("layout")
- "how does the grid system work?" → 매칭 ("grid")
- "how do layouts work?" → 매칭 ("layout")
- "how does it work?" → 미매칭

**해결:** skill-rules.json에 더 많은 키워드 변형 추가

#### 2. 의도 패턴이 너무 구체적

**확인:**
- `promptTriggers.intentPatterns` 확인
- https://regex101.com/ 에서 regex 테스트
- 더 광범위한 패턴이 필요할 수 있음

**예시:**
```json
"intentPatterns": [
  "(create|add).*?(database.*?table)"  // 너무 구체적
]
```
- "create a database table" → 매칭
- "add new table" → 미매칭 ("database" 없음)

**해결:** 패턴 확장:
```json
"intentPatterns": [
  "(create|add).*?(table|database)"  // 더 나음
]
```

#### 3. 스킬 이름에 오타

**확인:**
- SKILL.md frontmatter의 스킬 이름
- skill-rules.json의 스킬 이름
- 정확히 일치해야 함

**예시:**
```yaml
# SKILL.md
name: project-catalog-developer
```
```json
// skill-rules.json
"project-catalogue-developer": {  // 오타: catalogue vs catalog
  ...
}
```

**해결:** 이름을 정확히 일치시킴

#### 4. JSON 문법 오류

**확인:**
```bash
cat .claude/skills/skill-rules.json | jq .
```

JSON이 유효하지 않으면 jq가 오류를 표시함.

**일반적인 오류:**
- 후행 쉼표
- 따옴표 누락
- 이중 따옴표 대신 단일 따옴표 사용
- 문자열에서 이스케이프되지 않은 문자

**해결:** JSON 문법 수정, jq로 검증

#### 디버그 명령

훅을 수동으로 테스트:

```bash
echo '{"session_id":"debug","prompt":"your test prompt here"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

예상 결과: 스킬이 출력에 나타나야 함.

---

### PreToolUse가 차단하지 않음

**증상:** 가드레일을 트리거해야 하는 파일을 편집해도 차단이 발생하지 않음.

**일반적인 원인:**

#### 1. 파일 경로가 패턴과 매칭되지 않음

**확인:**
- 편집 중인 파일 경로
- skill-rules.json의 `fileTriggers.pathPatterns`
- Glob 패턴 문법

**예시:**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx"
]
```
- 편집: `frontend/src/components/Dashboard.tsx` → 매칭
- 편집: `frontend/tests/Dashboard.test.tsx` → 매칭 (제외 추가 필요!)
- 편집: `backend/src/app.ts` → 미매칭

**해결:** Glob 패턴 조정 또는 누락된 경로 추가

#### 2. pathExclusions에 의해 제외됨

**확인:**
- 테스트 파일을 편집하고 있는지 확인
- `fileTriggers.pathExclusions` 확인

**예시:**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.spec.ts"
]
```
- 편집: `services/user.test.ts` → 제외됨
- 편집: `services/user.ts` → 제외되지 않음

**해결:** 테스트 제외 범위가 너무 넓으면 좁히거나 제거

#### 3. 콘텐츠 패턴을 찾을 수 없음

**확인:**
- 파일에 실제로 패턴이 포함되어 있는지 확인
- `fileTriggers.contentPatterns` 확인
- regex가 올바른지 확인

**예시:**
```json
"contentPatterns": [
  "import.*[Pp]risma"
]
```
- 파일에 포함: `import { PrismaService } from './prisma'` → 매칭
- 파일에 포함: `import { Database } from './db'` → 미매칭

**디버그:**
```bash
# 파일에 패턴이 있는지 확인
grep -i "prisma" path/to/file.ts
```

**해결:** 콘텐츠 패턴 조정 또는 누락된 임포트 추가

#### 4. 세션에서 이미 스킬을 사용함

**세션 상태 확인:**
```bash
ls .claude/hooks/state/
cat .claude/hooks/state/skills-used-{session-id}.json
```

**예시:**
```json
{
  "skills_used": ["database-verification"],
  "files_verified": []
}
```

스킬이 `skills_used`에 있으면 이 세션에서 다시 차단하지 않음.

**해결:** 초기화하려면 상태 파일 삭제:
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

#### 5. 파일 마커 존재

**파일에서 건너뛰기 마커 확인:**
```bash
grep "@skip-validation" path/to/file.ts
```

발견되면 파일이 영구적으로 건너뛰어짐.

**해결:** 다시 검증이 필요하면 마커 제거

#### 6. 환경 변수 오버라이드

**확인:**
```bash
echo $SKIP_DB_VERIFICATION
echo $SKIP_SKILL_GUARDRAILS
```

설정되어 있으면 스킬이 비활성화됨.

**해결:** 환경 변수 해제:
```bash
unset SKIP_DB_VERIFICATION
```

#### 디버그 명령

훅을 수동으로 테스트:

```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts 2>&1
{
  "session_id": "debug",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/root/git/your-project/form/src/services/user.ts"}
}
EOF
echo "Exit code: $?"
```

예상 결과:
- 차단해야 하는 경우: 종료 코드 2 + stderr 메시지
- 허용해야 하는 경우: 종료 코드 0 + 출력 없음

---

## 오탐 (False Positives)

**증상:** 트리거되어서는 안 되는 상황에서 스킬이 트리거됨.

**일반적인 원인 및 해결책:**

### 1. 키워드가 너무 일반적

**문제:**
```json
"keywords": ["user", "system", "create"]  // 너무 광범위
```
- 다음 상황에서 트리거: "user manual", "file system", "create directory"

**해결:** 키워드를 더 구체적으로 만들기
```json
"keywords": [
  "user authentication",
  "user tracking",
  "create feature"
]
```

### 2. 의도 패턴이 너무 광범위

**문제:**
```json
"intentPatterns": [
  "(create)"  // "create"가 있는 모든 것에 매칭
]
```
- 다음 상황에서 트리거: "create file", "create folder", "create account"

**해결:** 패턴에 컨텍스트 추가
```json
"intentPatterns": [
  "(create|add).*?(database|table|feature)"  // 더 구체적
]
```

**고급:** 부정 전방탐색 사용으로 특정 경우 제외
```regex
(create)(?!.*test).*?(feature)  // "test"가 나타나면 매칭 안 함
```

### 3. 파일 경로가 너무 일반적

**문제:**
```json
"pathPatterns": [
  "form/**"  // form/의 모든 것에 매칭
]
```
- 다음에서 트리거: 테스트 파일, 설정 파일, 모든 것

**해결:** 더 좁은 패턴 사용
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 서비스 파일만
  "form/src/controllers/**/*.ts"
]
```

### 4. 콘텐츠 패턴이 관련 없는 코드를 캐치

**문제:**
```json
"contentPatterns": [
  "Prisma"  // 주석, 문자열 등에서도 매칭
]
```
- 다음 상황에서 트리거: `// Don't use Prisma here`
- 다음 상황에서 트리거: `const note = "Prisma is cool"`

**해결:** 패턴을 더 구체적으로 만들기
```json
"contentPatterns": [
  "import.*[Pp]risma",        // 임포트만
  "PrismaService\\.",         // 실제 사용만
  "prisma\\.(findMany|create)" // 특정 메서드
]
```

### 5. 강제 수준 조정

**최후 수단:** 오탐이 빈번한 경우:

```json
{
  "enforcement": "block"  // "suggest"로 변경
}
```

이렇게 하면 차단 대신 권고로 바뀜.

---

## 훅이 실행되지 않음

**증상:** 훅이 전혀 실행되지 않음 - 제안도 없고 차단도 없음.

**일반적인 원인:**

### 1. 훅이 등록되지 않음

**`.claude/settings.json` 확인:**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
cat .claude/settings.json | jq '.hooks.PreToolUse'
```

예상 결과: 훅 항목이 존재해야 함

**해결:** 누락된 훅 등록 추가:
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

### 2. Bash 래퍼가 실행 가능하지 않음

**확인:**
```bash
ls -l .claude/hooks/*.sh
```

예상 결과: `-rwxr-xr-x` (실행 가능)

**해결:**
```bash
chmod +x .claude/hooks/*.sh
```

### 3. 잘못된 Shebang

**확인:**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
```

예상 결과: `#!/bin/bash`

**해결:** 첫 번째 줄에 올바른 shebang 추가

### 4. npx/tsx를 사용할 수 없음

**확인:**
```bash
npx tsx --version
```

예상 결과: 버전 번호

**해결:** 의존성 설치:
```bash
cd .claude/hooks
npm install
```

### 5. TypeScript 컴파일 오류

**확인:**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
```

예상 결과: 출력 없음 (오류 없음)

**해결:** TypeScript 문법 오류 수정

---

## 성능 문제

**증상:** 훅이 느림, 프롬프트/편집 전에 눈에 띄는 지연 발생.

**일반적인 원인:**

### 1. 너무 많은 패턴

**확인:**
- skill-rules.json의 패턴 수 세기
- 각 패턴 = regex 컴파일 + 매칭

**해결:** 패턴 줄이기
- 유사한 패턴 결합
- 중복 패턴 제거
- 더 구체적인 패턴 사용 (더 빠른 매칭)

### 2. 복잡한 Regex

**문제:**
```regex
(create|add|modify|update|implement|build).*?(feature|endpoint|route|service|controller|component|UI|page)
```
- 긴 대안 목록 = 느림

**해결:** 단순화
```regex
(create|add).*?(feature|endpoint)  // 더 적은 대안
```

### 3. 너무 많은 파일 확인

**문제:**
```json
"pathPatterns": [
  "**/*.ts"  // 모든 TypeScript 파일 확인
]
```

**해결:** 더 구체적으로
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 특정 디렉토리만
  "form/src/controllers/**/*.ts"
]
```

### 4. 대용량 파일

콘텐츠 패턴 매칭은 파일 전체를 읽음 - 대용량 파일에서 느림.

**해결:**
- 꼭 필요한 경우에만 콘텐츠 패턴 사용
- 파일 크기 제한 고려 (향후 개선 사항)

### 성능 측정

```bash
# UserPromptSubmit
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

**목표 지표:**
- UserPromptSubmit: < 100ms
- PreToolUse: < 200ms

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 스킬 가이드
- [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md) - 훅 작동 방식
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 설정 참조
