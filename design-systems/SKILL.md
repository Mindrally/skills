---
name: design-systems
description: "Comprehensive design system guidelines for building consistent, accessible, and scalable component libraries. Use when creating design tokens, building component libraries, implementing design systems, setting up theming architecture, or establishing UI governance processes."
metadata:
  maintainer: Mindrally
  source: https://github.com/Mindrally/skills
---

# Design Systems Best Practices

Guidelines for creating and maintaining design systems that ensure visual consistency, accessibility compliance, and scalable component architecture across digital products.

## Workflow: Setting Up a Design System

1. **Audit existing UI** — Inventory current components, colors, typography, and spacing across the product to identify inconsistencies and reusable patterns.
2. **Define design tokens** — Establish primitive tokens (colors, spacing, typography, shadows) as CSS custom properties or a token format (e.g., Style Dictionary) that serves as the single source of truth.
3. **Build foundational components** — Create atomic components (Button, Input, Text, Icon) that consume design tokens and expose a clear props API with TypeScript types.
4. **Compose patterns and layouts** — Assemble atomic components into higher-order patterns (forms, cards, navigation) with documented usage guidelines.
5. **Set up documentation and Storybook** — Configure Storybook with stories for every component variant, state, and accessibility annotation. Publish as a living style guide.
6. **Integrate testing and governance** — Add visual regression tests (e.g., Chromatic), accessibility linting (e.g., axe-core), and a contribution/review process for new components.
7. **Ship and iterate** — Publish the library as a versioned package, gather consumer feedback, and run regular accessibility and performance audits.

## Foundation Elements

### Color System

- Define primary, secondary, and accent colors
- Include semantic colors for success, warning, error, info states
- Ensure all color combinations meet WCAG contrast requirements
- Document color usage guidelines and contexts
- Provide light and dark mode variants

### Typography

- Establish a type scale with consistent ratios
- Define font families for headings and body text
- Set line heights and letter spacing standards
- Document font weights and their usage
- Ensure readability across screen sizes

### Spacing System

- Define consistent spacing scale (4px, 8px, 16px, 24px, etc.)
- Create layout primitives for common patterns
- Document margin and padding conventions
- Ensure responsive spacing behavior
- Use CSS custom properties for maintainability

### Icons and Imagery

- Maintain consistent icon style and sizing
- Define icon grid and stroke weights
- Document icon naming conventions
- Optimize assets for web performance
- Provide multiple formats when needed (SVG, PNG)

## Component Architecture

### Component Structure

- Create atomic, reusable components
- Define clear component APIs (props/attributes)
- Document variants and states
- Ensure components are accessible by default
- Provide clear naming conventions

### Component States

- Default state
- Hover state
- Focus state (keyboard navigation)
- Active/pressed state
- Disabled state
- Loading state
- Error state

### Component Variants

- Size variants (small, medium, large)
- Color/theme variants
- Layout variants
- Contextual variants

## Accessibility Requirements

- Follow WCAG 2.1 AA guidelines minimum
- Use semantic HTML elements
- Provide ARIA labels where needed
- Ensure keyboard navigation
- Test with screen readers
- Maintain color contrast ratios
- Support reduced motion preferences

## Documentation Standards

### Component Documentation

- Purpose and use cases
- Props/API reference
- Code examples
- Do's and don'ts
- Accessibility notes
- Related components

### Pattern Documentation

- When to use
- Anatomy breakdown
- Behavior specifications
- Responsive considerations
- Edge cases

## Implementation Guidelines

### CSS Architecture

- Use CSS custom properties for tokens
- Implement utility classes for common patterns
- Follow BEM or similar naming convention
- Ensure specificity is manageable
- Support theming and customization

### Component Libraries

- Framework-agnostic when possible
- Tree-shakeable exports
- TypeScript support
- Comprehensive test coverage
- Storybook integration

## Governance

### Contribution Guidelines

- How to propose new components
- Review and approval process
- Versioning strategy
- Breaking change policy
- Deprecation process

### Maintenance

- Regular accessibility audits
- Performance monitoring
- Browser compatibility testing
- Documentation updates
- Community feedback incorporation

## Design Tokens

### Token Categories

- Colors
- Typography (font sizes, weights, line heights)
- Spacing
- Border radius
- Shadows
- Breakpoints
- Animation durations
- Z-index values

### Token Implementation

```css
:root {
  /* Colors */
  --color-primary-500: #0066cc;
  --color-neutral-100: #f5f5f5;

  /* Spacing */
  --space-1: 4px;
  --space-2: 8px;
  --space-4: 16px;

  /* Typography */
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
}
```

### Example: React Button Component Using Design Tokens

```tsx
import React from "react";

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "danger";
  size?: "sm" | "md" | "lg";
  loading?: boolean;
}

const sizeStyles: Record<string, React.CSSProperties> = {
  sm: { padding: "var(--space-1) var(--space-2)", fontSize: "var(--font-size-sm)" },
  md: { padding: "var(--space-2) var(--space-4)", fontSize: "var(--font-size-base)" },
  lg: { padding: "var(--space-3) var(--space-6)", fontSize: "var(--font-size-lg)" },
};

export const Button: React.FC<ButtonProps> = ({
  variant = "primary",
  size = "md",
  loading = false,
  disabled,
  children,
  ...props
}) => (
  <button
    style={{
      ...sizeStyles[size],
      backgroundColor: `var(--color-${variant}-500)`,
      color: "var(--color-neutral-100)",
      borderRadius: "var(--radius-md)",
      boxShadow: "var(--shadow-sm)",
      cursor: disabled || loading ? "not-allowed" : "pointer",
      opacity: disabled ? 0.5 : 1,
    }}
    disabled={disabled || loading}
    aria-busy={loading}
    {...props}
  >
    {loading ? "Loading…" : children}
  </button>
);
```

### Example: Storybook Configuration

```ts
// .storybook/preview.ts
import type { Preview } from "@storybook/react";
import "../src/tokens.css"; // import design tokens globally

const preview: Preview = {
  parameters: {
    a11y: { element: "#storybook-root" },
    backgrounds: {
      default: "light",
      values: [
        { name: "light", value: "#ffffff" },
        { name: "dark", value: "#1a1a1a" },
      ],
    },
  },
};

export default preview;
```

## Quality Assurance

- Visual regression testing
- Accessibility automated testing
- Cross-browser testing
- Performance benchmarking
- Component unit testing
- Integration testing

When evolving the design system, validate changes against WCAG 2.1 AA, run visual regression tests, and ensure token updates propagate correctly across all consuming applications.
