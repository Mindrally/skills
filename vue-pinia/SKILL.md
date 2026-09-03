---
name: vue-pinia
description: "Vue 3 state management with Pinia using the Composition API, covering setup stores, SSR/Nuxt integration, persistence, and testing. Use when creating or refactoring Pinia stores, deciding what state belongs in a store versus a component, wiring stores into router guards, persisting store state to storage, or testing store actions with @pinia/testing."
---

# Vue + Pinia

Pinia is the standard state management library for Vue 3, and setup stores built with the Composition API give shared application state the same ergonomics as local `ref`/`computed` state.

## Workflow for Adding a Pinia Store

1. **Decide if Pinia is the right tool** — Confirm the state is shared across routes/components, not server cache data, and not purely local UI state (see State Ownership below).
2. **Create the store file** — Add one file per domain under `stores/`, e.g. `stores/cart.ts`, using `defineStore('cart', () => { ... })`.
3. **Define state, getters, and actions** — Declare state with `ref`/`reactive`, derive values with `computed`, and put all writes/side effects in functions (actions).
4. **Return everything from the setup function** — Return every piece of state, every getter, and every action so Pinia devtools, SSR, and plugins can track them.
5. **Consume in components** — Call the store at the top of `<script setup>`, destructure reactive state with `storeToRefs()`, and destructure actions directly.
6. **Handle SSR/router-guard usage** — Call the store from inside setup, a getter, an action, or pass the active Pinia instance explicitly outside setup contexts.
7. **Add persistence and tests** — Allowlist persisted fields, validate rehydrated data, and test actions directly with `@pinia/testing`.

## State Ownership

- Keep component-only state in the component with `ref`, `reactive`, or `computed`.
- Use Pinia for shared client state that spans routes, layouts, or unrelated component trees.
- Use route params and query strings for shareable navigation state — don't duplicate it into a store.
- Use Nuxt `useFetch` / `useAsyncData`, TanStack Query for Vue, or the existing API layer for server state — Pinia is for client state, not a data-fetching cache.
- Do not copy server cache data into Pinia unless it is an intentional editable draft, offline cache, or workflow snapshot with its own lifecycle.

## Store Structure

- Prefer setup stores with `defineStore('name', () => { ... })` for Vue 3 Composition API projects over the older options-API `defineStore({ state, getters, actions })` form.
- Use one store file per domain under `stores/`, such as `stores/cart.ts` or `stores/session.ts`.
- Keep state, computed getters, and actions together when they represent one cohesive domain.
- Return every state property from setup stores so Pinia can track it for devtools, SSR, and plugins — an un-returned `ref` is invisible to the rest of the system.
- Keep getters pure and side-effect free; put writes, I/O, and orchestration in actions.

```ts
// stores/cart.ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

interface CartLine {
  id: string
  name: string
  quantity: number
  unitPrice: number
}

export const useCartStore = defineStore('cart', () => {
  const lines = ref<CartLine[]>([])

  const itemCount = computed(() =>
    lines.value.reduce((total, line) => total + line.quantity, 0),
  )

  const subtotal = computed(() =>
    lines.value.reduce((total, line) => total + line.quantity * line.unitPrice, 0),
  )

  function addLine(line: CartLine) {
    const existing = lines.value.find((item) => item.id === line.id)
    if (existing) {
      existing.quantity += line.quantity
      return
    }
    lines.value.push(line)
  }

  function removeLine(id: string) {
    lines.value = lines.value.filter((line) => line.id !== id)
  }

  function clearCart() {
    lines.value = []
  }

  return {
    lines,
    itemCount,
    subtotal,
    addLine,
    removeLine,
    clearCart,
  }
})
```

## Component Usage

- Call stores at the top of `<script setup>` or inside setup functions, getters, and actions — never conditionally, and never inside a plain `.js`/`.ts` module loaded at import time.
- Use `storeToRefs()` when destructuring store state or getters in components, so reactivity is preserved.
- Destructure actions directly when useful; actions remain bound to the store and don't need `storeToRefs()`.
- Avoid writing large business workflows in components; move them to store actions or composables so they're testable and reusable.
- Prefer computed values over watchers when deriving state — a `watch` that just recomputes a value should usually be a `computed`.

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCartStore } from '@/stores/cart'

const cart = useCartStore()
const { itemCount, subtotal } = storeToRefs(cart)
const { clearCart } = cart
</script>

<template>
  <div>
    <p>{{ itemCount }} items — ${{ subtotal.toFixed(2) }}</p>
    <button type="button" :disabled="itemCount === 0" @click="clearCart">
      Clear cart
    </button>
  </div>
</template>
```

## TypeScript

- Type store state, action payloads, and API responses explicitly.
- Avoid `any`; use `unknown` and narrow external inputs (API responses, persisted storage) before committing them to state.
- Use interfaces for object state that is shared across components or API boundaries.
- Prefer discriminated unions for workflow status and error state, e.g. `{ status: 'idle' } | { status: 'loading' } | { status: 'error'; error: string }`.
- Keep store IDs (the first argument to `defineStore`) stable and descriptive because they appear in devtools and persistence keys.

## Actions and Side Effects

- Actions may be sync or async; keep each action focused on one user or domain workflow.
- Validate action inputs at the boundary before mutating store state.
- Represent async workflows with explicit `status`, `error`, and `lastUpdatedAt` fields when the UI depends on them.
- Reset stale errors before retrying an async action so a previous failure doesn't linger in the UI.
- Keep subscriptions, intervals, sockets, and browser listeners outside stores unless the store owns their lifecycle and cleanup (e.g. an explicit `connect()`/`disconnect()` action pair).

## SSR, Nuxt, and Router Guards

- In SSR contexts, use the store inside setup, getters, or actions so Pinia can resolve the active app instance.
- When using a store outside setup, such as in a router guard, pass the active Pinia instance if the framework requires it (Nuxt's auto-imports handle this automatically).
- Do not read browser-only storage (`localStorage`, `window`) during server rendering — guard with `import.meta.client` (Nuxt) or an `onMounted` check.
- In Nuxt, prefer the `@pinia/nuxt` integration and SSR-safe composables for data fetching instead of manually wiring a Pinia instance.
- Avoid singleton state leaks across requests by relying on the framework-created Pinia instance per request rather than a module-level singleton.

```ts
// router guard example (outside setup)
router.beforeEach((to) => {
  const auth = useAuthStore(pinia) // pass the active Pinia instance explicitly
  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    return { name: 'login' }
  }
})
```

## Persistence

- Persist only the fields that must survive reloads, such as preferences or incomplete local drafts.
- Never persist secrets, access tokens, refresh tokens, raw PII, or authorization decisions in browser storage.
- Use field allowlists and versioned migrations for persisted store schemas so old persisted shapes don't crash a new store version.
- Treat persisted state as untrusted input and validate it before using it for critical workflows — a user can edit `localStorage` directly.
- Account for hydration timing before rendering UI that depends on persisted values, especially under SSR where the server has no access to browser storage.

## Testing and Tooling

- Use `@pinia/testing` (`createTestingPinia()`) for component tests that need stores, with actions stubbed by default.
- Test store actions directly for domain behavior and edge cases, independent of any component.
- Reset Pinia between tests to avoid shared state leakage across test cases.
- Add HMR support with `acceptHMRUpdate()` in stores when the project uses Vite HMR patterns.
- Keep stores easy to inspect in Vue Devtools by using clear state names and focused, single-domain stores.

```ts
import { setActivePinia, createPinia } from 'pinia'
import { describe, it, expect, beforeEach } from 'vitest'
import { useCartStore } from '@/stores/cart'

describe('cart store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('merges quantities for duplicate line items', () => {
    const cart = useCartStore()
    cart.addLine({ id: '1', name: 'Widget', quantity: 1, unitPrice: 10 })
    cart.addLine({ id: '1', name: 'Widget', quantity: 2, unitPrice: 10 })
    expect(cart.itemCount).toBe(3)
  })
})
```

## Anti-Patterns

- Do not use Pinia as a dumping ground for every reactive value — most component state should stay local.
- Do not destructure state directly from a store without `storeToRefs()`; this breaks reactivity silently.
- Do not mutate props or route objects through store actions.
- Do not put server-only objects, request instances, DOM nodes, or timers in store state — they don't serialize and break SSR/devtools.
- Do not create circular reads between stores in setup functions; compose stores through actions or computed values instead of having two stores read each other at module scope.
