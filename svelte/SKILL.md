---
name: svelte
description: "Modern Svelte 5 and SvelteKit development covering runes-based reactivity, snippets, SSR/SSG, and state management, including migration notes from legacy Svelte 4 syntax. Use when writing or reviewing Svelte components, deciding between runes and legacy reactive statements, building SvelteKit routes and load functions, or migrating a Svelte 4 codebase to Svelte 5."
---

# Svelte / SvelteKit

This skill covers Svelte and SvelteKit development with deep knowledge of SSR, SSG, and modern web patterns, including the Svelte 5 runes system and how it differs from legacy Svelte 4 reactivity.

## Core Principles

- Write concise, technical components with accurate SvelteKit examples
- Emphasize SSR/SSG capabilities and performance optimization
- Use TypeScript with proper naming conventions
- Prioritize minimal client-side JavaScript through server-side rendering
- Follow functional and declarative programming patterns

## Svelte 5 Runes

Svelte 5 introduces runes — compiler-level primitives that make reactivity explicit, replacing Svelte 4's implicit `let`/`$:` reactivity.

- Use `$state` for reactive state (replaces plain `let` bindings that needed reactivity)
- Use `$derived` for computed values (replaces `$: computed = ...`)
- Use `$effect` for side effects (replaces `$: { ... }` blocks)
- Use `$props` for component properties (replaces `export let prop`)
- Use `$bindable` for two-way bindable props (replaces implicit `bind:` support on exported props)
- Use `$inspect` during development to log reactive value changes
- Runes are compiler syntax, not imports — never `import { $state } from 'svelte'`. Only import genuine utilities that still require it, such as `tick`, `untrack`, `mount`, or `unmount`.

### Svelte 4 vs. Svelte 5

**Svelte 4 (legacy reactivity):**
```svelte
<script>
  let count = 0;
  $: double = count * 2;
  $: {
    if (count > 10) alert('Too high!');
  }
</script>
<button on:click={() => count++}>{count} / {double}</button>
```

**Svelte 5 (runes):**
```svelte
<script>
  let count = $state(0);
  let double = $derived(count * 2);

  $effect(() => {
    if (count > 10) alert('Too high!');
  });
</script>
<button onclick={() => count++}>{count} / {double}</button>
```

Key differences to apply when writing or reviewing components:

1. **Reactivity is explicit** — `$state()` marks reactive variables, `$derived()` replaces `$:` for computed values, `$effect()` replaces `$: {}` blocks for side effects.
2. **Event handling is standardized** — Svelte 5 treats event handlers as ordinary HTML properties (`onclick={handler}`) instead of Svelte-specific directives (`on:click={handler}`). There are no more event modifiers like `|preventDefault`; call `e.preventDefault()` inline instead: `onclick={e => { e.preventDefault(); handler(e); }}`.
3. **Component props** use `let { propName } = $props()` instead of multiple `export let propName` declarations. Mark a prop bindable from the parent with `$bindable()`.
4. **Snippets replace slots** for most reusable markup — `{#snippet name(params)}...{/snippet}` and `{@render name(args)}` reduce duplication and are more composable than named slots. Existing `<slot>` usage still works but prefer snippets in new Svelte 5 code.

When maintaining an existing Svelte 4 codebase, keep using `export let`, `on:event`, and `$:` reactive statements consistently within that codebase rather than mixing paradigms — full-file migrations to runes should be deliberate, not incidental.

## Styling

- Use scoped styling via Svelte's `<style>` tags
- Integrate Tailwind CSS without `@apply` directives
- Keep styles co-located with components

## State Management

- Use Svelte stores (or exported `$state` in `.svelte.js`/`.svelte.ts` modules under Svelte 5) for shared state
- Implement state management via classes for complex scenarios, using `$state` fields for reactive class properties
- Leverage Svelte's reactivity system effectively rather than reaching for external state libraries by default

## SvelteKit Features

- Use file-based routing
- Implement proper load functions
- Handle errors appropriately
- Use form actions for mutations
- Leverage server-side data loading

## Performance

- Focus on Web Vitals optimization
- Minimize client-side JavaScript
- Use preloading for faster navigation
- Implement lazy loading where appropriate

## Internationalization

- Use Paraglide.js for i18n support
- Implement proper locale handling

## Testing

- Use Vitest for unit testing
- Use Lighthouse for performance auditing
- Write comprehensive component tests

## Accessibility

- Ensure accessibility compliance
- Use semantic HTML
- Implement proper ARIA attributes
