# React Query Integration

ZSA integrates seamlessly with TanStack React Query for data fetching and caching.

## Installation

```bash
npm install zsa-react-query @tanstack/react-query
```

## Provider Setup

```typescript
// app/providers.tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

```typescript
// app/layout.tsx
import { Providers } from "./providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## useServerActionQuery

Use for fetching data (GET-like operations).

```typescript
// actions/posts.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const getPostsAction = createServerAction()
  .input(
    z.object({
      page: z.number().default(1),
      limit: z.number().default(10),
      status: z.enum(["draft", "published"]).optional(),
    })
  )
  .handler(async ({ input }) => {
    const posts = await db.post.findMany({
      where: { status: input.status },
      skip: (input.page - 1) * input.limit,
      take: input.limit,
    });
    return { posts, page: input.page };
  });

export const getPostByIdAction = createServerAction()
  .input(z.object({ id: z.string() }))
  .handler(async ({ input }) => {
    const post = await db.post.findUnique({ where: { id: input.id } });
    if (!post) throw new Error("Post not found");
    return post;
  });
```

```typescript
// components/posts-list.tsx
"use client";

import { useServerActionQuery } from "zsa-react-query";
import { getPostsAction } from "@/actions/posts";

export function PostsList() {
  const {
    data,
    isLoading,
    isError,
    error,
    refetch,
    isFetching,
  } = useServerActionQuery(getPostsAction, {
    input: { page: 1, limit: 10, status: "published" },
    queryKey: ["posts", "published"], // Custom query key
  });

  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  return (
    <div>
      {isFetching && <span>Refreshing...</span>}
      <button onClick={() => refetch()}>Refresh</button>
      <ul>
        {data?.posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## useServerActionMutation

Use for mutations (POST/PUT/DELETE-like operations).

```typescript
// actions/posts.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";
import { revalidatePath } from "next/cache";

export const createPostAction = createServerAction()
  .input(z.object({ title: z.string(), content: z.string() }))
  .handler(async ({ input }) => {
    const post = await db.post.create({ data: input });
    revalidatePath("/posts");
    return post;
  });

export const updatePostAction = createServerAction()
  .input(z.object({ id: z.string(), title: z.string(), content: z.string() }))
  .handler(async ({ input }) => {
    const post = await db.post.update({
      where: { id: input.id },
      data: { title: input.title, content: input.content },
    });
    revalidatePath("/posts");
    return post;
  });

export const deletePostAction = createServerAction()
  .input(z.object({ id: z.string() }))
  .handler(async ({ input }) => {
    await db.post.delete({ where: { id: input.id } });
    revalidatePath("/posts");
    return { success: true };
  });
```

```typescript
// components/create-post-form.tsx
"use client";

import { useServerActionMutation } from "zsa-react-query";
import { createPostAction } from "@/actions/posts";
import { useQueryClient } from "@tanstack/react-query";

export function CreatePostForm() {
  const queryClient = useQueryClient();

  const { mutate, isPending, isError, error } = useServerActionMutation(
    createPostAction,
    {
      onSuccess: (data) => {
        // Invalidate posts query to refetch
        queryClient.invalidateQueries({ queryKey: ["posts"] });
        toast.success("Post created!");
      },
      onError: (err) => {
        toast.error(err.message);
      },
    }
  );

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);

    mutate({
      title: formData.get("title") as string,
      content: formData.get("content") as string,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" placeholder="Title" disabled={isPending} />
      <textarea name="content" placeholder="Content" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Create Post"}
      </button>
      {isError && <p className="error">{error.message}</p>}
    </form>
  );
}
```

## Optimistic Updates with React Query

```typescript
// components/like-button.tsx
"use client";

import { useServerActionMutation } from "zsa-react-query";
import { useQueryClient } from "@tanstack/react-query";
import { toggleLikeAction } from "@/actions/likes";

interface Post {
  id: string;
  title: string;
  likes: number;
  isLiked: boolean;
}

export function LikeButton({ post }: { post: Post }) {
  const queryClient = useQueryClient();

  const { mutate, isPending } = useServerActionMutation(toggleLikeAction, {
    onMutate: async ({ postId }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ["post", postId] });

      // Snapshot previous value
      const previousPost = queryClient.getQueryData<Post>(["post", postId]);

      // Optimistically update
      queryClient.setQueryData<Post>(["post", postId], (old) =>
        old
          ? {
              ...old,
              isLiked: !old.isLiked,
              likes: old.isLiked ? old.likes - 1 : old.likes + 1,
            }
          : old
      );

      return { previousPost };
    },
    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousPost) {
        queryClient.setQueryData(
          ["post", variables.postId],
          context.previousPost
        );
      }
      toast.error("Failed to update like");
    },
    onSettled: (data, error, { postId }) => {
      // Always refetch after mutation
      queryClient.invalidateQueries({ queryKey: ["post", postId] });
    },
  });

  return (
    <button onClick={() => mutate({ postId: post.id })} disabled={isPending}>
      {post.isLiked ? "❤️" : "🤍"} {post.likes}
    </button>
  );
}
```

## Infinite Query

```typescript
// actions/posts.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const getPostsInfiniteAction = createServerAction()
  .input(
    z.object({
      cursor: z.string().optional(),
      limit: z.number().default(10),
    })
  )
  .handler(async ({ input }) => {
    const posts = await db.post.findMany({
      take: input.limit + 1,
      cursor: input.cursor ? { id: input.cursor } : undefined,
      orderBy: { createdAt: "desc" },
    });

    let nextCursor: string | undefined;
    if (posts.length > input.limit) {
      const nextItem = posts.pop();
      nextCursor = nextItem?.id;
    }

    return { posts, nextCursor };
  });
```

```typescript
// components/infinite-posts.tsx
"use client";

import { useServerActionInfiniteQuery } from "zsa-react-query";
import { getPostsInfiniteAction } from "@/actions/posts";

export function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useServerActionInfiniteQuery(getPostsInfiniteAction, {
    input: { limit: 10 },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    queryKey: ["posts", "infinite"],
  });

  if (isLoading) return <div>Loading...</div>;

  const allPosts = data?.pages.flatMap((page) => page.posts) ?? [];

  return (
    <div>
      <ul>
        {allPosts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? "Loading more..." : "Load More"}
        </button>
      )}
    </div>
  );
}
```

## Query with Dependencies

```typescript
// components/user-posts.tsx
"use client";

import { useServerActionQuery } from "zsa-react-query";
import { getUserAction, getUserPostsAction } from "@/actions/users";

export function UserPosts({ userId }: { userId: string }) {
  // First query
  const userQuery = useServerActionQuery(getUserAction, {
    input: { id: userId },
    queryKey: ["user", userId],
  });

  // Dependent query - only runs when user is loaded
  const postsQuery = useServerActionQuery(getUserPostsAction, {
    input: { userId },
    queryKey: ["user", userId, "posts"],
    enabled: !!userQuery.data, // Only fetch when user exists
  });

  if (userQuery.isLoading) return <div>Loading user...</div>;
  if (userQuery.isError) return <div>Error loading user</div>;

  return (
    <div>
      <h1>{userQuery.data.name}'s Posts</h1>
      {postsQuery.isLoading ? (
        <div>Loading posts...</div>
      ) : (
        <ul>
          {postsQuery.data?.map((post) => (
            <li key={post.id}>{post.title}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## Prefetching

```typescript
// app/posts/page.tsx
import { QueryClient, HydrationBoundary, dehydrate } from "@tanstack/react-query";
import { getPostsAction } from "@/actions/posts";
import { PostsList } from "@/components/posts-list";

export default async function PostsPage() {
  const queryClient = new QueryClient();

  // Prefetch on server
  await queryClient.prefetchQuery({
    queryKey: ["posts", "published"],
    queryFn: () => getPostsAction({ page: 1, limit: 10, status: "published" }),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostsList />
    </HydrationBoundary>
  );
}
```

## Polling/Refetch Interval

```typescript
// components/live-stats.tsx
"use client";

import { useServerActionQuery } from "zsa-react-query";
import { getStatsAction } from "@/actions/stats";

export function LiveStats() {
  const { data, dataUpdatedAt } = useServerActionQuery(getStatsAction, {
    input: {},
    queryKey: ["stats"],
    refetchInterval: 5000, // Refetch every 5 seconds
    refetchIntervalInBackground: true, // Continue polling when tab is hidden
  });

  return (
    <div>
      <p>Active Users: {data?.activeUsers}</p>
      <p>Last updated: {new Date(dataUpdatedAt).toLocaleTimeString()}</p>
    </div>
  );
}
```

## Error Retry Configuration

```typescript
// components/posts-with-retry.tsx
"use client";

import { useServerActionQuery } from "zsa-react-query";
import { getPostsAction } from "@/actions/posts";

export function PostsWithRetry() {
  const { data, isLoading, isError, error, failureCount } =
    useServerActionQuery(getPostsAction, {
      input: { page: 1 },
      queryKey: ["posts"],
      retry: 3, // Retry 3 times
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000), // Exponential backoff
    });

  if (isLoading) {
    return (
      <div>
        Loading...
        {failureCount > 0 && <span> (Retry {failureCount}/3)</span>}
      </div>
    );
  }

  if (isError) return <div>Error: {error.message}</div>;

  return <ul>{data?.posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

## Combining with useServerAction

```typescript
// components/post-manager.tsx
"use client";

import { useServerAction } from "zsa-react";
import { useServerActionQuery, useServerActionMutation } from "zsa-react-query";
import { useQueryClient } from "@tanstack/react-query";
import { getPostsAction, createPostAction, quickUpdateAction } from "@/actions/posts";

export function PostManager() {
  const queryClient = useQueryClient();

  // Query for fetching
  const { data: posts, isLoading } = useServerActionQuery(getPostsAction, {
    input: { page: 1 },
    queryKey: ["posts"],
  });

  // Mutation with React Query integration
  const createMutation = useServerActionMutation(createPostAction, {
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["posts"] });
    },
  });

  // Simple action without React Query (for one-off operations)
  const { execute: quickUpdate, isPending } = useServerAction(quickUpdateAction);

  return (
    <div>
      {/* UI implementation */}
    </div>
  );
}
```