# Advanced Patterns Reference

## Table of Contents
- [Configuration Best Practices](#configuration-best-practices)
- [Custom Hooks](#custom-hooks)
- [Prefetching](#prefetching)
- [Suspense Integration](#suspense-integration)
- [SSR Hydration](#ssr-hydration)
- [Testing Patterns](#testing-patterns)

## Configuration Best Practices

### QueryClient Setup

```typescript
// providers/query-provider.tsx
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,        // 1 minute
            gcTime: 5 * 60 * 1000,       // 5 minutes
            retry: 1,
            refetchOnWindowFocus: false,
            refetchOnReconnect: true,
          },
          mutations: {
            retry: 0,
          },
        },
      })
  )

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

### Environment-Specific Defaults

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: process.env.NODE_ENV === 'development' ? 0 : 60 * 1000,
      refetchOnWindowFocus: process.env.NODE_ENV === 'production',
    },
  },
})
```

### Option Guidelines

| Option | Recommended | Use Case |
|--------|-------------|----------|
| `staleTime` | 60s (lists), 5m (details) | Prevents unnecessary refetches |
| `gcTime` | 5-30m | Balance memory vs refetch cost |
| `retry` | 1-2 | Network flakiness recovery |
| `refetchOnWindowFocus` | false | User-initiated refetch preferred |
| `refetchOnMount` | true | Fresh data on navigation |

## Custom Hooks

### Encapsulating Query Logic

```typescript
// hooks/use-user.ts
import { useServerActionQuery } from '@saas4dev/core'
import { getUserAction } from '@/features/users/usecases/get/actions/get-user-action'
import { userKeys } from '@/lib/query-keys'

export function useUser(userId: string) {
  return useServerActionQuery(getUserAction, {
    input: { userId },
    queryKey: userKeys.detail(userId),
    enabled: !!userId,
  })
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading } = useUser(userId)
  return isLoading ? <Skeleton /> : <UserCard user={user} />
}
```

### Encapsulating Mutation Logic

```typescript
// hooks/use-create-user.ts
import { useQueryClient } from '@tanstack/react-query'
import { useServerActionMutation } from '@saas4dev/core'
import { createUserAction } from '@/features/users/usecases/create/actions/create-user-action'
import { userKeys } from '@/lib/query-keys'
import { toast } from 'sonner'

export function useCreateUser() {
  const queryClient = useQueryClient()

  return useServerActionMutation(createUserAction, {
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: userKeys.lists() })
      toast.success('User created')
    },
    onError: (error) => {
      toast.error(error.message)
    },
  })
}
```

### Combined Query + Mutation Hook

```typescript
// hooks/use-todo-item.ts
export function useTodoItem(todoId: string) {
  const queryClient = useQueryClient()

  const query = useServerActionQuery(getTodoAction, {
    input: { todoId },
    queryKey: todoKeys.detail(todoId),
  })

  const updateMutation = useServerActionMutation(updateTodoAction, {
    onSuccess: (updated) => {
      queryClient.setQueryData(todoKeys.detail(todoId), updated)
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
    },
  })

  const deleteMutation = useServerActionMutation(deleteTodoAction, {
    onSuccess: () => {
      queryClient.removeQueries({ queryKey: todoKeys.detail(todoId) })
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
    },
  })

  return {
    todo: query.data,
    isLoading: query.isLoading,
    update: updateMutation.mutate,
    delete: deleteMutation.mutate,
    isUpdating: updateMutation.isPending,
    isDeleting: deleteMutation.isPending,
  }
}
```

## Prefetching

### Prefetch on Hover

```typescript
function UserList({ users }: { users: User[] }) {
  const queryClient = useQueryClient()

  const prefetchUser = (userId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['users', userId],
      queryFn: () => fetchUser(userId),
      staleTime: 5 * 60 * 1000,
    })
  }

  return (
    <ul>
      {users.map((user) => (
        <li
          key={user.id}
          onMouseEnter={() => prefetchUser(user.id)}
        >
          <Link href={`/users/${user.id}`}>{user.name}</Link>
        </li>
      ))}
    </ul>
  )
}
```

### Prefetch on Route Navigation

```typescript
// In Next.js App Router
import { useRouter } from 'next/navigation'

function NavigationButton({ userId }: { userId: string }) {
  const router = useRouter()
  const queryClient = useQueryClient()

  const handleClick = async () => {
    // Prefetch data before navigation
    await queryClient.prefetchQuery({
      queryKey: ['users', userId],
      queryFn: () => fetchUser(userId),
    })
    router.push(`/users/${userId}`)
  }

  return <Button onClick={handleClick}>View User</Button>
}
```

## Suspense Integration

### useSuspenseQuery (React Query v5)

```typescript
import { useSuspenseQuery } from '@tanstack/react-query'

function UserProfile({ userId }: { userId: string }) {
  // Component suspends until data is ready
  const { data: user } = useSuspenseQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  })

  // data is always defined here
  return <UserCard user={user} />
}

// Parent component
function UserPage({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <UserProfile userId={userId} />
    </Suspense>
  )
}
```

### Error Boundaries with Suspense

```typescript
import { ErrorBoundary } from 'react-error-boundary'

function UserPage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

## SSR Hydration

### Next.js App Router Hydration

```typescript
// app/users/[id]/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'
import { getUserAction } from '@/features/users/usecases/get/actions/get-user-action'
import { UserProfile } from './user-profile'

export default async function UserPage({ params }: { params: { id: string } }) {
  const queryClient = new QueryClient()

  // Prefetch on server
  await queryClient.prefetchQuery({
    queryKey: ['users', params.id],
    queryFn: async () => {
      const [result, error] = await getUserAction({ userId: params.id })
      if (error) throw error
      return result
    },
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserProfile userId={params.id} />
    </HydrationBoundary>
  )
}
```

### Client Component Consuming Hydrated Data

```typescript
// user-profile.tsx
'use client'

import { useServerActionQuery } from '@saas4dev/core'
import { getUserAction } from '@/features/users/usecases/get/actions/get-user-action'

export function UserProfile({ userId }: { userId: string }) {
  // This will use the hydrated data immediately, no loading state
  const { data: user, isLoading } = useServerActionQuery(getUserAction, {
    input: { userId },
    queryKey: ['users', userId],
  })

  if (isLoading) return <Skeleton />
  return <UserCard user={user} />
}
```

## Testing Patterns

### Testing Queries

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, screen, waitFor } from '@testing-library/react'

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  })
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

test('renders user data', async () => {
  render(<UserProfile userId="1" />, { wrapper: createWrapper() })

  await waitFor(() => {
    expect(screen.getByText('John Doe')).toBeInTheDocument()
  })
})
```

### Mocking Server Actions

```typescript
import { vi } from 'vitest'

vi.mock('@/features/users/usecases/get/actions/get-user-action', () => ({
  getUserAction: vi.fn().mockResolvedValue([
    { id: '1', name: 'John Doe', email: 'john@example.com' },
    null,
  ]),
}))
```

### Testing Mutations

```typescript
import userEvent from '@testing-library/user-event'

test('creates user on form submit', async () => {
  const user = userEvent.setup()

  render(<CreateUserForm />, { wrapper: createWrapper() })

  await user.type(screen.getByLabelText('Name'), 'Jane Doe')
  await user.type(screen.getByLabelText('Email'), 'jane@example.com')
  await user.click(screen.getByRole('button', { name: 'Create' }))

  await waitFor(() => {
    expect(screen.getByText('User created')).toBeInTheDocument()
  })
})
```
