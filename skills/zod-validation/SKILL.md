---
name: zod-validation
description: "Guide for Zod validation schemas in Clean Architecture. Zod validates input SHAPE in Presentation layer; business rules belong in Entity.validate(). Use when creating input schemas for server actions."
---

# Zod Validation (Presentation Layer Only)

## Where Validation Belongs

| Validation Type | Location | Example |
|-----------------|----------|---------|
| **Input Shape** | Zod Schema (Presentation) | Required fields, string/number types |
| **Format** | Zod Schema | Email format, ULID format, URL |
| **Basic Constraints** | Zod Schema | min/max length, positive numbers |
| **Business Rules** | Entity.validate() (Domain) | Name uniqueness, status transitions |
| **Complex Rules** | Use Case (Application) | Cross-entity validation |

## Schema Location

```
src/features/<feature>/schemas/
└── <entity>-schemas.ts
```

## Input Schemas (Shape Only)

```typescript
// src/features/category/schemas/category-schemas.ts
import { z } from 'zod'

// Shape validation - NOT business rules
export const CreateCategorySchema = z.object({
  name: z.string().min(1, 'Name is required'),
  description: z.string().optional(),
})

export const UpdateCategorySchema = z.object({
  id: z.string().ulid(),
  name: z.string().min(1).optional(),
  description: z.string().optional(),
})

export const GetCategorySchema = z.object({
  id: z.string().ulid(),
})

export const ListCategoriesSchema = z.object({
  limit: z.coerce.number().min(1).max(100).optional(),
  cursor: z.string().optional(),
  status: z.enum(['active', 'inactive', 'archived']).optional(),
})

export type CreateCategoryInput = z.infer<typeof CreateCategorySchema>
export type UpdateCategoryInput = z.infer<typeof UpdateCategorySchema>
```

## Common Validators

```typescript
// ID formats
z.string().ulid()
z.string().uuid()

// Strings
z.string().min(1, 'Required')
z.string().max(255)
z.string().email()
z.string().url()
z.string().trim()

// Numbers
z.coerce.number()           // String to number
z.number().positive()
z.number().min(0).max(100)

// Optional/Nullable
z.string().optional()       // string | undefined
z.string().nullable()       // string | null

// Enums
z.enum(['active', 'inactive', 'archived'])

// Arrays
z.array(z.string()).min(1)
```

## Business Rules in Entity (NOT Zod)

### ❌ Bad: Business Rules in Zod
```typescript
export const CreateCategorySchema = z.object({
  name: z.string()
    .refine(name => !name.includes('banned'), 'Contains banned words')  // Business rule!
    .refine(name => !name.startsWith('_'), 'Invalid format'),           // Business rule!
})
```

### ✅ Good: Shape in Zod, Rules in Entity
```typescript
// Zod - shape only
export const CreateCategorySchema = z.object({
  name: z.string().min(1),
})

// Entity - business rules
export class Category {
  private validate(): void {
    if (this.props.name.includes('banned')) {
      throw new CategoryValidationException('Name contains banned words')
    }
    if (this.props.name.startsWith('_')) {
      throw new CategoryValidationException('Name cannot start with underscore')
    }
  }
}
```

## Zod in Server Actions

```typescript
'use server'
import { authedProcedure } from '@/lib/zsa'
import { CreateCategorySchema } from '../schemas/category-schemas'

export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)  // Validates shape before handler
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<CreateCategoryUseCase>(TOKENS.CreateCategoryUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

## Validation Error Handling

```typescript
// Client side
const [data, err] = await execute({ name: '' })
if (err) {
  // err.name = "ZodError"
  // err.fieldErrors = { name: ["Name is required"] }
}
```

## Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|------------------|
| Business rules in Zod (.refine) | Entity.validate() |
| Async validation in Zod | Use Case |
| Database checks in Zod | Repository |

## Detection Commands

```bash
# Business logic in schemas
grep -rn "refine\|superRefine" src/features/*/schemas/

# Complex validation (should be in Entity)
grep -rn "banned\|forbidden\|unique\|exists" src/features/*/schemas/
```

## Summary

| Layer | What to Validate |
|-------|------------------|
| **Presentation (Zod)** | Shape, format, basic constraints |
| **Domain (Entity)** | Business rules, invariants |
| **Application (Use Case)** | Cross-entity rules |

## References

- Server Actions: `skills/nextjs-server-actions/SKILL.md`
- Create Domain Module: `skills/create-domain-module/SKILL.md`
