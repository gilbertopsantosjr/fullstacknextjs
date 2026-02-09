---
name: nextjs-server-actions
description: "Guide for implementing thin adapter server actions using ZSA. Actions resolve Use Cases from DI Container and delegate business logic. Use when creating API endpoints in the Presentation layer."
---

# Next.js Server Actions (Thin Adapters)

Server actions are **thin adapters** in Clean Architecture. They:
- Resolve Use Cases from DI Container
- Pass input to Use Case's `execute()` method
- Return DTOs (not Entities)

## Thin Adapter Pattern (3-5 lines)

```typescript
// src/features/category/actions/create-category-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { CreateCategorySchema } from '../schemas/category-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { CreateCategoryUseCase } from '@/backend/application/category/use-cases'

export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<CreateCategoryUseCase>(TOKENS.CreateCategoryUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

## File Structure

```
src/features/<feature>/
├── actions/
│   ├── create-<entity>-action.ts
│   ├── update-<entity>-action.ts
│   ├── delete-<entity>-action.ts
│   ├── get-<entity>-action.ts
│   ├── list-<entity>s-action.ts
│   └── index.ts
└── schemas/
    └── <entity>-schemas.ts
```

## Required Directives

Every action file MUST include:
```typescript
'use server'           // First line - marks as server action
import 'server-only'   // Prevents client import
```

## Procedures

```typescript
// lib/procedures.ts
'use server'
import { createServerActionProcedure } from 'zsa'
import { auth } from '@/lib/auth'

export const authedProcedure = createServerActionProcedure()
  .handler(async () => {
    const session = await auth()
    if (!session?.user) throw new Error('Not authenticated')
    return { user: { id: session.user.id, email: session.user.email } }
  })

export const publicProcedure = createServerActionProcedure()
  .handler(async () => ({}))
```

## Action Patterns

### Query Action
```typescript
export const getCategoryAction = authedProcedure
  .createServerAction()
  .input(z.object({ id: z.string().ulid() }))
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<GetCategoryUseCase>(TOKENS.GetCategoryUseCase)
    return useCase.execute({ id: input.id, userId: ctx.user.id })
  })
```

### List Action with Pagination
```typescript
export const listCategoriesAction = authedProcedure
  .createServerAction()
  .input(z.object({
    limit: z.coerce.number().min(1).max(100).optional(),
    cursor: z.string().optional(),
  }))
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<ListCategoriesUseCase>(TOKENS.ListCategoriesUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

### Mutation with Revalidation
```typescript
export const updateCategoryAction = authedProcedure
  .createServerAction()
  .input(UpdateCategorySchema)
  .onComplete(async () => {
    revalidatePath('/categories')
  })
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<UpdateCategoryUseCase>(TOKENS.UpdateCategoryUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

## Calling Actions

### From Client
```typescript
'use client'
import { useServerAction } from 'zsa-react'
import { createCategoryAction } from '@/features/category'

export function CreateForm() {
  const { isPending, execute, error, isSuccess } = useServerAction(createCategoryAction)

  const handleSubmit = async (formData: FormData) => {
    const [data, err] = await execute({ name: formData.get('name') as string })
    if (err) return console.error(err.message)
    // Success
  }

  return <form action={handleSubmit}>...</form>
}
```

### From Server
```typescript
const [data, err] = await createCategoryAction({ name: 'New Category' })
if (err) console.error(err.code, err.message)
```

## Error Handling

Use Cases throw domain exceptions, which propagate to the client:

```typescript
// Use Case throws
throw new CategoryNotFoundException(input.id)

// Client receives
const [data, err] = await execute(input)
if (err) {
  // err.message = "Category with id 01HX... not found"
  // err.code = "ERROR"
}
```

## Anti-Patterns

### ❌ Fat Actions (Business Logic in Action)
```typescript
// BAD - 50+ lines with business logic
export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)
  .handler(async ({ input, ctx }) => {
    // Validation logic
    // Permission checks
    // Database operations
    // More business rules
  })
```

### ❌ Direct Repository Access
```typescript
// BAD - Bypasses Use Case
export const createCategoryAction = authedProcedure
  .handler(async ({ input, ctx }) => {
    const repo = DIContainer.resolve<ICategoryRepository>(TOKENS.CategoryRepository)
    const entity = Category.create({ ...input, userId: ctx.user.id })
    await repo.save(entity) // Direct repo access!
  })
```

### ❌ Direct Instantiation
```typescript
// BAD - Creates dependencies directly
export const createCategoryAction = authedProcedure
  .handler(async ({ input, ctx }) => {
    const repo = new DynamoDBCategoryRepository() // VIOLATION!
    const useCase = new CreateCategoryUseCase(repo) // VIOLATION!
    return useCase.execute(input)
  })
```

## Detection Commands

```bash
# Fat actions (direct DB access)
grep -rn "getDynamoDbTable\|getModel" src/features/*/actions/

# Direct instantiation
grep -rn "new.*UseCase(\|new.*Repository(" src/features/

# Action file sizes (should be <30 lines)
find src/features/*/actions -name "*.ts" ! -name "index.ts" -exec wc -l {} \;
```

## Best Practices

1. **3-5 lines in handler** - Resolve Use Case, execute, return
2. **DI Container** for all Use Case resolution
3. **Zod for input shape only** - Business rules in Entity
4. **Use `revalidatePath`/`revalidateTag`** after mutations
5. **Let Use Cases handle errors** - Domain exceptions propagate naturally

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- Zod Validation: `skills/zod-validation/SKILL.md`
- React Query: `skills/tanstack-react-query/SKILL.md`
