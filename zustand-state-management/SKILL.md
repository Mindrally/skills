---
name: zustand-state-management
description: "Best practices for Zustand state management in React and TypeScript applications, covering store design, selectors, middleware, SSR, and testing. Use when creating or refactoring Zustand stores, deciding whether state belongs in Zustand vs component state vs a server-state library, optimizing selector re-renders, adding persist/devtools/immer middleware, or handling Zustand with SSR/React Server Components."
---

# Zustand State Management

This skill covers designing and using Zustand stores in React and TypeScript applications, including state ownership decisions, store slicing, selector performance, middleware, SSR, and testing.

## State Ownership

- Keep ephemeral UI state in the nearest component with `useState` or `useReducer` — don't reach for Zustand by default.
- Use URL state for shareable filters, pagination, tabs, and search params.
- Use Zustand only for client state that is genuinely shared across unrelated components (auth session view state, command palette, cart draft, editor state, etc.).
- Use TanStack Query, SWR, RTK Query, or the project's existing data layer for server state — don't duplicate fetched server data into a Zustand store unless there's a documented offline or draft-editing requirement.
- Implement functional, declarative patterns; avoid classes. Use descriptive variable names with auxiliary verbs like `isLoading`, `hasError`.

## Store Design

- Model each store as state plus named actions; avoid exposing anonymous setters for component code to misuse.
- Keep stores small and domain-focused rather than one global store for the entire application.
- Split large stores into typed slices, then apply middleware only at the composed store boundary.
- Keep derived values as selectors or small pure helpers unless they must be cached in state.
- Store serializable data by default; keep DOM nodes, promises, sockets, and timers outside store state.

```typescript
import { create } from 'zustand'

interface SidebarState {
  isOpen: boolean
  activePanelId: string | null
}

interface SidebarActions {
  openPanel: (panelId: string) => void
  close: () => void
  toggle: () => void
}

type SidebarStore = SidebarState & SidebarActions

export const useSidebarStore = create<SidebarStore>()((set) => ({
  isOpen: false,
  activePanelId: null,
  openPanel: (panelId) => set({ isOpen: true, activePanelId: panelId }),
  close: () => set({ isOpen: false, activePanelId: null }),
  toggle: () => set((state) => ({ isOpen: !state.isOpen })),
}))
```

## Component Usage & Performance

- Subscribe to the smallest possible slice: `useStore((state) => state.value)`. Avoid selecting the entire store.
- Do not call a store hook without a selector in components unless the component truly needs every field.
- Use `useShallow` (from `zustand/react/shallow`) when a selector returns an object or tuple of multiple values, to avoid re-renders from a new object reference each call.
- Keep selectors pure and cheap; move expensive derivations into memoized helpers.

```tsx
import { useShallow } from 'zustand/react/shallow'
import { useSidebarStore } from '@/stores/sidebar-store'

export function SidebarToggle() {
  const { isOpen, toggle } = useSidebarStore(
    useShallow((state) => ({
      isOpen: state.isOpen,
      toggle: state.toggle,
    })),
  )

  return (
    <button type="button" aria-expanded={isOpen} onClick={toggle}>
      Toggle sidebar
    </button>
  )
}
```

## TypeScript

- Define explicit state and action interfaces for shared stores; combine them (`type Store = State & Actions`) rather than one flat interface for non-trivial stores.
- Avoid `any`; use `unknown` plus narrowing for external data.
- Type action payloads and return values, including async actions.
- Prefer discriminated unions for complex local status instead of several loosely related booleans.
- Export store state types when tests, utilities, or vanilla store factories need them.

## Middleware

### Updates
- Use functional `set((state) => nextState)` when the next value depends on current state.
- Treat nested state immutably; install and use the `immer` middleware only when it materially simplifies nested updates — don't mutate nested objects directly without it.

### Persistence
```typescript
import { persist } from 'zustand/middleware'

const useStore = create(
  persist(
    (set) => ({
      // state and actions
    }),
    { name: 'store-key' }
  )
)
```
- Use `persist` only for state that must survive reloads.
- Use `partialize`, `version`, and `migrate` when persisting anything beyond trivial preferences.
- Never persist secrets, access tokens, refresh tokens, raw PII, or other long-lived authorization state to browser storage.

### DevTools
```typescript
import { devtools } from 'zustand/middleware'

const useStore = create(
  devtools((set) => ({
    // state and actions
  }))
)
```
- Use `devtools` in development for complex flows and give important actions clear names so they're identifiable in the trace.

### Non-React Subscriptions
- Use `subscribeWithSelector` for non-React subscriptions that need fine-grained updates outside of component rendering.

## Async Actions & Error Handling

- Async store actions should coordinate client-only workflows, optimistic drafts, or local device APIs — keep HTTP fetching in the project's server-state layer unless the state is explicitly client-owned.
- Represent async client workflows with explicit statuses such as `idle`, `pending`, `success`, and `error` rather than ad hoc booleans.
- Handle errors at function start using early returns and guard clauses; use try-catch in async actions and provide meaningful error messages.
- Reset error state deliberately when retrying or closing a workflow.

## SSR and React Server Components

- Do not read or mutate browser-only stores from React Server Components.
- In SSR frameworks (Next.js App Router, etc.), create per-request vanilla stores when state must be initialized on the server, rather than a shared module-level store.
- Guard persisted stores against hydration mismatches before rendering storage-backed values.
- Keep store modules free of direct `window`, `document`, and storage access outside middleware configuration.

## Testing

- Test stores independently of components — test store actions directly without rendering React when possible.
- Reset stores between tests with their initial state.
- Assert selectors and actions separately from component behavior; mock Zustand stores in component tests when isolating UI logic.
- Mock server-state libraries instead of routing fetched data through Zustand for tests.
- Test middleware behavior (persistence, devtools) separately from core store logic.

## Anti-Patterns

- Do not create one global store for the entire application.
- Do not put form input state in Zustand unless multiple distant components edit the same draft.
- Do not mutate nested objects directly without Immer middleware.
- Do not use Zustand as an event bus; prefer explicit callbacks, services, or a scoped store.
- Do not introduce Redux-style reducers, action constants, or dispatch wrappers unless the project already uses that pattern.
