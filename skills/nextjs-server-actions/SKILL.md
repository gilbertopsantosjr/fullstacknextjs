---
name: nextjs-server-actions
description: Guide for implementing Next.js server actions using ZSA (Zod Server Actions) with authentication, validation, and React Query integration. Use when creating API endpoints, form handlers, mutations, or any server-side data operations.
---

# Next.js Server Actions with ZSA

## File Structure

```
features/<feature>/usecases/
├── create/actions/create-<entity>-action.ts
├── update/actions/update-<entity>-action.ts
├── delete/actions/delete-<entity>-action.ts
└── list/actions/list-<entity>-action.ts
```

## Action Pattern

```typescript
// create-account-action.ts
'use server'

import 'server-only'
import { revalidatePath } from 'next/cache'
import { authedProcedure } from '@saas4dev/auth'
import { CreateAccountSchema, AccountSchema } from '@/features/accounts/model/account-schemas'
import { AccountService } from '@/features/accounts/account-service'

export const createAccountAction = authedProcedure
  .createServerAction()
  .input(CreateAccountSchema, { type: 'formData' })
  .output(AccountSchema)
  .onComplete(async () => {
    revalidatePath('/accounts')
  })
  .handler(async ({ input, ctx }) => {
    const result = await AccountService.create(ctx.userId, input)
    if (!result.success) {
      throw new Error(result.error)
    }
    return result.data
  })
```

## Required Directives

Every action file MUST include:
```typescript
'use server'           // First line - marks as server action
import 'server-only'   // Prevents client import
```

## Authentication

Use `authedProcedure` for protected actions:
```typescript
import { authedProcedure } from '@saas4dev/auth'

export const myAction = authedProcedure
  .createServerAction()
  .handler(async ({ ctx }) => {
    const userId = ctx.userId  // Available from auth context
  })
```

For public actions (no auth):
```typescript
import { createServerAction } from 'zsa'

export const publicAction = createServerAction()
  .input(schema)
  .handler(async ({ input }) => { ... })
```

## Client Integration with React Query

```typescript
'use client'
import { useServerActionMutation, useServerActionQuery } from '@saas4dev/core'

// Mutations (create, update, delete)
const mutation = useServerActionMutation(createAccountAction, {
  onSuccess: () => toast.success('Created'),
  onError: (error) => toast.error(error.message),
})

mutation.mutate(formData)

// Queries (read, list)
const { data, isLoading } = useServerActionQuery(listAccountsAction, {
  input: { userId },
})
```

## Form Submission

```typescript
'use client'
import { useForm } from 'react-hook-form'
import { useServerActionMutation } from '@saas4dev/core'

export function CreateForm() {
  const form = useForm<Input>({ resolver: zodResolver(Schema) })
  const mutation = useServerActionMutation(createAction)

  const onSubmit = (data: Input) => mutation.mutate(data)

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* fields */}
      <Button disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </Button>
    </form>
  )
}
```

## Error Handling

```typescript
.handler(async ({ input, ctx }) => {
  try {
    const result = await Service.create(ctx.userId, input)
    if (!result.success) {
      throw new Error(result.error)
    }
    return result.data
  } catch (error) {
    // ZSA automatically handles errors
    throw error
  }
})
```

## Rules

- Call Service layer, NOT DAL directly
- Always validate input with Zod schema
- Use `revalidatePath` or `revalidateTag` after mutations
- Never expose internal errors to client
- Keep actions thin - business logic goes in services
