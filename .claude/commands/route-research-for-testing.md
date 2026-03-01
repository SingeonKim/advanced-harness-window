---
description: 수정된 라우트 매핑 및 테스트 실행
argument-hint: "[/extra/path …]"
allowed-tools: Bash(cat:*), Bash(awk:*), Bash(grep:*), Bash(sort:*), Bash(xargs:*), Bash(sed:*)
model: sonnet
---

## 컨텍스트

이번 세션에서 변경된 라우트 파일 (자동 생성):

!cat "$CLAUDE_PROJECT_DIR/.claude/tsc-cache"/\*/edited-files.log \
 | awk -F: '{print $2}' \
 | grep '/routes/' \
 | sort -u

사용자가 지정한 추가 라우트: `$ARGUMENTS`

## 태스크

번호가 매겨진 단계를 **정확히** 따라 실행하세요:

1. 자동 목록과 `$ARGUMENTS`를 결합하고, 중복을 제거하며, `src/app.ts`에 정의된 접두사를 해결합니다.
2. 각 최종 라우트에 대해 경로, 메서드, 예상 요청/응답 형식, 유효 및 유효하지 않은 페이로드 예시를 포함한 JSON 레코드를 출력합니다.
3. **이제 `Task` 도구를 호출합니다**:

```json
{
    "tool": "Task",
    "parameters": {
        "description": "라우트 스모크 테스트",
        "prompt": "위의 JSON으로 auth-route-tester 서브에이전트를 실행하세요."
    }
}
```
