---
name: rtl-internationalization
description: "Right-to-left (RTL) layout and bidirectional text support for Hebrew, Arabic, Persian, and Urdu interfaces. Use when building UI that must support RTL languages, converting physical CSS/Tailwind properties to logical ones, handling mixed-direction (bidi) text, mirroring directional icons, or testing an app with dir=\"rtl\"."
---

# RTL (Right-to-Left) Internationalization

Supporting RTL languages such as Hebrew, Arabic, Persian, and Urdu requires building layouts with direction-agnostic CSS and treating text direction as a first-class, runtime-switchable property rather than a fixed assumption baked into left/right styles.

## Workflow for Making a UI RTL-Safe

1. **Audit for physical properties** — Search the codebase for `margin-left/right`, `padding-left/right`, `left`/`right` positioning, `text-align: left/right`, and Tailwind classes like `ml-*`, `mr-*`, `pl-*`, `pr-*`, `left-*`, `right-*`.
2. **Convert to logical properties** — Replace each physical property/class with its logical equivalent (`margin-inline-start`, `ms-*`, etc.) so the browser flips it automatically based on `dir`.
3. **Set direction at the root** — Apply `dir="rtl"` (or `dir="auto"`) on `<html>` or a top-level container, driven by the active locale, not hardcoded per component.
4. **Handle mixed-script text** — Wrap user-generated or interpolated strings that may differ in direction from the surrounding text with `<bdi>`.
5. **Audit icons and directional components** — Mirror arrows, chevrons, and back/forward buttons; leave non-directional icons (search, settings, home) unmirrored; reverse carousels, sliders, and progress bars.
6. **Externalize all strings** — Route every user-facing string through a translation function and use `Intl` APIs for numbers, currency, and dates instead of manual formatting.
7. **Test in RTL** — Render the app with `dir="rtl"` and real RTL content (not just mirrored LTR placeholder text) and visually verify every screen.

## Logical CSS Properties

Always use CSS logical properties instead of physical ones — logical properties resolve to the correct physical side automatically based on the element's `direction` and `writing-mode`, so the same stylesheet works for LTR and RTL without duplication.

| Physical (avoid) | Logical (use) |
|---|---|
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| `left` | `inset-inline-start` |
| `right` | `inset-inline-end` |
| `border-left` | `border-inline-start` |
| `border-right` | `border-inline-end` |
| `text-align: left` | `text-align: start` |
| `text-align: right` | `text-align: end` |

```css
/* Before: breaks in RTL */
.card {
  margin-left: 1rem;
  padding-right: 1.5rem;
  border-left: 2px solid var(--border);
  text-align: left;
}

/* After: direction-agnostic */
.card {
  margin-inline-start: 1rem;
  padding-inline-end: 1.5rem;
  border-inline-start: 2px solid var(--border);
  text-align: start;
}
```

## Tailwind CSS Logical Classes

Use Tailwind's logical utilities instead of the directional (`l`/`r`) ones:

- `ms-4` instead of `ml-4` (margin-inline-start)
- `me-4` instead of `mr-4` (margin-inline-end)
- `ps-4` instead of `pl-4` (padding-inline-start)
- `pe-4` instead of `pr-4` (padding-inline-end)
- `start-0` instead of `left-0` (inset-inline-start)
- `end-0` instead of `right-0` (inset-inline-end)
- `text-start` instead of `text-left`
- `text-end` instead of `text-right`

```html
<!-- Before -->
<div class="ml-4 pr-6 text-left">...</div>

<!-- After -->
<div class="ms-4 pe-6 text-start">...</div>
```

## React Native Logical Properties

React Native's Yoga layout engine also supports logical properties — use them instead of `Left`/`Right` variants:

- `paddingStart` / `paddingEnd` instead of `paddingLeft` / `paddingRight`
- `marginStart` / `marginEnd` instead of `marginLeft` / `marginRight`
- `borderStartWidth` / `borderEndWidth` instead of `borderLeftWidth` / `borderRightWidth`
- `start` / `end` instead of `left` / `right` in absolute positioning

```tsx
const styles = StyleSheet.create({
  row: {
    flexDirection: 'row',
    paddingStart: 16,
    paddingEnd: 16,
  },
  badge: {
    position: 'absolute',
    end: 8, // flips automatically with I18nManager.isRTL
    top: 8,
  },
})
```

## Bidirectional Text Safety

Wrap text whose direction may differ from its surrounding context — usernames, product codes, file paths, or any user-generated content interpolated into a translated sentence — with `<bdi>` so the browser's bidi algorithm isolates it correctly.

```html
<p>User <bdi>{userName}</bdi> posted a comment</p>
<p>Order number: <bdi>{orderCode}</bdi> has shipped</p>
```

Without `<bdi>`, a Latin-script username embedded in an Arabic sentence (or vice versa) can visually reorder surrounding punctuation and neighboring words.

## Directional Icons and Components

- Mirror directional icons in RTL mode: arrows, chevrons, back/forward/next/previous buttons, and the "reply" icon in mail/chat UIs.
- Do **not** mirror non-directional icons: home, settings, search, print, trash, checkmarks, and icons of real-world objects (a clock face, a phone).
- Reverse the direction of carousels, sliders, and progress bars in RTL so "next" still corresponds to the reading direction.
- Reverse swipe gesture direction in RTL (e.g. a swipe-to-dismiss card should swipe toward the trailing edge, not always left).
- Flip charts and graphs that have a directional axis (e.g. a timeline running left-to-right) so time still flows in reading order.

```css
/* Mirror an icon only when the document is RTL */
[dir="rtl"] .icon-chevron-forward {
  transform: scaleX(-1);
}
```

```tsx
// React: pick the mirrored icon variant based on direction
function BackButton({ dir }: { dir: 'ltr' | 'rtl' }) {
  const Icon = dir === 'rtl' ? ChevronRightIcon : ChevronLeftIcon
  return (
    <button aria-label="Back">
      <Icon aria-hidden="true" />
    </button>
  )
}
```

## Internationalization Fundamentals

- Never hardcode user-facing strings — route every string through a translation function (`t()`, `intl.formatMessage()`, etc.) so it can be localized and so RTL locales get correct punctuation and word order.
- Use `Intl.NumberFormat` for numbers and currency instead of manual string formatting — digit shaping and grouping separators differ by locale.
- Use `Intl.DateTimeFormat` for dates and times instead of manual concatenation.
- Set `dir="auto"` on user-generated content fields (comments, chat messages) so the browser infers direction from content, and set an explicit `dir="rtl"` or `dir="ltr"` on the document root based on the active locale.

```ts
const currency = new Intl.NumberFormat('ar-EG', {
  style: 'currency',
  currency: 'EGP',
}).format(1234.5)
// "١٬٢٣٤٫٥٠ ج.م.‏"

document.documentElement.dir = locale.startsWith('ar') || locale.startsWith('he') ? 'rtl' : 'ltr'
document.documentElement.lang = locale
```

## Testing

- Always test with `dir="rtl"` applied to the root element, not just a mirrored screenshot of the LTR layout.
- Verify with actual RTL content (real Arabic/Hebrew strings, which are typically shorter or longer than their English equivalents) rather than reversed LTR placeholder text — string length differences surface truncation and overflow bugs that mirroring alone won't catch.
- Include RTL snapshots or visual regression tests for key screens (navigation, forms, data tables, modals) alongside the LTR versions.
- Check that focus order and keyboard navigation (Tab order) follow the visual reading direction in RTL, since some frameworks don't reverse DOM/tab order automatically.
- Verify third-party components (date pickers, charts, rich text editors) respect `dir` — many popular libraries default to LTR-only and need an explicit RTL mode or CSS override.

## Common Mistakes

- Using physical CSS/Tailwind properties (`ml-*`, `left`, `text-align: left`) that silently break when a locale switches to RTL.
- Mirroring icons that shouldn't be mirrored (e.g. a clock or a "play" triangle), which reads as visually wrong to native RTL users.
- Concatenating translated strings with hardcoded punctuation or ordering (e.g. `name + ": " + value`) instead of using ICU message formatting, which breaks word order in RTL locales.
- Assuming `dir="rtl"` alone fixes everything — inline styles, canvas-drawn UI, and absolutely-positioned elements using `left`/`right` still need manual conversion to logical properties.
