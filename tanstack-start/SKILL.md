---
name: tanstack-start
description: "Best practices for TanStack Start, the full-stack React framework built on TanStack Router and Vite/Nitro. Use when building server functions with createServerFn, setting up SSR and streaming with defer()/Await, writing API routes, configuring app.config.ts, or deploying a TanStack Start app to Node, Vercel, Netlify, Bun, or Cloudflare."
---

# TanStack Start

TanStack Start is a full-stack React framework that layers server functions, SSR, streaming, and API routes on top of TanStack Router and a Vite/Nitro build pipeline, giving end-to-end type safety from server to client.

## Workflow for Building a TanStack Start Feature

1. **Configure the app** — Set up `app.config.ts` with the Vite plugins and a `server.preset` matching the deployment target.
2. **Define the root route** — Create `src/routes/__root.tsx` with the HTML document shell (`<html>`, `<head>`, `<body>`, `<Outlet>`, `<ScrollRestoration>`, `<Scripts>`).
3. **Write server functions** — Use `createServerFn()` with a `.validator()` and `.handler()` for any logic that must run on the server (database access, secrets, file I/O).
4. **Wire server functions into routes** — Call server functions from route `loader`s for reads and from mutation hooks (`useMutation`) for writes.
5. **Add API routes for raw HTTP** — Use `createAPIFileRoute` for webhooks, OAuth callbacks, or endpoints that external services call directly.
6. **Stream non-critical data** — Wrap slow, non-blocking loader data in `defer()` and render it with `<Suspense>` + `<Await>`.
7. **Deploy** — Pick the `server.preset` for the target platform and verify environment variable access rules before shipping.

## Core Principles

- TanStack Start = TanStack Router + Vite + Nitro, giving full-stack React with a single type-safe codebase.
- `createServerFn` is the primary mechanism for server-side logic — prefer it over hand-rolled REST endpoints for internal data access.
- All TanStack Router conventions apply on top of Start: file-based routing, loaders, search param validation, and router context.
- Streaming and Suspense are first-class citizens; use `defer()` for anything that shouldn't block the initial response.
- Keep server-only code (secrets, DB clients, filesystem access) strictly inside server functions or API routes — never import it into client components.

## Configuration

### app.config.ts

```ts
import { defineConfig } from '@tanstack/start/config'
import tsConfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  vite: { plugins: [tsConfigPaths()] },
  server: {
    preset: 'node-server', // or: 'vercel', 'netlify', 'bun', 'cloudflare-pages'
  },
})
```

### Root Route HTML Shell

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet, ScrollRestoration } from '@tanstack/react-router'
import { Scripts } from '@tanstack/start'

export const Route = createRootRoute({
  component: () => (
    <html lang="en">
      <head />
      <body>
        <Outlet />
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  ),
})
```

## Server Functions

- Define server functions in a dedicated `src/server/functions/` (or colocated `*.server.ts`) module for discoverability.
- Always validate input with `.validator()` (Zod is the common choice) before the handler runs — never trust client-supplied `data`.
- Throw descriptive `Error`s (or a typed error) from handlers; TanStack Start serializes thrown errors back to the client.
- Server functions can be called directly from loaders, actions, event handlers, or `useMutation` — they behave like normal async functions on the client but execute on the server.

```ts
// src/server/functions/posts.ts
import { createServerFn } from '@tanstack/start'
import { z } from 'zod'
import { db } from '~/server/db'

export const getPost = createServerFn()
  .validator(z.object({ id: z.string() }))
  .handler(async ({ data }) => {
    const post = await db.post.findUnique({ where: { id: data.id } })
    if (!post) throw new Error('Post not found')
    return post
  })

export const createPost = createServerFn()
  .validator(z.object({ title: z.string().min(1), body: z.string() }))
  .handler(async ({ data }) => db.post.create({ data }))
```

### Using Server Functions in Routes

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => getPost({ data: { id: params.postId } }),
  component: PostDetail,
})

function PostDetail() {
  const post = Route.useLoaderData()
  return <h1>{post.title}</h1>
}
```

### Mutations with Server Functions

```tsx
import { useMutation } from '@tanstack/react-query'
import { useQueryClient } from '@tanstack/react-query'
import { createPost } from '~/server/functions/posts'

function NewPostForm() {
  const queryClient = useQueryClient()
  const mutation = useMutation({
    mutationFn: (input: { title: string; body: string }) => createPost({ data: input }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['posts'] }),
  })

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        const form = new FormData(e.currentTarget)
        mutation.mutate({
          title: String(form.get('title')),
          body: String(form.get('body')),
        })
      }}
    >
      <input name="title" placeholder="Title" required />
      <textarea name="body" placeholder="Body" required />
      <button type="submit" disabled={mutation.isPending}>Publish</button>
    </form>
  )
}
```

## API Routes

Use API routes for endpoints that must speak raw HTTP (webhooks, OAuth callbacks, health checks) instead of server functions, which are meant to be called from your own client code.

```ts
// src/routes/api/webhook.ts
import { createAPIFileRoute } from '@tanstack/start/api'

export const Route = createAPIFileRoute('/api/webhook')({
  POST: async ({ request }) => {
    const body = await request.json()
    // verify signature, enqueue processing, etc.
    return Response.json({ received: true })
  },
})
```

## Streaming with defer()

- Await only the data required for the initial paint; wrap everything else in `defer()` so the shell renders immediately.
- Pair deferred values with `<Suspense>` and `<Await>` so the UI shows a fallback until the promise resolves.

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await getPost({ data: { id: params.postId } }) // critical, awaited
    const comments = getComments({ data: { postId: params.postId } }) // not awaited
    return { post, comments: defer(comments) }
  },
  component: PostDetail,
})

function PostDetail() {
  const { post, comments } = Route.useLoaderData()
  return (
    <div>
      <h1>{post.title}</h1>
      <Suspense fallback={<CommentsSkeleton />}>
        <Await promise={comments}>{(c) => <CommentsList comments={c} />}</Await>
      </Suspense>
    </div>
  )
}
```

## Environment Variables

- Access server-only secrets via `process.env` only inside server functions or API routes.
- Use `import.meta.env.VITE_*` for any variable that must be exposed to the client bundle.
- Never reference `process.env` directly in a client component — Vite will not inline server secrets, and doing so is a common source of leaked configuration.

## Deployment Targets

Configure `server.preset` in `app.config.ts` to match the hosting platform:

- `node-server` — default, runs anywhere Node.js runs.
- `vercel` — Vercel serverless/edge functions.
- `netlify` — Netlify Functions.
- `bun` — Bun runtime.
- `cloudflare-pages` — Cloudflare Pages + Workers (watch for Node API compatibility gaps in server functions).

## Common Mistakes

- Calling a server function's handler logic directly from a client component instead of going through `createServerFn` — this breaks the client/server boundary and type inference.
- Awaiting everything in a loader, which defeats the purpose of streaming and slows the initial response.
- Forgetting `.validator()`, leaving handlers to trust unvalidated client input.
- Mixing `process.env` reads into shared utility modules that are imported by both client and server code.
