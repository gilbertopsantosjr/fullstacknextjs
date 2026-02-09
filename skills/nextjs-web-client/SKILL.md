---
name: nextjs-web-client
description: Guide for building Next.js 15+ React 19+ frontend components with Clean Architecture. Components receive DTOs from server actions, never Entities directly. Use when creating UI components, pages, layouts, forms, or client-side interactivity.
---

# Next.js Web Client Development

## Clean Architecture Data Flow

Components receive **DTOs** from server actions, never Entities:

```
Component → Server Action → Use Case → Repository → DB
                                ↓
                              DTO (returned)
```

## Importing from Features

```typescript
// ✅ CORRECT: Import DTOs and actions
import type { CategoryDTO } from '@/features/category'
import { listCategoriesAction, createCategoryAction } from '@/features/category'

// ❌ WRONG: Never import from backend directly
import { Category } from '@/backend/domain/category/entities' // VIOLATION!
import { CreateCategoryUseCase } from '@/backend/application/category' // VIOLATION!
```

## Server vs Client Components

**Default to Server Components** unless you need interactivity:

```
components/
├── CategoryList.tsx           # Server component (default)
├── CategoryForm.tsx           # Re-exports server
├── CategoryForm-server.tsx    # Calls action, passes DTO
└── CategoryForm-client.tsx    # 'use client', handles form
```

### Server Component with Action

```typescript
// CategoryList.tsx (Server Component)
import type { CategoryDTO } from '@/features/category'
import { listCategoriesAction } from '@/features/category'

export async function CategoryList() {
  const [result, err] = await listCategoriesAction({})

  if (err) return <div>Error loading categories</div>

  return (
    <ul>
      {result.items.map((category: CategoryDTO) => (
        <li key={category.id}>{category.name}</li>
      ))}
    </ul>
  )
}
```

### Client Component with Mutation

```typescript
// CategoryForm-client.tsx
'use client'
import { useServerAction } from 'zsa-react'
import { createCategoryAction } from '@/features/category'
import type { CreateCategoryInput } from '@/features/category'

export function CategoryFormClient() {
  const { execute, isPending, error } = useServerAction(createCategoryAction)

  const onSubmit = async (data: CreateCategoryInput) => {
    const [result, err] = await execute(data)
    if (err) return toast.error(err.message)
    toast.success('Created!')
  }

  return <form onSubmit={handleSubmit(onSubmit)}>...</form>
}
```

## Forms with React Hook Form

```typescript
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { CreateCategorySchema, type CreateCategoryInput } from '@/features/category'
import { createCategoryAction } from '@/features/category'

export function CreateCategoryForm() {
  const form = useForm<CreateCategoryInput>({
    resolver: zodResolver(CreateCategorySchema),
    defaultValues: { name: '' },
  })

  const { execute, isPending } = useServerAction(createCategoryAction)

  const onSubmit = async (data: CreateCategoryInput) => {
    const [result, err] = await execute(data)
    if (err) return form.setError('root', { message: err.message })
    // Success handling
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField name="name" ... />
        <Button type="submit" disabled={isPending}>Create</Button>
      </form>
    </Form>
  )
}
```

## Data Tables with DTOs

```typescript
import type { CategoryDTO } from '@/features/category'

export function CategoryTable({ items }: { items: CategoryDTO[] }) {
  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Name</TableHead>
          <TableHead>Status</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {items.map((category) => (
          <TableRow key={category.id}>
            <TableCell>{category.name}</TableCell>
            <TableCell>{category.status}</TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  )
}
```

## Component Naming

| Type | Case | Example |
|------|------|---------|
| Components | PascalCase | `CategoryCard.tsx` |
| Client suffix | -client | `CategoryForm-client.tsx` |
| Server suffix | -server | `CategoryForm-server.tsx` |
| Hooks | use-kebab | `use-category-form.ts` |
| Pages | kebab-case | `app/(dashboard)/categories/page.tsx` |

## Rules for Components

1. **Import DTOs** from feature index, never Entities
2. **Call actions** for data operations, never Use Cases directly
3. **Type with DTOs** - `items: CategoryDTO[]` not `items: Category[]`
4. **Handle errors** from action `[result, err]` tuple
5. **No backend imports** - features are the boundary

## Detection Commands

```bash
# Components importing from backend (VIOLATION)
grep -rn "from '@/backend/" src/features/*/components/
grep -rn "from '@/backend/" src/app/

# Components importing Entities (VIOLATION)
grep -rn "import.*Entity" src/features/*/components/
grep -rn "import.*Entity" src/app/
```

## References

- Server Actions: `skills/nextjs-server-actions/SKILL.md`
- React Query: `skills/tanstack-react-query/SKILL.md`
