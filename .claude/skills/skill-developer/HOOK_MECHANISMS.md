# 훅 메커니즘 - 심층 분석

UserPromptSubmit 및 PreToolUse 훅이 어떻게 작동하는지 기술적으로 심층 분석합니다.

## 목차

- [UserPromptSubmit 훅 흐름](#userpromptsubmit-훅-흐름)
- [PreToolUse 훅 흐름](#pretooluse-훅-흐름)
- [종료 코드 동작 (중요)](#종료-코드-동작-중요)
- [세션 상태 관리](#세션-상태-관리)
- [성능 고려 사항](#성능-고려-사항)

---

## UserPromptSubmit 훅 흐름

### 실행 순서

```
사용자가 프롬프트 제출
    ↓
.claude/settings.json이 훅 등록
    ↓
skill-activation-prompt.sh 실행
    ↓
npx tsx skill-activation-prompt.ts
    ↓
훅이 stdin 읽기 (프롬프트가 있는 JSON)
    ↓
skill-rules.json 로드
    ↓
키워드 + 의도 패턴 매칭
    ↓
우선순위별 매칭 그룹화 (critical → high → medium → low)
    ↓
형식화된 메시지를 stdout으로 출력
    ↓
stdout이 Claude의 컨텍스트가 됨 (프롬프트 전에 주입)
    ↓
Claude가 봄: [스킬 제안] + 사용자 프롬프트
```

### 핵심 사항

- **종료 코드**: 항상 0 (허용)
- **stdout**: → Claude의 컨텍스트 (시스템 메시지로 주입)
- **타이밍**: Claude가 프롬프트를 처리하기 전에 실행
- **동작**: 비차단, 권고만
- **목적**: Claude가 관련 스킬을 인식하도록 함

### 입력 형식

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "how does the layout system work?"
}
```

### 출력 형식 (stdout으로)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 SKILL ACTIVATION CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 RECOMMENDED SKILLS:
  → project-catalog-developer

ACTION: Use Skill tool BEFORE responding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Claude는 사용자 프롬프트를 처리하기 전에 이 출력을 추가 컨텍스트로 봅니다.

---

## PreToolUse 훅 흐름

### 실행 순서

```
Claude가 Edit/Write 도구 호출
    ↓
.claude/settings.json이 훅 등록 (matcher: Edit|Write)
    ↓
skill-verification-guard.sh 실행
    ↓
npx tsx skill-verification-guard.ts
    ↓
훅이 stdin 읽기 (tool_name, tool_input이 있는 JSON)
    ↓
skill-rules.json 로드
    ↓
파일 경로 패턴 확인 (glob 매칭)
    ↓
콘텐츠 패턴을 위해 파일 읽기 (파일이 존재하는 경우)
    ↓
세션 상태 확인 (스킬이 이미 사용됐는가?)
    ↓
건너뛰기 조건 확인 (파일 마커, 환경 변수)
    ↓
매칭되고 건너뛰기 않은 경우:
  세션 상태 업데이트 (스킬이 강제됨으로 표시)
  차단 메시지를 stderr로 출력
  코드 2로 종료 (차단)
아닌 경우:
  코드 0으로 종료 (허용)
    ↓
차단된 경우:
  stderr → Claude가 메시지 봄
  Edit/Write 도구가 실행되지 않음
  Claude가 스킬을 사용하고 재시도해야 함
허용된 경우:
  도구가 정상 실행
```

### 핵심 사항

- **종료 코드 2**: 차단 (stderr → Claude)
- **종료 코드 0**: 허용
- **타이밍**: 도구 실행 전에 실행
- **세션 추적**: 같은 세션에서 반복 차단 방지
- **실패 개방**: 오류 시 작업 허용 (워크플로우 방해 없음)
- **목적**: 중요한 가드레일 강제

### 입력 형식

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/root/git/your-project/form/src/services/user.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

### 출력 형식 (차단 시 stderr로)

```
⚠️ BLOCKED - Database Operation Detected

📋 REQUIRED ACTION:
1. Use Skill tool: 'database-verification'
2. Verify ALL table and column names against schema
3. Check database structure with DESCRIBE commands
4. Then retry this edit

Reason: Prevent column name errors in Prisma queries
File: form/src/services/user.ts

💡 TIP: Add '// @skip-validation' comment to skip future checks
```

Claude는 이 메시지를 받고 편집을 재시도하기 전에 스킬을 사용해야 한다는 것을 이해합니다.

---

## 종료 코드 동작 (중요)

### 종료 코드 참조 표

| 종료 코드 | stdout | stderr | 도구 실행 | Claude가 봄 |
|-----------|--------|--------|-----------|-------------|
| 0 (UserPromptSubmit) | → 컨텍스트 | → 사용자만 | 해당 없음 | stdout 내용 |
| 0 (PreToolUse) | → 사용자만 | → 사용자만 | **진행** | 없음 |
| 2 (PreToolUse) | → 사용자만 | → **Claude** | **차단** | stderr 내용 |
| 기타 | → 사용자만 | → 사용자만 | 차단 | 없음 |

### 종료 코드 2가 중요한 이유

이것이 강제의 핵심 메커니즘입니다:

1. PreToolUse에서 Claude에게 메시지를 보내는 **유일한 방법**
2. stderr 내용이 "Claude에게 자동으로 피드백"됨
3. Claude가 차단 메시지를 보고 무엇을 해야 하는지 이해
4. 도구 실행이 방지됨
5. 가드레일 강제에 중요

### 예시 대화 흐름

```
사용자: "Add a new user service with Prisma"

Claude: "I'll create the user service..."
    [form/src/services/user.ts 편집 시도]

PreToolUse 훅: [종료 코드 2]
    stderr: "⚠️ BLOCKED - Use database-verification"

Claude가 오류를 보고 응답:
    "I need to verify the database schema first."
    [스킬 도구 사용: database-verification]
    [컬럼명 검증]
    [편집 재시도 - 이제 허용 (세션 추적)]
```

---

## 세션 상태 관리

### 목적

같은 세션에서 반복 알림 방지 - Claude가 스킬을 사용하면 다시 차단하지 않음.

### 상태 파일 위치

`.claude/hooks/state/skills-used-{session_id}.json`

### 상태 파일 구조

```json
{
  "skills_used": [
    "database-verification",
    "error-tracking"
  ],
  "files_verified": []
}
```

### 작동 방식

1. **Prisma가 있는 파일의 첫 번째 편집**:
   - 훅이 종료 코드 2로 차단
   - 세션 상태 업데이트: skills_used에 "database-verification" 추가
   - Claude가 메시지를 보고 스킬 사용

2. **두 번째 편집 (같은 세션)**:
   - 훅이 세션 상태 확인
   - skills_used에서 "database-verification" 발견
   - 코드 0으로 종료 (허용)
   - Claude에게 메시지 없음

3. **다른 세션**:
   - 새 세션 ID = 새 상태 파일
   - 훅이 다시 차단

### 제한 사항

훅은 스킬이 *실제로* 호출됐는지 감지할 수 없음 - 세션당 스킬당 한 번만 차단합니다. 즉:

- Claude가 스킬을 사용하지 않지만 다른 편집을 하면 다시 차단하지 않음
- Claude가 지시를 따른다고 신뢰
- 향후 개선: 실제 스킬 도구 사용 감지

---

## 성능 고려 사항

### 목표 지표

- **UserPromptSubmit**: < 100ms
- **PreToolUse**: < 200ms

### 성능 병목 지점

1. **skill-rules.json 로드** (매 실행마다)
   - 향후: 메모리에 캐시
   - 향후: 변경 사항 감시, 필요시에만 다시 로드

2. **파일 내용 읽기** (PreToolUse)
   - contentPatterns가 설정된 경우에만
   - 파일이 존재하는 경우에만
   - 대용량 파일에서 느릴 수 있음

3. **Glob 매칭** (PreToolUse)
   - 각 패턴에 대한 Regex 컴파일
   - 향후: 한 번 컴파일, 캐시

4. **Regex 매칭** (두 훅 모두)
   - 의도 패턴 (UserPromptSubmit)
   - 콘텐츠 패턴 (PreToolUse)
   - 향후: 지연 컴파일, 컴파일된 regex 캐시

### 최적화 전략

**패턴 줄이기:**
- 더 구체적인 패턴 사용 (확인할 패턴 줄이기)
- 가능한 경우 유사한 패턴 결합

**파일 경로 패턴:**
- 더 구체적 = 확인할 파일 줄어듦
- 예: `form/src/services/**`가 `form/**`보다 좋음

**콘텐츠 패턴:**
- 진정으로 필요할 때만 추가
- 단순한 regex = 더 빠른 매칭

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 스킬 가이드
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 훅 문제 디버깅
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 설정 참조
