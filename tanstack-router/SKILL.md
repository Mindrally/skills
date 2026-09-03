---
name: tanstack-router
description: "Type-safe, file-based routing for React with TanStack Router. Use when defining routes with createFileRoute, validating search params with Zod, writing route loaders, setting up auth guards with beforeLoad, integrating TanStack Query into loaders, or configuring router context and preloading."
---

# TanStack Router

TanStack Router is a fully type-safe router for React that treats routes, loaders, params, and search params as first-class typed data rather than untyped strings.

## Workflow for Adding a New Route

1. **Create the route file** — Add a file under `src/routes/` following the file-based routing convention (see below).
2. **Define the route** — Export `Route` from `createFileRoute('/path')({...})` with `component`, and optionally `loader`, `validateSearch`, `beforeLoad`, `errorComponent`, and `pendingComponent`.
3. **Validate search params** — If the route reads query params, define a Zod schema and pass it as `validateSearch`.
4. **Load data** — Fetch data in `loader`, not in a component `useEffect`; integrate with TanStack Query via `queryClient.ensureQueryData` when caching is needed.
5. **Guard access** — Add `beforeLoad` checks (e.g. auth) that `throw redirect({ to: '/login' })` when a precondition fails.
6. **Link to the route** — Navigate with `<Link to="/path" params={...} search={...}>` so the compiler validates params and search at every call site.
7. **Regenerate the route tree** — Ensure `routeTree.gen.ts` is regenerated (automatic under the Vite plugin's dev server / build) before running or building the app.

## Core Principles

- TanStack Router is 100% type-safe — lean on TypeScript generics for params, search params, and loader data instead of manual casting.
- Prefer file-based routing with `@tanstack/router-vite-plugin` (or `@tanstack/router-plugin/vite`) for scalability over manually constructed route trees.
- Always define routes with `createFileRoute` (leaf/nested routes) or `createRootRoute` / `createRootRouteWithContext` (root).
- Route data loading belongs in `loader` functions, not in component `useEffect` — this enables preloading, parallel loading, and pending/error states.
- Search params are first-class state — always define their schema with Zod (or another standard-schema validator) so they are typed and validated on every read.

## File-Based Route Conventions

```
src/routes/
  __root.tsx          ← Root layout
  index.tsx           ← / route
  posts/
    index.tsx         ← /posts
    $postId.tsx       ← /posts/:postId (dynamic segment)
    _layout.tsx       ← Layout route (no path segment)
  _auth/               ← Pathless auth layout group
    dashboard.tsx
```

- A leading underscore on a segment (`_layout`, `_auth`) creates a pathless layout route used purely for grouping/shared UI.
- A `$` prefix (`$postId`) marks a dynamic path segment, matching `Route.useParams()`.
- `index.tsx` inside a folder matches the folder's own path with no additional segment.

## Route Definition

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
  component: PostComponent,
  errorComponent: ({ error }) => <ErrorBanner message={error.message} />,
  pendingComponent: () => <PostSkeleton />,
})

function PostComponent() {
  const post = Route.useLoaderData()     // type-safe
  const { postId } = Route.useParams()   // type-safe
  return <div>{post.title}</div>
}
```

- `errorComponent` renders when the loader throws; `pendingComponent` renders while the loader is in flight (subject to `defaultPendingMs`).
- Loader return values and thrown errors are both fully typed and flow into `useLoaderData()` and `errorComponent` respectively.

## Type-Safe Search Params

- Always define search params with Zod and `validateSearch`.
- Access them with `Route.useSearch()` — never read `window.location.search` or `URLSearchParams` directly, which bypasses type safety and validation.

```tsx
const searchSchema = z.object({
  page: z.number().int().min(1).default(1),
  q: z.string().optional(),
})

export const Route = createFileRoute('/search')({
  validateSearch: searchSchema,
  component: SearchPage,
})

function SearchPage() {
  const { page, q } = Route.useSearch()
  const navigate = Route.useNavigate()

  return (
    <button onClick={() => navigate({ search: (prev) => ({ ...prev, page: page + 1 }) })}>
      Next page
    </button>
  )
}
```

## Navigation

- Use `<Link>` for internal navigation — never a raw `<a href>`, which triggers a full page reload and loses client-side routing state.
- Always pass typed `params` and `search`; the TypeScript compiler catches missing or mistyped route params at build time.

```tsx
<Link to="/posts/$postId" params={{ postId: '123' }}>View Post</Link>
<Link to="/search" search={{ page: 1, q: 'react' }}>Search</Link>
```

## Loaders + TanStack Query Integration

- Put the `QueryClient` in router context so loaders can prefetch and cache through TanStack Query rather than duplicating fetch logic.
- Use `ensureQueryData` (not `fetchQuery`) in loaders so an already-cached query is reused instead of refetched.

```tsx
export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQueryOptions()),
  component: PostsPage,
})

function PostsPage() {
  const { data: posts } = useSuspenseQuery(postsQueryOptions())
  return <PostList posts={posts} />
}
```

## Router Context for Dependency Injection

```tsx
// __root.tsx
interface RouterContext {
  queryClient: QueryClient
  auth: AuthState
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootLayout,
})

// main.tsx
const router = createRouter({ routeTree, context: { queryClient, auth } })
```

- Router context flows down to every route's `loader` and `beforeLoad` via the `context` argument, giving each route typed access to shared dependencies without prop drilling or global singletons.

## Auth Guards

```tsx
export const Route = createFileRoute('/_auth/dashboard')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) throw redirect({ to: '/login' })
  },
  component: Dashboard,
})
```

- `beforeLoad` runs before the loader and before the component renders, making it the right place for auth checks, feature flag gates, and redirects.
- Prefer `throw redirect(...)` over imperative navigation inside components — it works during SSR, preloading, and client navigation alike.

## Performance

- Set `defaultPreload: 'intent'` on the router so links prefetch their route's data on hover/focus, making navigation feel instant.
- Use `React.lazy` (or the router's built-in code-splitting via `.lazy()` route files) for route component code splitting on large apps.
- Install `@tanstack/router-devtools` and render `<TanStackRouterDevtools />` in development to inspect route matches, pending states, and cache.

## Common Mistakes

- Fetching data with `useEffect` inside a route component instead of a `loader`, which loses preloading and creates render-then-fetch waterfalls.
- Reading raw `window.location.search` instead of `Route.useSearch()`, bypassing the search param schema and type safety.
- Performing auth checks only inside components (`useEffect` redirect) instead of `beforeLoad`, allowing a flash of protected content before redirecting.
- Forgetting to add new route params/search fields to `<Link>` call sites — let the compiler catch it rather than testing manually.
