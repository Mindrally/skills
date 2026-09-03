---
name: swiftui-development
description: "Guidelines for building SwiftUI applications with MVVM/Clean Architecture, Swift 6 strict concurrency, and modern state management. Use when building SwiftUI views, structuring view models, managing state with @State/@Observable, handling async data flow, or optimizing SwiftUI view performance."
---

# SwiftUI Development

This skill covers best practices for building clear, performant, and maintainable SwiftUI applications, including architecture, state management, Swift 6 concurrency, layout, animation, and testing.

## Key Principles

- Write correct, up-to-date, bug-free, fully functional, secure, and performant code
- Favor readability, but not at the expense of correctness or safety
- Fully implement all requested functionality — leave no TODOs, placeholders, or missing pieces
- Target Swift 6.0+ with strict concurrency checking enabled; treat concurrency warnings as errors
- Target iOS 17+ where possible to use modern APIs (`@Observable`, `NavigationStack`, `#Preview`); note the minimum deployment target when it forces older patterns

## Architecture

- Use MVVM (Model-View-ViewModel) or Clean Architecture; introduce a Coordinator layer for complex, multi-screen navigation flows
- Apply SOLID principles to views and view models:
  - **Single Responsibility** — each view or view model has one reason to change
  - **Open/Closed** — extend behavior via composition and protocols rather than modifying existing types
  - **Liskov Substitution** — protocol conformances must be substitutable without surprising callers
  - **Interface Segregation** — keep protocols small and focused rather than one large "do everything" interface
  - **Dependency Inversion** — depend on protocol abstractions, not concrete types, and inject dependencies via `init`
- Implement protocol-oriented programming; prefer structs over classes for data models
- Use extensions for code organization and separation of concerns

## SwiftUI View Structure

- Keep views small and focused on a single responsibility
- Treat 50 lines as a practical ceiling for a `body` property — past that, extract subviews into small, reusable structs or private extension functions
- Use `@ViewBuilder` for custom container views and complex conditional view logic
- Implement proper view composition patterns

## State Management

- `@State` — local, value-type view state (Bool, Int, String, small structs)
- `@Binding` — two-way data binding with child views
- `@StateObject` — use only in the view that creates/owns the object's lifecycle
- `@ObservedObject` — use in child views that react to changes but don't own the object
- `@EnvironmentObject` / `@Environment` — use sparingly for broadly shared dependencies or system values; prefer explicit dependency injection via `init` when a dependency belongs to a specific view hierarchy rather than the whole app
- `@Published` — for observable properties on `ObservableObject` classes (pre-iOS 17 or when `ObservableObject` is otherwise required)
- `@Observable` macro (iOS 17+) — prefer this over `ObservableObject`/`@Published` for new view models; it removes the need for `@Published` on every property and only triggers view updates for properties actually read by that view, which reduces unnecessary redraws

## Naming Conventions

- camelCase for variables, functions, and methods; PascalCase for types (classes, structs, enums, protocols)
- Use descriptive, verbose names — `fetchUserData` over `getData`
- Prefix boolean variables with `is`, `has`, `should`, etc.
- Use verb phrases for function names

## SwiftUI Best Practices

- Use SF Symbols for system icons; use semantic colors from the asset catalog for automatic dark mode support
- Support Dynamic Type and implement proper keyboard avoidance
- Use `NavigationStack` (iOS 16+) over the deprecated `NavigationView`
- Prefer type-inferred shorthand dot syntax where available (e.g., `.background(.blue)`)
- Add `.contentShape(Rectangle())` to `HStack`/`VStack` rows that have transparent backgrounds, so taps register across the whole row rather than only on opaque content

## Layout and Styling

- Use native SwiftUI layout containers (VStack, HStack, ZStack, Grid); use `LazyVStack`/`LazyHStack` for dynamic or long lists
- Use `GeometryReader` sparingly — it expands to fill all available space and can hurt layout performance; prefer `.background(GeometryReader { ... })` scoped to a single view, or the `Layout` protocol for custom layouts
- Give list items stable, meaningful `id`s (from `Identifiable`) rather than `\.self`
- Implement adaptive layouts for different screen sizes
- Use `ViewModifier`s for reusable styling; create custom `ButtonStyle`, `TextFieldStyle`, etc.

## Animations and Transitions

- Prefer `.animation(_:value:)` scoped to a specific state value over broad `withAnimation` calls, especially inside `body`
- Reserve `withAnimation` for animations explicitly triggered by user interaction
- Implement custom transitions using `AnyTransition`; use `matchedGeometryEffect` for hero animations
- Use `TimelineView` for high-frequency, time-driven visual updates instead of a `Timer` + published property

## Swift 6 Concurrency & Data Flow

- Annotate view models with `@MainActor`; all UI updates must happen on the main actor
- Prefer `.task(id:)` over `.onAppear` for starting async work — it automatically cancels the task when the view disappears or the id changes, avoiding orphaned work
- Use `async`/`await` for asynchronous operations and `Result` (or typed throws) for error handling
- Prefer `actor` types for shared mutable state or services accessed from multiple tasks
- Mark pure logic functions `nonisolated` when they don't touch the main actor, to avoid unnecessary hops
- Never block the main thread — move heavy computation to a detached `Task`
- Handle loading, error, and success states explicitly rather than leaving implicit/undefined states

## Memory Management & Safety

- Default to `[weak self]` in closures that outlive the current scope; use `guard let self else { return }` at the start of async closures
- Only use `[unowned self]` when the closure's lifetime is provably shorter than `self`'s
- Handle optionals safely — no force unwrapping; use `guard` for early returns
- For remote images, use `AsyncImage` (with a caching layer, or a library like Nuke/Kingfisher for production apps) and apply `.resizable()` immediately

## Performance Optimization

- Minimize view body recalculations; adopt `Equatable` conformance where it helps SwiftUI skip redundant diffing
- Implement proper list diffing with `Identifiable` items and stable ids
- Profile with Instruments before optimizing; cache expensive computations rather than recomputing them in `body`

## Accessibility

- Add accessibility labels, hints, and traits appropriately; support VoiceOver
- Assign distinct `accessibilityIdentifier` strings to interactive elements so UI tests can target them reliably
- Test with accessibility features (Dynamic Type, VoiceOver, Reduce Motion) enabled

## Testing and Previews

- Always provide a preview using `#Preview` (Xcode 15+) or `PreviewProvider` on older toolchains, injecting realistic mock data
- Preview in multiple color schemes and device sizes
- Structure unit tests with Given-When-Then; generate protocol-based mocks for external dependencies so view models can be tested without hitting real services

## Code Quality

- Write self-documenting code; add comments only for non-obvious logic
- Follow the Swift API Design Guidelines
- Use `MARK: - Section Name` to organize longer files, and place private helpers in a `private extension`

## Common Patterns

### View with @Observable ViewModel (iOS 17+)
```swift
@Observable
@MainActor
final class ContentViewModel {
    var items: [Item] = []
    var isLoading = false

    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        // Load items
    }
}

struct ContentView: View {
    @State private var viewModel = ContentViewModel()

    var body: some View {
        List(viewModel.items) { item in
            Text(item.name)
        }
        .task(id: viewModel.items.count) {
            await viewModel.loadItems()
        }
    }
}
```

### View with ObservableObject ViewModel (pre-iOS 17 / legacy)
```swift
struct ContentView: View {
    @StateObject private var viewModel = ContentViewModel()

    var body: some View {
        // View implementation
    }
}

@MainActor
class ContentViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false

    func loadItems() async {
        isLoading = true
        // Load items
        isLoading = false
    }
}
```

### Reusable View Modifier
```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(Color(.systemBackground))
            .cornerRadius(12)
            .shadow(radius: 4)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}
```
</content>
