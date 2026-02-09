---
name: sst-infra
description: Guide for AWS serverless infrastructure using SST v3. Covers DynamoDB, Next.js deployment, Lambda handlers with Clean Architecture adapter pattern, and CI/CD configuration.
---

# SST v3 Infrastructure

## Project Structure

```
project/
├── sst.config.ts              # Main SST config
├── stacks/
│   ├── dynamodb.ts            # Database stack
│   ├── nextjs.ts              # Next.js deployment
│   └── api.ts                 # Lambda API stack
├── open-next.config.ts        # Lambda streaming config
└── src/
    └── backend/               # Clean Architecture backend
```

## Main Config

```typescript
// sst.config.ts
export default $config({
  app(input) {
    return {
      name: 'my-app',
      removal: input?.stage === 'prod' ? 'retain' : 'remove',
      protect: ['prod'].includes(input?.stage ?? ''),
      home: 'aws',
      providers: { aws: { region: 'us-east-1' } },
    }
  },
  async run() {
    const { table } = await import('./stacks/dynamodb')
    const { site } = await import('./stacks/nextjs')
    return { url: site.url, tableName: table.name }
  },
})
```

## DynamoDB Stack

```typescript
// stacks/dynamodb.ts
export const table = new sst.aws.Dynamo('Table', {
  fields: {
    pk: 'string', sk: 'string',
    gsi1pk: 'string', gsi1sk: 'string',
  },
  primaryIndex: { hashKey: 'pk', rangeKey: 'sk' },
  globalIndexes: {
    gsi1: { hashKey: 'gsi1pk', rangeKey: 'gsi1sk' },
  },
  transform: {
    table: (args) => { args.billingMode = 'PAY_PER_REQUEST' },
  },
})
```

## Next.js Stack

```typescript
// stacks/nextjs.ts
import { table } from './dynamodb'

export const site = new sst.aws.Nextjs('Site', {
  path: 'apps/web',
  link: [table],
  environment: { TABLE_NAME: table.name },
  domain: {
    name: `${$app.stage === 'prod' ? '' : `${$app.stage}.`}myapp.com`,
    dns: sst.aws.dns({ zone: 'myapp.com' }),
  },
})
```

## Lambda Handler with Clean Architecture

Lambda handlers follow the **thin adapter pattern** - resolve Use Case from DI and execute.

```typescript
// src/functions/create-category.ts
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda'
import { DIContainer, TOKENS, initializeDI } from '@/backend/di'
import { CreateCategoryUseCase } from '@/backend/application/category/use-cases'
import { DomainException } from '@/backend/domain/shared/exceptions'

// Initialize DI once per cold start
let initialized = false

export async function handler(
  event: APIGatewayProxyEventV2
): Promise<APIGatewayProxyResultV2> {
  // Initialize DI Container (cold start only)
  if (!initialized) {
    await initializeDI()
    initialized = true
  }

  try {
    const input = JSON.parse(event.body ?? '{}')

    // Thin adapter: resolve → execute → return
    const useCase = DIContainer.resolve<CreateCategoryUseCase>(
      TOKENS.CreateCategoryUseCase
    )
    const result = await useCase.execute(input)

    return {
      statusCode: 201,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(result),
    }
  } catch (error) {
    return mapErrorToResponse(error)
  }
}

function mapErrorToResponse(error: unknown): APIGatewayProxyResultV2 {
  if (error instanceof DomainException) {
    return {
      statusCode: 400,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: error.message, code: error.code }),
    }
  }

  console.error('Unhandled error:', error)
  return {
    statusCode: 500,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ error: 'Internal server error' }),
  }
}
```

## Lambda API Stack

```typescript
// stacks/api.ts
import { table } from './dynamodb'

const api = new sst.aws.ApiGatewayV2('Api')

api.route('POST /categories', {
  handler: 'src/functions/create-category.handler',
  link: [table],
  environment: { TABLE_NAME: table.name },
})

api.route('GET /categories/{id}', {
  handler: 'src/functions/get-category.handler',
  link: [table],
  environment: { TABLE_NAME: table.name },
})

export { api }
```

## OpenNext Config

```typescript
// open-next.config.ts
import type { OpenNextConfig } from 'open-next/types/open-next'

const config: OpenNextConfig = {
  default: {
    override: { wrapper: 'aws-lambda-streaming' },
  },
}
export default config
```

## Commands

```bash
# Development
npx sst dev --stage dev

# Deploy
npx sst deploy --stage dev
npx sst deploy --stage prod

# Secrets
npx sst secret set AUTH_SECRET "value" --stage dev
npx sst secret list --stage dev

# Outputs
npx sst outputs --stage dev
```

## Environment Stages

| Stage | Domain | Protection | Removal |
|-------|--------|------------|---------|
| dev | dev.app.com | No | Remove |
| test | test.app.com | No | Remove |
| prod | app.com | Yes | Retain |

## CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      stage:
        type: choice
        options: [dev, test, prod]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: pnpm install --frozen-lockfile
      - run: npx sst deploy --stage ${{ inputs.stage }}
```

## Local Development

```bash
# DynamoDB Local
docker run -p 8000:8000 amazon/dynamodb-local

# Environment
TABLE_NAME=dev-Table
DYNAMODB_LOCAL=true
```

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- Bun Lambda: `skills/bun-aws-lambda/SKILL.md`
