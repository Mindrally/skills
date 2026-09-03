---
name: react-router-v7
description: "React Router v7 framework mode conventions for route modules, data loaders, actions, and progressive enhancement. Use when building file-based routes with loaders/actions, handling form submissions with Form/useFetcher/useSubmit, adding route ErrorBoundary components, revalidating data after mutations, or migrating Remix-style route modules to React Router v7."
---

# React Router v7

React Router v7 framework mode unifies routing, data loading, and mutations into route modules, each exporting a `loader`, `action`, `default` component, and `ErrorBoundary` for a single URL segment.

## Workflow for Building a Route Module

1. **Create the route file** — Add a file under `app/routes/` (or your configured routes directory) for the URL segment, using nested folders/files for layouts and dynamic segments.
2. **Export a loader** — Fetch the data the route needs before render; validate route params and search params at the top of the loader.
3. **Export a component** — Read loader data via `useLoaderData()` and render UI; keep the component focused on presentation.
4. **Export an action** — Handle `POST`/`PUT`/`DELETE` submissions from `<Form>` or `useFetcher`; return typed data (including validation errors) instead of throwing for expected failures.
5. **Export an ErrorBoundary** — Handle thrown errors (400s, 500s, unexpected exceptions) locally so one route's failure doesn't take down the whole app.
6. **Wire up navigation** — Use `<Form>`, `useFetcher`, and `useSubmit` for mutations so the app works with JavaScript disabled and gets optimistic UI when it's enabled.
7. **Test the module** — Write tests for the loader, action, validation failures, and error boundary independently of the rendered component tree.

## Route Modules

- Use route modules as the boundary for route UI, loader data, actions, metadata, and error boundaries — a route module is the unit of code splitting and data loading.
- Keep route modules small; move shared UI to `app/components/` and reusable data access to a services layer (e.g. `app/services/posts.server.ts`).
- Prefer file-based routing in framework mode when the project is configured for it; use the explicit `routes.ts` config for advanced/dynamic route trees.
- Use nested routes for shared layouts and progressive disclosure — a parent route's `<Outlet />` renders the matched child.
- Export route-specific `ErrorBoundary` components for recoverable route failures instead of relying on a single app-wide boundary.

```tsx
// app/routes/posts.$postId.tsx
import { Form, useLoaderData, useNavigation } from "react-router";
import type { Route } from "./+types/posts.$postId";
import { getPost, updatePost } from "~/services/posts.server";

export async function loader({ params }: Route.LoaderArgs) {
  const post = await getPost(params.postId);
  if (!post) {
    throw new Response("Not Found", { status: 404 });
  }
  return { post };
}

export async function action({ request, params }: Route.ActionArgs) {
  const formData = await request.formData();
  const title = String(formData.get("title") ?? "");

  if (title.trim().length === 0) {
    return { ok: false as const, errors: { title: "Title is required" } };
  }

  await updatePost(params.postId, { title });
  return { ok: true as const, errors: null };
}

export default function PostDetail() {
  const { post } = useLoaderData<typeof loader>();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === "submitting";

  return (
    <Form method="post">
      <input name="title" defaultValue={post.title} />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Saving..." : "Save"}
      </button>
    </Form>
  );
}

export function ErrorBoundary({ error }: Route.ErrorBoundaryProps) {
  if (error instanceof Response && error.status === 404) {
    return <p>Post not found.</p>;
  }
  return <p>Something went wrong loading this post.</p>;
}
```

## Data Loading

- Use loaders for route data that should be available before render — this avoids loading spinners for data the page can't function without.
- Keep loaders deterministic and side-effect free; a loader should read data, not write it.
- Validate params and search params at the loader boundary (e.g. parse and range-check a `page` query param) rather than trusting them downstream.
- Return typed data and consume it through `useLoaderData<typeof loader>()` rather than duplicating fetch logic in the component.
- Use deferred or streaming patterns (`defer`, or returning promises for `<Await>`) only when they measurably improve perceived performance — not by default.

## Mutations

- Use actions for route mutations and form submissions; an action runs on `POST`/`PUT`/`PATCH`/`DELETE` navigations and fetcher submissions to that route.
- Prefer `<Form>`, `useFetcher`, and `useSubmit` for progressive enhancement — these work without client-side JavaScript and layer on optimistic UI when it's available.
- Revalidate affected loader data after mutations; React Router automatically revalidates loaders on the current route tree after an action completes, so avoid manually refetching in `useEffect`.
- Handle validation errors as typed action data (`return { errors }`) instead of throwing generic exceptions — reserve thrown `Response`s for actual HTTP-status failures (404, 401, 500).
- Keep server-only secrets and privileged operations (API keys, direct DB writes) out of client actions — perform them in `.server.ts` modules that are stripped from the client bundle.

```tsx
// Optimistic UI with useFetcher
function LikeButton({ postId, liked }: { postId: string; liked: boolean }) {
  const fetcher = useFetcher();
  const optimisticLiked = fetcher.formData
    ? fetcher.formData.get("liked") === "true"
    : liked;

  return (
    <fetcher.Form method="post" action={`/posts/${postId}/like`}>
      <input type="hidden" name="liked" value={String(!optimisticLiked)} />
      <button type="submit">{optimisticLiked ? "Unlike" : "Like"}</button>
    </fetcher.Form>
  );
}
```

## Navigation and State

- Store shareable state in URL params or search params (filters, pagination, selected tab) so links and reloads reproduce the same view.
- Keep ephemeral UI state (open/closed dropdown, hover state) local to components with `useState`.
- Use pending navigation state (`useNavigation()`, `fetcher.state`) to show optimistic or loading UI instead of a separate global spinner store.
- Avoid global state (Redux/Zustand/Context) for data that belongs to route loaders — it duplicates the router's own cache and goes stale.

## TypeScript and Testing

- Type loader and action return values; use the generated `Route.LoaderArgs`/`Route.ActionArgs` types (from `./+types/<route>`) for full inference of params and return types.
- Add tests for route loaders, actions, validation failures, and error boundaries as isolated units, independent of full component rendering.
- Use integration tests (e.g. Playwright, Testing Library with a router harness) for critical form and navigation flows.
- Mock network and persistence at the route-service boundary (`app/services/*.server.ts`) rather than mocking `fetch` deep inside components.

## Common Mistakes

- Duplicating loader fetches in `useEffect` — this causes a loading waterfall and defeats the purpose of route-level data loading.
- Mutating data directly inside a loader — loaders can run multiple times (revalidation, prefetch) and must stay read-only.
- Hiding route errors behind a single generic app-level catch-all `ErrorBoundary` instead of route-scoped boundaries that preserve navigation and layout.
- Putting auth checks only in components (e.g. redirecting from a `useEffect`) when the underlying loader data is protected — check and redirect inside the loader itself so no protected data is ever fetched or flashed.
