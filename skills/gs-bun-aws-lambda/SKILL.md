---
name: gs-bun-aws-lambda
description: Bun AWS Lambda handlers with Clean Architecture adapter pattern. Handlers resolve Use Cases from DI Container and map domain exceptions to HTTP responses. Covers deployment patterns and cold start optimization.
---

# Bun AWS Lambda (Clean Architecture)

## Handler Pattern - Thin Adapter

Lambda handlers are thin adapters that:
1. Initialize DI Container (cold start)
2. Parse input
3. Resolve Use Case from DI
4. Execute and return result
5. Map domain exceptions to HTTP status

```typescript
// src/functions/create-category.ts
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda'
import { DIContainer, TOKENS, initializeDI } from '@/backend/di'
import type { CreateCategoryUseCase } from '@/backend/application/category/use-cases'
import { DomainException, NotFoundException } from '@/backend/domain/shared/exceptions'

let initialized = false

export async function handler(
  event: APIGatewayProxyEventV2
): Promise<APIGatewayProxyResultV2> {
  if (!initialized) {
    await initializeDI()
    initialized = true
  }

  try {
    const input = JSON.parse(event.body ?? '{}')
    const userId = event.requestContext.authorizer?.jwt?.claims?.sub

    const useCase = DIContainer.resolve<CreateCategoryUseCase>(
      TOKENS.CreateCategoryUseCase
    )
    const result = await useCase.execute({ ...input, userId })

    return response(201, result)
  } catch (error) {
    return handleError(error)
  }
}

function response(status: number, data: unknown): APIGatewayProxyResultV2 {
  return {
    statusCode: status,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  }
}

function handleError(error: unknown): APIGatewayProxyResultV2 {
  if (error instanceof NotFoundException) {
    return response(404, { error: error.message, code: error.code })
  }
  if (error instanceof DomainException) {
    return response(400, { error: error.message, code: error.code })
  }
  console.error('Unhandled:', error)
  return response(500, { error: 'Internal server error' })
}
```

## Event Source Types

```
├── HTTP API (API Gateway v2) → APIGatewayProxyEventV2
├── REST API (API Gateway v1) → APIGatewayProxyEvent
├── SQS → SQSEvent
├── SNS → SNSEvent
├── EventBridge → EventBridgeEvent<T>
├── S3 → S3Event
└── DynamoDB Streams → DynamoDBStreamEvent
```

## Deployment: Container Image

### Dockerfile

```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile --production
COPY src/ ./src/
RUN bun build src/handler.ts --outdir=dist --target=bun --minify

FROM public.ecr.aws/lambda/provided:al2023
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"
WORKDIR ${LAMBDA_TASK_ROOT}
COPY --from=builder /app/dist/ ./
COPY bootstrap ${LAMBDA_RUNTIME_DIR}/bootstrap
RUN chmod +x ${LAMBDA_RUNTIME_DIR}/bootstrap
CMD ["handler.handler"]
```

### Bootstrap

```typescript
// bootstrap.ts
const RUNTIME_API = `http://${Bun.env.AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime`

const [moduleName, functionName] = (Bun.env._HANDLER ?? 'handler.handler').split('.')
const handlerModule = await import(`./${moduleName}.js`)
const handler = handlerModule[functionName]

while (true) {
  const next = await fetch(`${RUNTIME_API}/invocation/next`)
  const requestId = next.headers.get('Lambda-Runtime-Aws-Request-Id')!
  const event = await next.json()

  try {
    const result = await handler(event, { awsRequestId: requestId })
    await fetch(`${RUNTIME_API}/invocation/${requestId}/response`, {
      method: 'POST',
      body: JSON.stringify(result),
    })
  } catch (error) {
    await fetch(`${RUNTIME_API}/invocation/${requestId}/error`, {
      method: 'POST',
      body: JSON.stringify({ errorMessage: String(error) }),
    })
  }
}
```

## DI Initialization

```typescript
// src/backend/di/initialize.ts
import { DIContainer, TOKENS } from './container'
import { CategoryRepositoryImpl } from '@/backend/infrastructure/category/repositories'
import { CreateCategoryUseCase, GetCategoryUseCase } from '@/backend/application/category/use-cases'

export async function initializeDI() {
  const table = await getTable() // OneTable instance

  // Register repositories
  DIContainer.register(TOKENS.CategoryRepository, {
    useFactory: () => new CategoryRepositoryImpl(table),
  })

  // Register use cases
  DIContainer.register(TOKENS.CreateCategoryUseCase, {
    useFactory: () => new CreateCategoryUseCase(
      DIContainer.resolve(TOKENS.CategoryRepository)
    ),
  })

  DIContainer.register(TOKENS.GetCategoryUseCase, {
    useFactory: () => new GetCategoryUseCase(
      DIContainer.resolve(TOKENS.CategoryRepository)
    ),
  })
}
```

## Cold Start Optimization

1. **Lazy DI init** - Initialize container on first request only
2. **Bundle with Bun** - Single file, tree-shaken
3. **AWS SDK v3** - Modular imports
4. **Minimal deps** - Use native fetch, Bun APIs

```typescript
// Lazy repository initialization
let repository: ICategoryRepository | null = null

function getRepository(): ICategoryRepository {
  if (!repository) {
    repository = DIContainer.resolve(TOKENS.CategoryRepository)
  }
  return repository
}
```

## SQS Handler Example

```typescript
// src/functions/process-queue.ts
import type { SQSEvent, SQSBatchResponse } from 'aws-lambda'
import { DIContainer, TOKENS, initializeDI } from '@/backend/di'
import type { ProcessMessageUseCase } from '@/backend/application/messaging/use-cases'

let initialized = false

export async function handler(event: SQSEvent): Promise<SQSBatchResponse> {
  if (!initialized) {
    await initializeDI()
    initialized = true
  }

  const useCase = DIContainer.resolve<ProcessMessageUseCase>(
    TOKENS.ProcessMessageUseCase
  )

  const failures: SQSBatchResponse['batchItemFailures'] = []

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body)
      await useCase.execute(message)
    } catch (error) {
      console.error(`Failed record ${record.messageId}:`, error)
      failures.push({ itemIdentifier: record.messageId })
    }
  }

  return { batchItemFailures: failures }
}
```

## Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|-----------------|
| Business logic in handler | Use Case classes |
| Direct DB access in handler | Repository via DI |
| `new Repository()` in handler | DI Container resolution |
| Generic error responses | Map domain exceptions |

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- SST Infrastructure: `skills/sst-infra/SKILL.md`
