---
name: tanstack-query
description: "Best practices for TanStack Query v5 (React Query) server state management, including query key factories, the queryOptions helper, mutations, optimistic updates, infinite queries, and Suspense mode. Use when fetching or caching server data in React, writing custom query/mutation hooks, setting up a QueryClient, implementing optimistic updates, or migrating v4 patterns to v5."
---

# TanStack Query Best Practices

TanStack Query (formerly React Query) handles server-state caching, background updates, and stale-data management out of the box. This skill covers v5 patterns and APIs — v5 introduced several breaking changes from v4 that older examples online still don't reflect.

## Core Principles

- Use TanStack Query for all server state management and data fetching; it is not a general client-state manager — keep client-only state in `useState`/context/a state library instead
- Minimize `useEffect` and `useState` for server data; favor TanStack Query's built-in state management
- Every query needs a stable, serializable query key that uniquely describes the data it holds
- Mutations handle writes; queries handle reads — don't blur this boundary
- Implement proper error handling with user-friendly messages
- Use TypeScript for full type safety with query responses

## v5 Breaking Changes to Watch For

If you see or write any of these v4 patterns, update them:

- **Object syntax only**: `useQuery`, `useInfiniteQuery`, etc. no longer accept positional arguments (`useQuery(key, fn, options)`). Always pass a single options object: `useQuery({ queryKey, queryFn, ...options })`.
- **`isPending` replaces `isLoading`** as the name for "no data yet and a fetch is in flight" on `useMutation`. On `useQuery`, `isPending` means no cached data exists at all; `isLoading` is now derived (`isPending && isFetching`) and still usable for the classic "first load" spinner case.
- **`cacheTime` renamed to `gcTime`** (garbage collection time).
- **`queryOptions()` helper** for defining reusable, typed query definitions shared between components, loaders, and prefetch calls.
- **`useSuspenseQuery`** (and `useSuspenseInfiniteQuery`) for Suspense-based data fetching, replacing the old `suspense: true` option.
- **`placeholderData: keepPreviousData`** replaces the old `keepPreviousData: true` boolean for pagination.

## Project Structure

```
src/
  api/
    client.ts             # API client configuration
    endpoints/
      users.ts            # User-related API calls
      posts.ts            # Post-related API calls
  queries/
    postKeys.ts            # Query key factory
    postQueryOptions.ts     # queryOptions() definitions
  hooks/
    queries/
      useUsers.ts         # User query hooks
      usePosts.ts         # Post query hooks
    mutations/
      useCreateUser.ts    # User mutation hooks
  providers/
    QueryProvider.tsx     # Query client provider setup
  types/
    api.ts                # API response types
```

## Setup and Configuration

### Query Client Configuration

```typescript
// providers/QueryProvider.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes; defaults to 0 (always stale) if unset
      gcTime: 1000 * 60 * 30,   // 30 minutes (formerly cacheTime)
      retry: (failureCount, error: any) => error?.status !== 404 && failureCount < 3,
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
    },
  },
});

export function QueryProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

Instantiate `QueryClient` once at the app root — never inside a component, or the cache resets on every render.

## Query Best Practices

### 1. Query Key Organization

Use consistent, hierarchical query keys for efficient cache management:

```typescript
// Query key factory pattern
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};
```

### 2. queryOptions Helper (v5)

Define a query once with `queryOptions()` and reuse the same definition across components, router loaders, and prefetch calls — this keeps the query key, query function, and options in one place instead of duplicating them:

```typescript
// queries/postQueryOptions.ts
import { queryOptions } from '@tanstack/react-query';
import { postKeys } from './postKeys';
import { fetchPost } from '@/api/endpoints/posts';

export const postQueryOptions = (id: string) =>
  queryOptions({
    queryKey: postKeys.detail(id),
    queryFn: () => fetchPost(id),
    staleTime: 1000 * 60 * 5,
  });

// In a component
const { data } = useQuery(postQueryOptions(postId));

// In a TanStack Router loader — eliminates loading spinners on navigation
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params, context: { queryClient } }) =>
    queryClient.ensureQueryData(postQueryOptions(params.postId)),
});
```

Always define `queryOptions` outside components — never inline a fresh object literal in every `useQuery()` call — so the definition can be shared and prefetched.

### 3. Custom Query Hooks

Create reusable, typed query hooks when a `queryOptions()` factory isn't reused elsewhere:

```typescript
// hooks/queries/useUser.ts
import { useQuery, UseQueryOptions } from '@tanstack/react-query';
import { userKeys } from '@/api/queryKeys';
import { getUser, User } from '@/api/endpoints/users';

export function useUser(
  userId: string,
  options?: Omit<UseQueryOptions<User, Error>, 'queryKey' | 'queryFn'>
) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => getUser(userId),
    enabled: !!userId,
    ...options,
  });
}
```

### 4. Dependent Queries

Handle queries that depend on other data:

```typescript
function useUserPosts(userId: string) {
  const { data: user } = useUser(userId);

  return useQuery({
    queryKey: ['posts', { userId }],
    queryFn: () => fetchUserPosts(userId),
    enabled: !!user, // Only run when user data is available
  });
}
```

### 5. Parallel Queries

Fetch multiple resources simultaneously:

```typescript
import { useQueries } from '@tanstack/react-query';

function useMultipleUsers(userIds: string[]) {
  return useQueries({
    queries: userIds.map((id) => ({
      queryKey: userKeys.detail(id),
      queryFn: () => getUser(id),
    })),
  });
}
```

## Mutation Best Practices

### 1. Basic Mutations

```typescript
const { mutate, mutateAsync, isPending } = useMutation({
  mutationFn: (input: CreatePostInput) => createPost(input),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    toast.success('Post created!');
  },
  onError: (error) => {
    toast.error(error.message);
  },
});

// Usage
mutate({ title: 'Hello', body: '...' });
```

`isPending` is the v5 name for "mutation in flight" (v4 called this `isLoading` on mutations too — that name is gone).

### 2. Optimistic Updates

Provide instant feedback while mutations are in flight:

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateUser,
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: userKeys.detail(newUser.id) });

      // Snapshot previous value
      const previousUser = queryClient.getQueryData(userKeys.detail(newUser.id));

      // Optimistically update
      queryClient.setQueryData(userKeys.detail(newUser.id), newUser);

      return { previousUser };
    },
    onError: (err, newUser, context) => {
      // Rollback on error
      queryClient.setQueryData(
        userKeys.detail(newUser.id),
        context?.previousUser
      );
    },
    onSettled: (data, error, variables) => {
      // Refetch after error or success
      queryClient.invalidateQueries({ queryKey: userKeys.detail(variables.id) });
    },
  });
}
```

### 3. Cache Invalidation

Properly invalidate related queries after mutations:

```typescript
function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: deleteUser,
    onSuccess: () => {
      // Invalidate all user-related queries
      queryClient.invalidateQueries({ queryKey: userKeys.all });
    },
  });
}
```

Other cache operations worth knowing:

```typescript
// Remove from cache entirely (not just marked stale)
queryClient.removeQueries({ queryKey: userKeys.detail(id) });

// Directly write to the cache without a refetch
queryClient.setQueryData(userKeys.detail(id), newData);
```

## Error Handling

### 1. Global Error Handler

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      throwOnError: false,
    },
    mutations: {
      onError: (error) => {
        // Global error handling (e.g., toast notification)
        toast.error(error.message);
      },
    },
  },
});
```

### 2. Component-Level Error Handling

```typescript
function UserProfile({ userId }: { userId: string }) {
  const { data, error, isPending, isError } = useUser(userId);

  if (isPending) return <Skeleton />;
  if (isError) return <ErrorMessage error={error} />;

  return <UserCard user={data} />;
}
```

### 3. Conditional Retry Logic

Skip retries for errors that will never succeed on retry, like 404s:

```typescript
retry: (failureCount, error) => {
  if (error.status === 404) return false;
  return failureCount < 3;
},
```

### 4. Suspense Mode (v5)

Use `useSuspenseQuery` for Suspense-based data fetching instead of the old `suspense: true` option — it also narrows the return type since `data` can never be `undefined`:

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';
import { Suspense } from 'react';

function UserProfile({ userId }: { userId: string }) {
  // No need to check isPending — Suspense handles the loading state
  const { data } = useSuspenseQuery(userQueryOptions(userId));
  return <UserCard user={data} />;
}

function App() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<Loading />}>
        <UserProfile userId="123" />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Use `throwOnError: true` on a regular `useQuery` if you want errors to bubble to the nearest `ErrorBoundary` without switching to Suspense.

## Performance Optimization

### 1. Select and Transform Data

Only subscribe to the data you need:

```typescript
function useUserName(userId: string) {
  return useUser(userId, {
    select: (user) => user.name,
  });
}
```

### 2. Prefetching

Prefetch data before it's needed — on hover, or during routing:

```typescript
function UserList() {
  const queryClient = useQueryClient();

  const prefetchUser = (userId: string) => {
    queryClient.prefetchQuery(userQueryOptions(userId));
  };

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id} onMouseEnter={() => prefetchUser(user.id)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

### 3. Infinite Queries

Handle paginated data efficiently:

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: postKeys.lists(),
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    getPreviousPageParam: (firstPage) => firstPage.prevCursor,
  });
}

// data.pages is an array of page results — flatten for rendering
const allPosts = data?.pages.flatMap((page) => page.items) ?? [];
```

Use `placeholderData: keepPreviousData` (imported from `@tanstack/react-query`) on paginated or filtered queries to keep showing the previous page's data while the next page loads, instead of flashing a loading state:

```typescript
import { keepPreviousData, useQuery } from '@tanstack/react-query';

useQuery({
  queryKey: postKeys.list({ page }),
  queryFn: () => fetchPosts({ page }),
  placeholderData: keepPreviousData,
});
```

Use `notifyOnChangeProps` to limit re-renders to only the specific result properties a component actually reads.

## TypeScript Tips

- Always type `queryFn` return value explicitly, or infer it from typed API functions
- Use `QueryObserverResult<TData, TError>` to type hook return values
- Use `UseMutationResult<TData, TError, TVariables>` for mutations

## Key Conventions

1. **Feature-based organization**: Group query hooks and `queryOptions` factories within feature-specific directories
2. **Consistent query keys**: Use factory functions for type-safe, organized keys
3. **queryOptions everywhere reusable**: Prefer `queryOptions()` over ad hoc inline options whenever a query is used in more than one place (component, loader, prefetch)
4. **Type safety**: Define TypeScript interfaces for all API responses
5. **DevTools**: Always include React Query DevTools in development
6. **Avoid deeply nested queries**: Flatten query structures when possible
7. **Fetch only needed data**: Use API parameters to limit response size
8. **Handle loading and error states**: Always provide appropriate UI feedback

## Anti-Patterns to Avoid

- Do not use `useEffect` to fetch data — use queries or router loaders instead
- Do not store server state in local state (`useState`)
- Do not pass positional arguments to `useQuery`/`useInfiniteQuery` — v5 requires the options-object form
- Do not check `isLoading` alone on a mutation — use `isPending`
- Do not forget to handle loading and error states
- Do not create overly specific query keys that prevent cache reuse
- Do not skip cache invalidation after mutations
- Do not ignore the `enabled` option for conditional queries
- Do not define `queryOptions`/query configs inline inside components when they're reused elsewhere — co-locate and share them
