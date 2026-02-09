---
name: santry-observability
description: Sentry observability for Clean Architecture layers. Error tracking per layer, transaction tracing for Use Cases, user context in procedures, and performance monitoring.
---

# Sentry Observability (Clean Architecture)

## Layer-Specific Logging

| Layer | What to Log | Sentry Feature |
|-------|-------------|----------------|
| Domain | Entity validation failures | Exception with tags |
| Application | Use Case execution, timing | Transaction spans |
| Infrastructure | Repository operations, DB timing | Child spans |
| Presentation | Action errors, user context | Breadcrumbs + user |

## Installation

```bash
npm install @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

## Configuration

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
})
```

## Use Case Tracing

Wrap Use Case execution with transaction spans.

```typescript
// src/backend/application/category/use-cases/create-category.ts
import * as Sentry from '@sentry/nextjs'

export class CreateCategoryUseCase {
  constructor(private readonly repository: ICategoryRepository) {}

  async execute(input: CreateCategoryInput): Promise<CategoryDTO> {
    return Sentry.startSpan(
      {
        name: 'CreateCategoryUseCase.execute',
        op: 'usecase.execute',
        attributes: { userId: input.userId },
      },
      async () => {
        // Domain operation
        const category = Sentry.startSpan(
          { name: 'Category.create', op: 'domain.entity' },
          () => Category.create({ id: crypto.randomUUID(), ...input })
        )

        // Repository operation
        await Sentry.startSpan(
          { name: 'repository.save', op: 'db.write' },
          () => this.repository.save(category)
        )

        return CategoryDTO.fromEntity(category)
      }
    )
  }
}
```

## Repository Timing

Log database operations with timing.

```typescript
// src/backend/infrastructure/category/repositories/category-repository-impl.ts
import * as Sentry from '@sentry/nextjs'

export class CategoryRepositoryImpl implements ICategoryRepository {
  async save(category: Category): Promise<void> {
    return Sentry.startSpan(
      {
        name: 'CategoryRepository.save',
        op: 'db.dynamodb',
        attributes: { categoryId: category.id },
      },
      async () => {
        await this.model.create(category.toPersistence())
      }
    )
  }

  async findById(id: string, userId: string): Promise<Category | null> {
    return Sentry.startSpan(
      {
        name: 'CategoryRepository.findById',
        op: 'db.dynamodb',
        attributes: { categoryId: id },
      },
      async () => {
        const data = await this.model.get({ pk: userId, sk: id })
        return data ? Category.fromPersistence(data) : null
      }
    )
  }
}
```

## Server Action Error Handling

Capture domain exceptions with context.

```typescript
// src/features/category/actions/create-category.ts
'use server'
import * as Sentry from '@sentry/nextjs'
import { authedProcedure } from '@/lib/procedures'
import { DIContainer, TOKENS } from '@/backend/di'
import { DomainException } from '@/backend/domain/shared/exceptions'

export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)
  .handler(async ({ input, ctx }) => {
    try {
      const useCase = DIContainer.resolve(TOKENS.CreateCategoryUseCase)
      return await useCase.execute({ ...input, userId: ctx.user.id })
    } catch (error) {
      if (error instanceof DomainException) {
        Sentry.captureException(error, {
          tags: {
            action: 'createCategory',
            layer: 'domain',
            code: error.code,
          },
          contexts: {
            input: { name: input.name },
            user: { id: ctx.user.id },
          },
        })
      }
      throw error
    }
  })
```

## Procedure with User Context

Set user context early in the procedure chain.

```typescript
// src/lib/procedures.ts
import * as Sentry from '@sentry/nextjs'
import { createServerActionProcedure } from 'zsa'

export const authedProcedure = createServerActionProcedure()
  .handler(async () => {
    const session = await auth()

    if (!session?.user) {
      Sentry.captureMessage('Unauthenticated access attempt', {
        level: 'warning',
      })
      throw new Error('Not authenticated')
    }

    // Set user context for all subsequent errors
    Sentry.setUser({
      id: session.user.id,
      email: session.user.email,
    })

    Sentry.addBreadcrumb({
      category: 'auth',
      message: 'User authenticated',
      level: 'info',
    })

    return { user: session.user }
  })
```

## Exception Hierarchy Mapping

Map domain exceptions to Sentry severity levels.

```typescript
// src/backend/domain/shared/exceptions/sentry-mapper.ts
import * as Sentry from '@sentry/nextjs'
import { DomainException, NotFoundException, ValidationException } from './index'

export function captureToSentry(error: unknown, context: Record<string, unknown>) {
  if (error instanceof ValidationException) {
    Sentry.captureException(error, {
      level: 'warning',
      tags: { layer: 'domain', type: 'validation' },
      contexts: { validation: context },
    })
  } else if (error instanceof NotFoundException) {
    Sentry.captureException(error, {
      level: 'info',
      tags: { layer: 'domain', type: 'not_found' },
      contexts: { query: context },
    })
  } else if (error instanceof DomainException) {
    Sentry.captureException(error, {
      level: 'error',
      tags: { layer: 'domain', type: 'business_rule' },
      contexts: { domain: context },
    })
  } else {
    Sentry.captureException(error, {
      level: 'error',
      tags: { layer: 'unknown' },
      contexts: { raw: context },
    })
  }
}
```

## Client Error Boundary

```typescript
// src/components/error-boundary.tsx
'use client'
import * as Sentry from '@sentry/nextjs'
import { useEffect } from 'react'

export function ActionErrorBoundary({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    Sentry.captureException(error, {
      tags: { component: 'ActionErrorBoundary' },
    })
  }, [error])

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

## Best Practices

1. **Set user context in procedures** - Not in individual actions
2. **Use spans for Use Cases** - Track execution time
3. **Add breadcrumbs for flow** - Auth, validation, processing
4. **Tag by layer** - domain, application, infrastructure, presentation
5. **Map exceptions to severity** - validation=warning, not_found=info, business=error
6. **Scrub sensitive data** - Never log passwords, tokens

## Environment Variables

```bash
NEXT_PUBLIC_SENTRY_DSN=your-client-dsn
SENTRY_DSN=your-server-dsn
SENTRY_AUTH_TOKEN=your-auth-token
SENTRY_ORG=your-org
SENTRY_PROJECT=your-project
```

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- Server Actions: `skills/nextjs-server-actions/SKILL.md`
