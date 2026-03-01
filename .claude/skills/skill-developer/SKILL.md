---
name: skill-developer
description: Create and manage Claude Code skills following Anthropic best practices. Use when creating new skills, modifying skill-rules.json, understanding trigger patterns, working with hooks, debugging skill activation, or implementing progressive disclosure. Covers skill structure, YAML frontmatter, trigger types (keywords, intent patterns, file paths, content patterns), enforcement levels (block, suggest, warn), hook mechanisms (UserPromptSubmit, PreToolUse), session tracking, and the 500-line rule.
---

# 스킬 개발자 가이드

## 목적

Anthropic의 공식 모범 사례(500줄 규칙 및 점진적 공개 패턴 포함)를 따르는 자동 활성화 시스템으로 Claude Code에서 스킬을 생성하고 관리하는 종합 가이드.

## 이 스킬을 사용하는 경우

다음을 언급할 때 자동 활성화됩니다:
- 스킬 생성 또는 추가
- 스킬 트리거 또는 규칙 수정
- 스킬 활성화 작동 방식 이해
- 스킬 활성화 문제 디버깅
- skill-rules.json 작업
- 훅 시스템 메커니즘
- Claude Code 모범 사례
- 점진적 공개
- YAML frontmatter
- 500줄 규칙

---

## 시스템 개요

### 두 가지 훅 아키텍처

**1. UserPromptSubmit 훅** (사전 제안)
- **파일**: `.claude/hooks/skill-activation-prompt.ts`
- **트리거**: Claude가 사용자 프롬프트를 보기 전
- **목적**: 키워드 + 의도 패턴 기반으로 관련 스킬 제안
- **방법**: 형식화된 알림을 컨텍스트로 주입 (stdout → Claude의 입력)
- **사용 사례**: 주제 기반 스킬, 암묵적 작업 감지

**2. Stop 훅 - 오류 처리 알림** (부드러운 알림)
- **파일**: `.claude/hooks/error-handling-reminder.ts`
- **트리거**: Claude가 응답 완료 후
- **목적**: 작성된 코드의 오류 처리를 자가 평가하도록 부드러운 알림
- **방법**: 위험한 패턴에 대해 편집된 파일 분석, 필요시 알림 표시
- **사용 사례**: 워크플로우 차단 없이 오류 처리 인식

**철학 변경 (2025-10-27):** Sentry/오류 처리를 위한 PreToolUse 차단에서 벗어나 코드 품질 인식을 유지하면서 워크플로우를 방해하지 않는 부드러운 사후 응답 알림을 사용합니다.

### 설정 파일

**위치**: `.claude/skills/skill-rules.json`

정의 내용:
- 모든 스킬 및 트리거 조건
- 강제 수준 (block, suggest, warn)
- 파일 경로 패턴 (glob)
- 콘텐츠 감지 패턴 (regex)
- 건너뛰기 조건 (세션 추적, 파일 마커, 환경 변수)

---

## 스킬 유형

### 1. 가드레일 스킬

**목적:** 오류를 방지하는 중요한 모범 사례 강제

**특성:**
- 유형: `"guardrail"`
- 강제: `"block"`
- 우선순위: `"critical"` 또는 `"high"`
- 스킬이 사용될 때까지 파일 편집 차단
- 일반적인 실수 방지 (컬럼명, 중요한 오류)
- 세션 인식 (같은 세션에서 반복 알림 없음)

**예시:**
- `database-verification` - Prisma 쿼리 전 테이블/컬럼명 검증
- `frontend-dev-guidelines` - React/TypeScript 패턴 강제

**사용 시기:**
- 런타임 오류를 유발하는 실수
- 데이터 무결성 우려
- 중요한 호환성 문제

### 2. 도메인 스킬

**목적:** 특정 영역에 대한 포괄적인 안내 제공

**특성:**
- 유형: `"domain"`
- 강제: `"suggest"`
- 우선순위: `"high"` 또는 `"medium"`
- 필수가 아닌 권고 사항
- 주제 또는 도메인 특화
- 포괄적인 문서

**예시:**
- `backend-dev-guidelines` - Node.js/Express/TypeScript 패턴
- `frontend-dev-guidelines` - React/TypeScript 모범 사례
- `error-tracking` - Sentry 통합 안내

**사용 시기:**
- 심층 지식이 필요한 복잡한 시스템
- 모범 사례 문서
- 아키텍처 패턴
- 사용 방법 가이드

---

## 빠른 시작: 새 스킬 생성

### 1단계: 스킬 파일 생성

**위치:** `.claude/skills/{skill-name}/SKILL.md`

**템플릿:**
```markdown
---
name: my-new-skill
description: Brief description including keywords that trigger this skill. Mention topics, file types, and use cases. Be explicit about trigger terms.
---

# My New Skill

## Purpose
What this skill helps with

## When to Use
Specific scenarios and conditions

## Key Information
The actual guidance, documentation, patterns, examples
```

**모범 사례:**
- 이름: 소문자, 하이픈, 동명사 형태 (동사 + -ing) 선호
- 설명: 모든 트리거 키워드/구문 포함 (최대 1024자)
- 내용: 500줄 미만 - 세부 사항은 참조 파일 사용
- 예시: 실제 코드 예시
- 구조: 명확한 제목, 목록, 코드 블록

### 2단계: skill-rules.json에 추가

전체 스키마는 [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) 참조.

**기본 템플릿:**
```json
{
  "my-new-skill": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    }
  }
}
```

### 3단계: 트리거 테스트

**UserPromptSubmit 테스트:**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**PreToolUse 테스트:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

### 4단계: 패턴 개선

테스트 결과를 바탕으로:
- 누락된 키워드 추가
- 오탐을 줄이기 위해 의도 패턴 개선
- 파일 경로 패턴 조정
- 실제 파일에 대해 콘텐츠 패턴 테스트

### 5단계: Anthropic 모범 사례 따르기

- SKILL.md를 500줄 미만으로 유지
- 참조 파일로 점진적 공개 사용
- 100줄 이상의 참조 파일에 목차 추가
- 트리거 키워드가 포함된 상세한 설명 작성
- 광범위한 문서 작성 전에 3개 이상의 실제 시나리오 테스트
- 실제 사용을 기반으로 반복 개선

---

## 강제 수준

### BLOCK (중요한 가드레일)

- Edit/Write 도구 실행을 물리적으로 차단
- 훅에서 종료 코드 2, stderr → Claude
- Claude가 메시지를 보고 계속하려면 스킬 사용 필요
- **사용 시**: 중요한 실수, 데이터 무결성, 보안 문제

**예시:** 데이터베이스 컬럼명 검증

### SUGGEST (권장)

- Claude가 프롬프트를 보기 전에 주입된 알림
- Claude가 관련 스킬을 인식
- 강제되지 않고 권고 사항만
- **사용 시**: 도메인 안내, 모범 사례, 사용 방법 가이드

**예시:** 프론트엔드 개발 가이드라인

### WARN (선택 사항)

- 낮은 우선순위 제안
- 권고 사항만, 최소한의 강제
- **사용 시**: 있으면 좋은 제안, 정보 알림

**거의 사용되지 않음** - 대부분의 스킬은 BLOCK 또는 SUGGEST입니다.

---

## 건너뛰기 조건 및 사용자 제어

### 1. 세션 추적

**목적:** 같은 세션에서 반복 알림 방지

**작동 방식:**
- 첫 번째 편집 → 훅이 차단하고 세션 상태 업데이트
- 두 번째 편집 (같은 세션) → 훅이 허용
- 다른 세션 → 다시 차단

**상태 파일:** `.claude/hooks/state/skills-used-{session_id}.json`

### 2. 파일 마커

**목적:** 검증된 파일의 영구적 건너뛰기

**마커:** `// @skip-validation`

**사용법:**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// This file has been manually verified
```

**참고:** 과도한 사용은 목적을 훼손하므로 드물게 사용

### 3. 환경 변수

**목적:** 비상 비활성화, 임시 재정의

**전체 비활성화:**
```bash
export SKIP_SKILL_GUARDRAILS=true  # 모든 PreToolUse 차단 비활성화
```

**스킬별:**
```bash
export SKIP_DB_VERIFICATION=true
export SKIP_ERROR_REMINDER=true
```

---

## 테스트 체크리스트

새 스킬 생성 시 확인:

- [ ] `.claude/skills/{name}/SKILL.md`에 스킬 파일 생성됨
- [ ] name과 description이 있는 올바른 frontmatter
- [ ] `skill-rules.json`에 항목 추가됨
- [ ] 실제 프롬프트로 키워드 테스트됨
- [ ] 다양한 변형으로 의도 패턴 테스트됨
- [ ] 실제 파일로 파일 경로 패턴 테스트됨
- [ ] 파일 내용에 대해 콘텐츠 패턴 테스트됨
- [ ] 차단 메시지가 명확하고 실행 가능함 (가드레일인 경우)
- [ ] 건너뛰기 조건이 적절히 설정됨
- [ ] 우선순위 수준이 중요도와 일치함
- [ ] 테스트에서 오탐 없음
- [ ] 테스트에서 미탐 없음
- [ ] 성능 허용 가능 (<100ms 또는 <200ms)
- [ ] JSON 문법 검증: `jq . skill-rules.json`
- [ ] **SKILL.md가 500줄 미만** ⭐
- [ ] 필요한 경우 참조 파일 생성됨
- [ ] 100줄 이상의 파일에 목차 추가됨

---

## 참조 파일

특정 주제에 대한 자세한 내용은 다음을 참조하세요:

### [TRIGGER_TYPES.md](TRIGGER_TYPES.md)
모든 트리거 유형에 대한 완전한 가이드:
- 키워드 트리거 (명시적 주제 매칭)
- 의도 패턴 (암묵적 작업 감지)
- 파일 경로 트리거 (glob 패턴)
- 콘텐츠 패턴 (파일 내 regex)
- 각각의 모범 사례 및 예시
- 일반적인 함정 및 테스트 전략

### [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)
완전한 skill-rules.json 스키마:
- 전체 TypeScript 인터페이스 정의
- 필드별 설명
- 완전한 가드레일 스킬 예시
- 완전한 도메인 스킬 예시
- 검증 가이드 및 일반적인 오류

### [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md)
훅 내부 심층 분석:
- UserPromptSubmit 흐름 (상세)
- PreToolUse 흐름 (상세)
- 종료 코드 동작 표 (중요)
- 세션 상태 관리
- 성능 고려 사항

### [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
포괄적인 디버깅 가이드:
- 스킬이 트리거되지 않음 (UserPromptSubmit)
- PreToolUse가 차단하지 않음
- 오탐 (너무 많은 트리거)
- 훅이 전혀 실행되지 않음
- 성능 문제

### [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md)
바로 사용 가능한 패턴 모음:
- 의도 패턴 라이브러리 (regex)
- 파일 경로 패턴 라이브러리 (glob)
- 콘텐츠 패턴 라이브러리 (regex)
- 사용 사례별 정리
- 복사-붙여넣기 준비 완료

### [ADVANCED.md](ADVANCED.md)
향후 개선 사항 및 아이디어:
- 동적 규칙 업데이트
- 스킬 의존성
- 조건부 강제
- 스킬 분석
- 스킬 버전 관리

---

## 빠른 참조 요약

### 새 스킬 생성 (5단계)

1. frontmatter가 있는 `.claude/skills/{name}/SKILL.md` 생성
2. `.claude/skills/skill-rules.json`에 항목 추가
3. `npx tsx` 명령으로 테스트
4. 테스트 결과를 기반으로 패턴 개선
5. SKILL.md를 500줄 미만으로 유지

### 트리거 유형

- **키워드**: 명시적 주제 언급
- **의도**: 암묵적 작업 감지
- **파일 경로**: 위치 기반 활성화
- **콘텐츠**: 기술 특화 감지

자세한 내용은 [TRIGGER_TYPES.md](TRIGGER_TYPES.md) 참조.

### 강제

- **BLOCK**: 종료 코드 2, 중요한 경우에만
- **SUGGEST**: 컨텍스트 주입, 가장 일반적
- **WARN**: 권고 사항, 거의 사용되지 않음

### 건너뛰기 조건

- **세션 추적**: 자동 (반복 알림 방지)
- **파일 마커**: `// @skip-validation` (영구적 건너뛰기)
- **환경 변수**: `SKIP_SKILL_GUARDRAILS` (비상 비활성화)

### Anthropic 모범 사례

- **500줄 규칙**: SKILL.md를 500줄 미만으로 유지
- **점진적 공개**: 세부 사항에 참조 파일 사용
- **목차**: 100줄 이상의 참조 파일에 추가
- **한 단계 깊이**: 참조를 깊게 중첩하지 말 것
- **풍부한 설명**: 모든 트리거 키워드 포함 (최대 1024자)
- **테스트 우선**: 광범위한 문서 작성 전에 3개 이상의 평가 구축
- **동명사 명명**: 동사 + -ing 선호 (예: "processing-pdfs")

### 문제 해결

훅 수동 테스트:
```bash
# UserPromptSubmit
echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

완전한 디버깅 가이드는 [TROUBLESHOOTING.md](TROUBLESHOOTING.md) 참조.

---

## 관련 파일

**설정:**
- `.claude/skills/skill-rules.json` - 마스터 설정
- `.claude/hooks/state/` - 세션 추적
- `.claude/settings.json` - 훅 등록

**훅:**
- `.claude/hooks/skill-activation-prompt.ts` - UserPromptSubmit
- `.claude/hooks/error-handling-reminder.ts` - Stop 이벤트 (부드러운 알림)

**모든 스킬:**
- `.claude/skills/*/SKILL.md` - 스킬 내용 파일

---

**스킬 상태**: 완료 - Anthropic 모범 사례에 따라 재구성
**줄 수**: < 500 (500줄 규칙 준수)
**점진적 공개**: 자세한 정보는 참조 파일 사용

**다음 단계**: 더 많은 스킬 생성, 사용 기반으로 패턴 개선
