# Query Patterns Reference

## Table of Contents
- [Query Key Structure](#query-key-structure)
- [useQuery Patterns](#usequery-patterns)
- [useServerActionQuery Patterns](#useserveractionquery-patterns)
- [Conditional Queries](#conditional-queries)
- [Derived Data with select](#derived-data-with-select)
- [Pagination and Infinite Queries](#pagination-and-infinite-queries)
- [Parallel and Dependent Queries](#parallel-and-dependent-queries)

## Query Key Structure

### Key Hierarchy Pattern

```typescript
// Entity-based hierarchy
['users']                          // All users list
['users', 'list', { status }]      // Filtered users list
['users', 'detail', userId]        // Single user by ID
['users', userId, 'posts']         // User's posts (nested resource)

// Feature-based hierarchy
['todos']                          // All todos
['todos', 'list', { filter }]      // Filtered todos
['todos', 'detail', todoId]        // Single todo
['todos', todoId, 'comments']      // Todo's comments
```

### Query Key Factory Pattern

```typescript
// query-keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
}

// Usage
queryClient.invalidateQueries({ queryKey: userKeys.all })           // Invalidate all user queries
queryClient.invalidateQueries({ queryKey: userKeys.lists() })       // Invalidate all lists
queryClient.invalidateQueries({ queryKey: userKeys.detail(id) })    // Invalidate specific user
```

## useQuery Patterns

### Basic Query

```typescript
const { data, isLoading, isError, error, refetch } = useQuery({
  queryKey: ['users', userId],
  queryFn: () => fetchUser(userId),
})
```

### Query with Options

```typescript
const { data } = useQuery({
  queryKey: ['users', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000,      // Data fresh for 5 minutes
  gcTime: 30 * 60 * 1000,        // Keep in cache for 30 minutes (v5: gcTime replaces cacheTime)
  retry: 2,                       // Retry failed requests twice
  refetchOnWindowFocus: false,    // Don't refetch on window focus
  refetchOnMount: true,           // Refetch on component mount if stale
})
```

### Recommended Default Options

```typescript
// In QueryClientProvider setup
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,        // 1 minute
      gcTime: 5 * 60 * 1000,       // 5 minutes
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
})
```

## useServerActionQuery Patterns

Integration with ZSA server actions via `@saas4dev/core`:

### Basic Server Action Query

```typescript
import { useServerActionQuery } from '@saas4dev/core'
import { getUserAction } from '@/features/users/usecases/get/actions/get-user-action'

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error } = useServerActionQuery(getUserAction, {
    input: { userId },
    queryKey: ['users', userId],
  })

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage error={error} />
  return <UserCard user={data} />
}
```

### List Query with Filters

```typescript
import { useServerActionQuery } from '@saas4dev/core'
import { listUsersAction } from '@/features/users/usecases/list/actions/list-users-action'

function UserList({ status }: { status: string }) {
  const { data, isLoading } = useServerActionQuery(listUsersAction, {
    input: { status },
    queryKey: ['users', 'list', { status }],
  })

  return (
    <ul>
      {data?.map((user) => <UserRow key={user.id} user={user} />)}
    </ul>
  )
}
```

## Conditional Queries

### Using enabled Option

```typescript
// Only fetch when userId exists
const { data } = useServerActionQuery(getUserAction, {
  input: { userId: userId! },
  queryKey: ['users', userId],
  enabled: !!userId,
})

// Chain dependent queries
const { data: user } = useServerActionQuery(getUserAction, {
  input: { userId },
  queryKey: ['users', userId],
})

const { data: posts } = useServerActionQuery(getUserPostsAction, {
  input: { userId: user?.id! },
  queryKey: ['users', user?.id, 'posts'],
  enabled: !!user?.id,
})
```

## Derived Data with select

Transform query data before rendering:

```typescript
const { data: activeUsers } = useServerActionQuery(listUsersAction, {
  input: {},
  queryKey: ['users', 'list'],
  select: (users) => users.filter((u) => u.status === 'active'),
})

const { data: userCount } = useServerActionQuery(listUsersAction, {
  input: {},
  queryKey: ['users', 'list'],
  select: (users) => users.length,
})
```

Benefits:
- Original data stays in cache
- Derived data recomputed only when source changes
- Multiple components can use same query with different selects

## Pagination and Infinite Queries

### Offset-based Pagination

```typescript
function PaginatedList() {
  const [page, setPage] = useState(1)

  const { data, isLoading, isPlaceholderData } = useServerActionQuery(
    listUsersAction,
    {
      input: { page, limit: 20 },
      queryKey: ['users', 'list', { page }],
      placeholderData: (prev) => prev, // Keep previous data while fetching
    }
  )

  return (
    <div className={isPlaceholderData ? 'opacity-50' : ''}>
      <UserTable users={data?.items ?? []} />
      <Pagination
        page={page}
        totalPages={data?.totalPages ?? 1}
        onPageChange={setPage}
      />
    </div>
  )
}
```

### Infinite Scroll with useServerActionInfiniteQuery

```typescript
import { useServerActionInfiniteQuery } from '@saas4dev/core'

function InfiniteUserList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useServerActionInfiniteQuery(listUsersAction, {
    input: {},
    queryKey: ['users', 'infinite'],
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  })

  const allUsers = data?.pages.flatMap((page) => page.items) ?? []

  return (
    <>
      {allUsers.map((user) => <UserCard key={user.id} user={user} />)}
      {hasNextPage && (
        <Button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </Button>
      )}
    </>
  )
}
```

## Parallel and Dependent Queries

### Parallel Queries

```typescript
function Dashboard({ userId }: { userId: string }) {
  // These queries run in parallel
  const userQuery = useServerActionQuery(getUserAction, {
    input: { userId },
    queryKey: ['users', userId],
  })

  const statsQuery = useServerActionQuery(getUserStatsAction, {
    input: { userId },
    queryKey: ['users', userId, 'stats'],
  })

  const postsQuery = useServerActionQuery(getUserPostsAction, {
    input: { userId },
    queryKey: ['users', userId, 'posts'],
  })

  if (userQuery.isLoading || statsQuery.isLoading || postsQuery.isLoading) {
    return <DashboardSkeleton />
  }

  return (
    <DashboardLayout
      user={userQuery.data}
      stats={statsQuery.data}
      posts={postsQuery.data}
    />
  )
}
```

### Dependent (Waterfall) Queries

```typescript
function UserWithOrganization({ userId }: { userId: string }) {
  const { data: user } = useServerActionQuery(getUserAction, {
    input: { userId },
    queryKey: ['users', userId],
  })

  // Only runs after user is loaded
  const { data: organization } = useServerActionQuery(getOrganizationAction, {
    input: { orgId: user?.organizationId! },
    queryKey: ['organizations', user?.organizationId],
    enabled: !!user?.organizationId,
  })

  return <UserOrgDisplay user={user} organization={organization} />
}
```
