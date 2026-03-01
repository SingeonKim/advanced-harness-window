---
name: web-design-guidelines
description: Review UI code for Web Interface Guidelines compliance. Use when asked to "review my UI", "check accessibility", "audit design", "review UX", or "check my site against best practices".
metadata:
  author: vercel
  version: "1.0.0"
  argument-hint: <file-or-pattern>
---

# Web Interface 가이드라인

파일이 Web Interface 가이드라인을 준수하는지 검토합니다.

## 작동 방식

1. 아래 소스 URL에서 최신 가이드라인 가져오기
2. 지정된 파일 읽기 (또는 파일/패턴에 대해 사용자에게 확인)
3. 가져온 가이드라인의 모든 규칙에 대해 확인
4. 간결한 `file:line` 형식으로 결과 출력

## 가이드라인 소스

각 검토 전에 최신 가이드라인 가져오기:

```
https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md
```

WebFetch를 사용하여 최신 규칙을 가져오세요. 가져온 내용에는 모든 규칙과 출력 형식 지침이 포함됩니다.

## 사용법

사용자가 파일 또는 패턴 인수를 제공할 때:
1. 위 소스 URL에서 가이드라인 가져오기
2. 지정된 파일 읽기
3. 가져온 가이드라인의 모든 규칙 적용
4. 가이드라인에 지정된 형식으로 결과 출력

파일이 지정되지 않은 경우 사용자에게 어떤 파일을 검토할지 확인합니다.
