---
name: error-tracking
description: 프로젝트 서비스에 Sentry v8 에러 추적 및 성능 모니터링을 추가합니다. 에러 처리 추가, 새 컨트롤러 생성, cron job 계측, 데이터베이스 성능 추적 시 이 스킬을 사용하세요. 모든 에러는 반드시 Sentry에 캡처해야 합니다 - 예외 없음.
---

# 프로젝트 Sentry 통합 스킬

## 목적
이 스킬은 Sentry v8 패턴에 따라 모든 프로젝트 서비스에 걸쳐 포괄적인 Sentry 에러 추적 및 성능 모니터링을 강제합니다.

## 이 스킬을 사용하는 경우
- 임의의 코드에 에러 처리 추가 시
- 새 컨트롤러 또는 라우트 생성 시
- cron job 계측 시
- 데이터베이스 성능 추적 시
- 성능 span 추가 시
- 워크플로우 에러 처리 시

## 중요 규칙

**모든 에러는 반드시 Sentry에 캡처해야 합니다** - 예외 없음. console.error만 단독으로 사용하지 마세요.

## 현재 상태

### Form Service 완료
- Sentry v8 완전 통합
- 모든 워크플로우 에러 추적
- SystemActionQueueProcessor 계측 완료
- 테스트 엔드포인트 사용 가능

### Email Service 진행 중
- Phase 1-2 완료 (6/22 태스크)
- ErrorLogger.log() 호출 189개 남음

## Sentry 통합 패턴

### 1. 컨트롤러 에러 처리

```typescript
// 올바른 방법 - BaseController 사용
import { BaseController } from '../controllers/BaseController';

export class MyController extends BaseController {
    async myMethod() {
        try {
            // ... 코드
        } catch (error) {
            this.handleError(error, 'myMethod'); // 자동으로 Sentry에 전송
        }
    }
}
```

### 2. 라우트 에러 처리 (BaseController 미사용 시)

```typescript
import * as Sentry from '@sentry/node';

router.get('/route', async (req, res) => {
    try {
        // ... 코드
    } catch (error) {
        Sentry.captureException(error, {
            tags: { route: '/route', method: 'GET' },
            extra: { userId: req.user?.id }
        });
        res.status(500).json({ error: 'Internal server error' });
    }
});
```

### 3. 워크플로우 에러 처리

```typescript
import { WorkflowSentryHelper } from '../workflow/utils/sentryHelper';

// 올바른 방법 - WorkflowSentryHelper 사용
WorkflowSentryHelper.captureWorkflowError(error, {
    workflowCode: 'DHS_CLOSEOUT',
    instanceId: 123,
    stepId: 456,
    userId: 'user-123',
    operation: 'stepCompletion',
    metadata: { additionalInfo: 'value' }
});
```

### 4. Cron Jobs (필수 패턴)

```typescript
#!/usr/bin/env node
// shebang 다음 첫 번째 줄 - 매우 중요!
import '../instrument';
import * as Sentry from '@sentry/node';

async function main() {
    return await Sentry.startSpan({
        name: 'cron.job-name',
        op: 'cron',
        attributes: {
            'cron.job': 'job-name',
            'cron.startTime': new Date().toISOString(),
        }
    }, async () => {
        try {
            // cron job 로직
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Job] Error:', error);
            process.exit(1);
        }
    });
}

main()
    .then(() => {
        console.log('[Job] Completed successfully');
        process.exit(0);
    })
    .catch((error) => {
        console.error('[Job] Fatal error:', error);
        process.exit(1);
    });
```

### 5. 데이터베이스 성능 모니터링

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

// 올바른 방법 - 데이터베이스 작업을 래핑
const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({
            take: 5,
        });
    }
);
```

### 6. Span을 사용한 비동기 작업

```typescript
import * as Sentry from '@sentry/node';

const result = await Sentry.startSpan({
    name: 'operation.name',
    op: 'operation.type',
    attributes: {
        'custom.attribute': 'value'
    }
}, async () => {
    // 비동기 작업
    return await someAsyncOperation();
});
```

## 에러 레벨

적절한 심각도 레벨을 사용하세요:

- **fatal**: 시스템 사용 불가 (데이터베이스 다운, 심각한 서비스 장애)
- **error**: 작업 실패, 즉각적인 주의 필요
- **warning**: 복구 가능한 문제, 성능 저하
- **info**: 정보성 메시지, 성공한 작업
- **debug**: 상세 디버깅 정보 (개발 환경 전용)

## 필수 컨텍스트

```typescript
import * as Sentry from '@sentry/node';

Sentry.withScope((scope) => {
    // 가능한 경우 항상 포함
    scope.setUser({ id: userId });
    scope.setTag('service', 'form'); // 또는 'email', 'users' 등
    scope.setTag('environment', process.env.NODE_ENV);

    // 작업별 컨텍스트 추가
    scope.setContext('operation', {
        type: 'workflow.start',
        workflowCode: 'DHS_CLOSEOUT',
        entityId: 123
    });

    Sentry.captureException(error);
});
```

## 서비스별 통합

### Form Service

**위치**: `./blog-api/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**주요 헬퍼**:
- `WorkflowSentryHelper` - 워크플로우 특화 에러
- `DatabasePerformanceMonitor` - DB 쿼리 추적
- `BaseController` - 컨트롤러 에러 처리

### Email Service

**위치**: `./notifications/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**주요 헬퍼**:
- `EmailSentryHelper` - 이메일 특화 에러
- `BaseController` - 컨트롤러 에러 처리

## 설정 (config.ini)

```ini
[sentry]
dsn = your-sentry-dsn
environment = development
tracesSampleRate = 0.1
profilesSampleRate = 0.1

[databaseMonitoring]
enableDbTracing = true
slowQueryThreshold = 100
logDbQueries = false
dbErrorCapture = true
enableN1Detection = true
```

## Sentry 통합 테스트

### Form Service 테스트 엔드포인트

```bash
# 기본 에러 캡처 테스트
curl http://localhost:3002/blog-api/api/sentry/test-error

# 워크플로우 에러 테스트
curl http://localhost:3002/blog-api/api/sentry/test-workflow-error

# 데이터베이스 성능 테스트
curl http://localhost:3002/blog-api/api/sentry/test-database-performance

# 에러 바운더리 테스트
curl http://localhost:3002/blog-api/api/sentry/test-error-boundary
```

### Email Service 테스트 엔드포인트

```bash
# 기본 에러 캡처 테스트
curl http://localhost:3003/notifications/api/sentry/test-error

# 이메일 특화 에러 테스트
curl http://localhost:3003/notifications/api/sentry/test-email-error

# 성능 추적 테스트
curl http://localhost:3003/notifications/api/sentry/test-performance
```

## 성능 모니터링

### 요구사항

1. **모든 API 엔드포인트**에 트랜잭션 추적 필수
2. **100ms 초과 데이터베이스 쿼리**는 자동으로 플래그 처리
3. **N+1 쿼리**는 감지 및 보고
4. **Cron jobs**는 실행 시간 추적 필수

### 트랜잭션 추적

```typescript
import * as Sentry from '@sentry/node';

// Express 라우트용 자동 트랜잭션 추적
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// 커스텀 작업용 수동 트랜잭션
const transaction = Sentry.startTransaction({
    op: 'operation.type',
    name: 'Operation Name',
});

try {
    // 작업
} finally {
    transaction.finish();
}
```

## 피해야 할 일반적인 실수

- Sentry 없이 console.error만 사용하지 말 것
- 에러를 조용히 삼키지 말 것
- 에러 컨텍스트에 민감한 데이터를 노출하지 말 것
- 컨텍스트 없는 일반적인 에러 메시지를 사용하지 말 것
- 비동기 작업에서 에러 처리를 건너뛰지 말 것
- cron job에서 instrument.ts 임포트를 첫 번째 줄에 추가하는 것을 잊지 말 것

## 구현 체크리스트

새 코드에 Sentry 추가 시:

- [ ] Sentry 또는 적절한 헬퍼 임포트
- [ ] 모든 try/catch 블록에서 Sentry에 캡처
- [ ] 에러에 의미 있는 컨텍스트 추가
- [ ] 적절한 에러 레벨 사용
- [ ] 에러 메시지에 민감한 데이터 없음
- [ ] 느린 작업에 성능 추적 추가
- [ ] 에러 처리 경로 테스트
- [ ] cron job의 경우: instrument.ts 첫 번째 임포트

## 주요 파일

### Form Service
- `/blog-api/src/instrument.ts` - Sentry 초기화
- `/blog-api/src/workflow/utils/sentryHelper.ts` - 워크플로우 에러
- `/blog-api/src/utils/databasePerformance.ts` - DB 모니터링
- `/blog-api/src/controllers/BaseController.ts` - 컨트롤러 기반 클래스

### Email Service
- `/notifications/src/instrument.ts` - Sentry 초기화
- `/notifications/src/utils/EmailSentryHelper.ts` - 이메일 에러
- `/notifications/src/controllers/BaseController.ts` - 컨트롤러 기반 클래스

### 설정
- `/blog-api/config.ini` - Form service 설정
- `/notifications/config.ini` - Email service 설정
- `/sentry.ini` - 공유 Sentry 설정

## 문서

- 전체 구현: `/dev/active/email-sentry-integration/`
- Form service 문서: `/blog-api/docs/sentry-integration.md`
- Email service 문서: `/notifications/docs/sentry-integration.md`

## 관련 스킬

- 데이터베이스 작업 전에 **database-verification** 사용
- 워크플로우 에러 컨텍스트에는 **workflow-builder** 사용
- 데이터베이스 에러 처리에는 **database-scripts** 사용
