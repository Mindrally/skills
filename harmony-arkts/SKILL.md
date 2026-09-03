---
name: harmony-arkts
description: "Best practices for HarmonyOS application development with ArkTS and ArkUI, covering components, state management, resources, layout, lifecycle, and accessibility. Use when writing .ets component files, using ArkUI decorators like @Component/@State/@Prop/@Link, building layouts with Column/Row/Stack/List, managing HarmonyOS app resources with $r(), or handling component lifecycle and accessibility on HarmonyOS."
---

# HarmonyOS ArkTS Development

This skill covers building HarmonyOS applications with ArkTS and the declarative ArkUI framework, including component structure, state management, layout, lifecycle, resources, and accessibility.

## Workflow for Building an ArkTS Component

1. **Define the component** — Create a `struct` decorated with `@Component`, named in PascalCase.
2. **Declare state near the top** — List `@State`, `@Prop`, `@Link`, and other state decorators first, before any methods.
3. **Add lifecycle hooks** — Implement `aboutToAppear()`/`aboutToDisappear()` (and others as needed) after state, before `build()`.
4. **Compose the UI in build()** — Place `build()` last, using ArkUI primitives (`Column`, `Row`, `Stack`, `List`, etc.) to lay out child components.
5. **Extract complex UI** — Once `build()` grows large or nested, pull sections into smaller `@Component` structs or `@Builder` functions.
6. **Wire events** — Attach arrow-function event handlers that delegate non-trivial logic to named methods on the component.
7. **Externalize resources** — Move strings, colors, dimensions, and images into the resource system and reference them with `$r()`.
8. **Verify accessibility and responsiveness** — Add accessibility labels, check touch target sizes, and test across representative device sizes/orientations.

## Component Structure

- Use `@Component` to define components and PascalCase for the component struct name (e.g., `struct UserProfileCard`).
- Keep state declarations (`@State`, `@Prop`, `@Link`, `@StorageLink`, plain fields) near the top of the component, before methods and lifecycle hooks.
- Group lifecycle hooks (`aboutToAppear`, `aboutToDisappear`, `onPageShow`, `onPageHide`) before `build()` so the component's lifecycle is easy to scan.
- Place `build()` last and keep it focused purely on UI composition — pull data transformation and business logic out into methods called from `build()`, not embedded inline.
- Extract complex UI into smaller `@Component` structs (or `@Builder` functions for lightweight, reusable UI fragments) once a `build()` method grows past a screenful or nests more than a few levels deep.

## State and Data Flow

- Use `@State` for state owned and mutated by the component itself.
- Use `@Prop` for one-way parent-to-child data — the child gets its own copy and local changes don't propagate back up.
- Use `@Link` only for intentional two-way binding where the child genuinely needs to mutate the parent's state (e.g., a custom form control).
- Keep derived values in computed helper methods rather than duplicating and manually syncing state (don't store both `items` and `itemCount` as separate `@State` fields when `itemCount` can be `this.items.length`).
- Avoid broad global state (`AppStorage`/`LocalStorage` used indiscriminately) unless the project has an established app-state pattern; prefer passing data down through props/links for anything not genuinely global.

### Example: A Component with State, Props, and Lifecycle

```typescript
@Component
export struct CounterCard {
  @Prop label: string = 'Counter';
  @State private count: number = 0;
  private maxCount: number = 99;

  aboutToAppear(): void {
    console.info(`CounterCard mounted with label=${this.label}`);
  }

  private increment = (): void => {
    if (this.count >= this.maxCount) {
      return;
    }
    this.count += 1;
  };

  private reset = (): void => {
    this.count = 0;
  };

  build() {
    Column({ space: 12 }) {
      Text(this.label)
        .fontSize(18)
        .fontWeight(FontWeight.Medium)

      Text(`${this.count}`)
        .fontSize(32)
        .fontColor($r('app.color.primary'))
        .accessibilityText(`Current count is ${this.count}`)

      Row({ space: 8 }) {
        Button('Increment')
          .onClick(this.increment)
          .accessibilityText('Increment counter')

        Button('Reset')
          .onClick(this.reset)
          .accessibilityText('Reset counter')
      }
    }
    .width('100%')
    .padding(16)
    .alignItems(HorizontalAlign.Center)
  }
}
```

## Layout and Styling

- Use `Column`, `Row`, `Stack`, `List`, and other ArkUI layout primitives intentionally based on the axis and stacking behavior actually needed, rather than defaulting to `Stack` and positioning everything absolutely.
- Group layout properties — width, height, alignment, layout weight (`layoutWeight`) — before visual properties (color, font, border) in the chained modifier calls, so structure reads before decoration.
- Use object notation for margin and padding when the sides differ (`.margin({ top: 8, bottom: 16, left: 12, right: 12 })`) rather than a single shorthand that implies uniform spacing.
- Use logical pixels (`vp`) consistently for sizes so layouts scale correctly across device pixel densities; avoid mixing raw pixel values.
- Use percentage strings (`'100%'`, `'50%'`) for sizes that should be relative to the parent container instead of hardcoded fixed values.
- Keep reusable spacing, colors, and typography values in the resource system (`$r('app.color.primary')`, `$r('app.float.spacing_md')`) when the project supports it, instead of repeating literals across components.

## Events and Lifecycle

- Use arrow functions for event handlers (`.onClick(() => this.handleTap())` or a bound class field like `private handleTap = () => {...}`) so `this` binds correctly without extra boilerplate.
- Keep event handlers themselves short — read the event, then delegate any non-trivial logic to a named method that can be tested and reasoned about independently.
- Handle async failures explicitly (`try`/`catch` around `await`) and surface user-facing errors (a toast, an inline error state) instead of letting a rejected promise disappear silently.
- Use lifecycle hooks (`aboutToAppear`, `aboutToDisappear`, `onPageShow`, `onPageHide`, `onBackPress`) only for setup/teardown that genuinely depends on the component or page's lifecycle — don't use them as a dumping ground for logic that could live in a regular method.

## Resources and Accessibility

- Use `$r()` to reference app resources (strings, colors, media, floats) instead of embedding literal values directly in component code.
- Group resource references consistently — organize `resources/base/element/string.json`, `color.json`, `float.json` by feature or screen so they stay navigable as the app grows.
- Add descriptive accessibility labels (`.accessibilityText()`, `.accessibilityDescription()`) and correct focus handling (`.focusable(true)`, `.tabIndex()`) for every interactive element.
- Maintain sufficient color contrast between text and background, and keep touch targets at least the platform-recommended minimum size (generally 40vp or larger) for anything tappable.
- Test on representative device sizes and orientations (phone, foldable, tablet, landscape/portrait) since ArkUI layouts that look correct on one form factor can break on another.

## Common Mistakes

- Burying business logic (data fetching, validation, complex branching) directly inside `build()` instead of in dedicated methods — `build()` should read like a UI tree, not a program.
- Using `@Link` two-way binding when a simple one-way `@Prop` would do, which makes data flow harder to trace and invites accidental mutation from child components.
- Hardcoding repeated strings, colors, and dimensions that belong in the resource system, making localization and theming changes require hunting through component code.
- Leaving debug `console.log`/`console.info` calls in production code paths, which adds noise and can leak internal state into logs.
