# Mutation Patterns Reference

## Table of Contents
- [Basic Mutations](#basic-mutations)
- [useServerActionMutation Patterns](#useserveractionmutation-patterns)
- [Cache Invalidation Strategies](#cache-invalidation-strategies)
- [Optimistic Updates](#optimistic-updates)
- [Error Handling and Rollback](#error-handling-and-rollback)
- [Mutation Lifecycle Callbacks](#mutation-lifecycle-callbacks)

## Basic Mutations

### useMutation Structure

```typescript
const mutation = useMutation({
  mutationFn: (data: CreateUserInput) => createUser(data),
  onSuccess: (data, variables, context) => {
    // Runs on success
  },
  onError: (error, variables, context) => {
    // Runs on error
  },
  onSettled: (data, error, variables, context) => {
    // Runs on both success and error
  },
})

// Usage
mutation.mutate({ name: 'John' })
```

## useServerActionMutation Patterns

Integration with ZSA server actions:

### Basic Server Action Mutation

```typescript
import { useServerActionMutation } from '@saas4dev/core'
import { createUserAction } from '@/features/users/usecases/create/actions/create-user-action'

function CreateUserForm() {
  const mutation = useServerActionMutation(createUserAction, {
    onSuccess: () => {
      toast.success('User created')
    },
    onError: (error) => {
      toast.error(error.message)
    },
  })

  const handleSubmit = (data: CreateUserInput) => {
    mutation.mutate(data)
  }

  return (
    <form onSubmit={form.handleSubmit(handleSubmit)}>
      {/* fields */}
      <Button disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create User'}
      </Button>
    </form>
  )
}
```

### Update Mutation

```typescript
import { useServerActionMutation } from '@saas4dev/core'
import { updateUserAction } from '@/features/users/usecases/update/actions/update-user-action'

function EditUserForm({ user }: { user: User }) {
  const mutation = useServerActionMutation(updateUserAction, {
    onSuccess: () => {
      toast.success('User updated')
    },
  })

  const handleSubmit = (data: UpdateUserInput) => {
    mutation.mutate({ userId: user.id, ...data })
  }

  return <form onSubmit={form.handleSubmit(handleSubmit)}>{/* fields */}</form>
}
```

### Delete Mutation

```typescript
import { useServerActionMutation } from '@saas4dev/core'
import { deleteUserAction } from '@/features/users/usecases/delete/actions/delete-user-action'

function DeleteUserButton({ userId }: { userId: string }) {
  const mutation = useServerActionMutation(deleteUserAction, {
    onSuccess: () => {
      toast.success('User deleted')
    },
  })

  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button variant="destructive">Delete</Button>
      </AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Delete user?</AlertDialogTitle>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction
            onClick={() => mutation.mutate({ userId })}
            disabled={mutation.isPending}
          >
            {mutation.isPending ? 'Deleting...' : 'Delete'}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

## Cache Invalidation Strategies

### Basic Invalidation

```typescript
import { useQueryClient } from '@tanstack/react-query'

function CreateUserForm() {
  const queryClient = useQueryClient()

  const mutation = useServerActionMutation(createUserAction, {
    onSuccess: () => {
      // Invalidate user list to refetch
      queryClient.invalidateQueries({ queryKey: ['users'] })
    },
  })
}
```

### Granular Invalidation

```typescript
const queryClient = useQueryClient()

const mutation = useServerActionMutation(updateUserAction, {
  onSuccess: (data, variables) => {
    // Invalidate specific user detail
    queryClient.invalidateQueries({ queryKey: ['users', variables.userId] })
    // Invalidate user lists (will refetch all list variants)
    queryClient.invalidateQueries({ queryKey: ['users', 'list'] })
  },
})
```

### Invalidation with Query Key Factory

```typescript
import { userKeys } from '@/lib/query-keys'

const mutation = useServerActionMutation(deleteUserAction, {
  onSuccess: (data, variables) => {
    // Remove specific user from cache
    queryClient.removeQueries({ queryKey: userKeys.detail(variables.userId) })
    // Invalidate all user lists
    queryClient.invalidateQueries({ queryKey: userKeys.lists() })
  },
})
```

### Cache Update Without Refetch

```typescript
const mutation = useServerActionMutation(updateUserAction, {
  onSuccess: (updatedUser) => {
    // Directly update cache without refetching
    queryClient.setQueryData(['users', updatedUser.id], updatedUser)

    // Update user in list cache
    queryClient.setQueryData(['users', 'list'], (oldData: User[] | undefined) =>
      oldData?.map((user) =>
        user.id === updatedUser.id ? updatedUser : user
      )
    )
  },
})
```

## Optimistic Updates

### When to Use Optimistic Updates

Use optimistic updates when:
- The mutation is very likely to succeed
- Immediate UI feedback improves UX significantly
- The data shape is predictable
- Rollback is straightforward

Avoid when:
- Mutation may fail frequently
- Server returns transformed/computed data
- Multiple caches need coordinated updates

### Optimistic Update Pattern

```typescript
const queryClient = useQueryClient()

const mutation = useServerActionMutation(updateTodoAction, {
  onMutate: async (newTodo) => {
    // Cancel any outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos', newTodo.id] })

    // Snapshot previous value
    const previousTodo = queryClient.getQueryData(['todos', newTodo.id])

    // Optimistically update cache
    queryClient.setQueryData(['todos', newTodo.id], (old: Todo) => ({
      ...old,
      ...newTodo,
    }))

    // Return context for rollback
    return { previousTodo }
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    if (context?.previousTodo) {
      queryClient.setQueryData(['todos', newTodo.id], context.previousTodo)
    }
    toast.error('Update failed')
  },
  onSettled: (data, error, variables) => {
    // Always refetch after mutation settles
    queryClient.invalidateQueries({ queryKey: ['todos', variables.id] })
  },
})
```

### Optimistic Update for List Operations

```typescript
// Adding to list
const addMutation = useServerActionMutation(createTodoAction, {
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    const previousTodos = queryClient.getQueryData(['todos'])

    // Optimistically add with temporary ID
    queryClient.setQueryData(['todos'], (old: Todo[] = []) => [
      ...old,
      { ...newTodo, id: `temp-${Date.now()}`, status: 'pending' },
    ])

    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})

// Removing from list
const deleteMutation = useServerActionMutation(deleteTodoAction, {
  onMutate: async ({ todoId }) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    const previousTodos = queryClient.getQueryData(['todos'])

    queryClient.setQueryData(['todos'], (old: Todo[] = []) =>
      old.filter((todo) => todo.id !== todoId)
    )

    return { previousTodos }
  },
  onError: (err, variables, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Error Handling and Rollback

### Error State Handling

```typescript
function MutationForm() {
  const mutation = useServerActionMutation(createAction)

  return (
    <form onSubmit={form.handleSubmit((data) => mutation.mutate(data))}>
      {mutation.isError && (
        <Alert variant="destructive">
          <AlertDescription>
            {mutation.error?.message ?? 'Something went wrong'}
          </AlertDescription>
        </Alert>
      )}
      {/* fields */}
    </form>
  )
}
```

### Retry Logic

```typescript
const mutation = useServerActionMutation(createAction, {
  retry: 2,                    // Retry twice on failure
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
})
```

## Mutation Lifecycle Callbacks

### Callback Execution Order

```typescript
const mutation = useServerActionMutation(action, {
  // 1. Called before mutation function
  onMutate: (variables) => {
    // Setup, optimistic updates
    return context  // Passed to onError/onSuccess/onSettled
  },

  // 2a. Called on success
  onSuccess: (data, variables, context) => {
    // Update cache, show success message
  },

  // 2b. Called on error
  onError: (error, variables, context) => {
    // Rollback, show error message
  },

  // 3. Always called after success or error
  onSettled: (data, error, variables, context) => {
    // Invalidate queries, cleanup
  },
})
```

### Mutation with Toast Notifications

```typescript
const mutation = useServerActionMutation(createUserAction, {
  onMutate: () => {
    // Show loading toast
    return { toastId: toast.loading('Creating user...') }
  },
  onSuccess: (data, variables, context) => {
    toast.success('User created', { id: context?.toastId })
  },
  onError: (error, variables, context) => {
    toast.error(error.message, { id: context?.toastId })
  },
})
```
