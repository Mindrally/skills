---
name: nuxtjs-vue-typescript
description: "Best practices for building Nuxt 3 and Vue 3 applications with TypeScript, the Composition API, and Tailwind CSS. Use when structuring Nuxt/Vue projects, writing composables, typing components and props, wiring up server routes and plugins, or optimizing Nuxt builds and Web Vitals."
---

# NuxtJS Vue TypeScript Development

Guidelines for building Nuxt 3 and Vue 3 applications with TypeScript, Shadcn Vue, Radix Vue, VueUse, and Tailwind.

## Code Style and Structure

- Write concise, technical TypeScript with accurate examples
- Employ composition API with declarative patterns; avoid options API
- Favor iteration and modularity over code duplication
- Use descriptive variable names with auxiliary verbs (isLoading, hasError)
- Organize files: exported component, composables, helpers, static content, types
- Keep component boundaries clear — a component should own one piece of UI/behavior; extract composables when logic grows beyond simple template glue

## Naming Conventions

- Directories: lowercase with dashes (components/auth-wizard)
- Components: PascalCase (AuthWizard.vue)
- Composables: camelCase (useAuthState.ts)

## TypeScript Usage

- Utilize TypeScript throughout; prefer types over interfaces, but use interfaces where you expect a shape to be extended or implemented (e.g. component prop contracts)
- Keep shared types close to the feature that owns them; only promote a type to a shared `types/` directory once a second feature needs it
- Avoid enums; use const objects instead
- Leverage Vue 3 with TypeScript, defineComponent, and PropType

## Syntax and Formatting

- Use arrow functions for methods and computed properties
- Minimize curly braces in conditionals
- Employ template syntax for declarative rendering

## UI and Styling

- Implement Shadcn Vue, Radix Vue, and Tailwind
- Design responsively with mobile-first Tailwind approach
- Keep Tailwind usage accessible and consistent with the project's design system rather than one-off utility soup

## Performance

- Leverage Nuxt's built-in optimizations
- Use Suspense for async components
- Implement lazy loading for routes and components
- Optimize images: WebP format, size data, lazy loading
- Review bundle size periodically (`nuxi analyze`) and use dynamic `import()` for heavy, rarely-used components

## Key Conventions

- VueUse for common composables — reach for them where they simplify reactivity, but avoid ones that hide state transitions you need to reason about explicitly
- Pinia for state management
- Optimize Web Vitals (LCP, CLS, FID)
- Use Nuxt's auto-imports feature

## Nuxt-Specific Guidelines

- Follow Nuxt 3 directory structure (pages/, components/, composables/, layouts/, plugins/, server/)
- Leverage auto-imports, file-based routing, server routes, plugins
- Use layouts (`layouts/default.vue`, `<NuxtLayout>`) to share page chrome instead of duplicating it per page
- Use `server/api/` routes (Nitro) for server-side endpoints, and `useRuntimeConfig()` for environment-specific config instead of hardcoding values
- Use useFetch and useAsyncData for data fetching
- Implement SEO with useHead and useSeoMeta

## Vue 3 Composition API Best Practices

- Use `<script setup>` syntax
- Leverage ref, reactive, and computed
- Use provide/inject for dependency injection
- Create custom composables for reusable logic
