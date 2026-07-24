Language: English | [中文](../前端知识库/07-CSS深入与布局.md)

# CSS In Depth and Layout

---

## Table of Contents

1. [CSS Box Model](#css-box-model)
2. [BFC, IFC, FFC, and GFC](#bfc-ifc-ffc-and-gfc)
3. [Layout System Overview](#layout-system-overview)
4. [Flexbox In Depth](#flexbox-in-depth)
5. [Grid In Depth](#grid-in-depth)
6. [Responsive Design](#responsive-design)
7. [CSS Variables and Theming](#css-variables-and-theming)
8. [CSS Animation and Performance](#css-animation-and-performance)
9. [Selectors and Specificity](#selectors-and-specificity)
10. [Modern CSS Features](#modern-css-features)
11. [CSS Engineering](#css-engineering)
12. [Interview Self-check](#interview-self-check)
13. [Production Scenarios](#production-scenarios)

---

## CSS Box Model

The box model describes how an element occupies space:

- Content.
- Padding.
- Border.
- Margin.

```css
.card {
  box-sizing: border-box;
  width: 320px;
  padding: 16px;
  border: 1px solid #ddd;
}
```

`box-sizing: border-box` makes width include content, padding, and border. It is
usually easier for layout systems.

## BFC, IFC, FFC, and GFC

### BFC

A block formatting context is an isolated layout area for block boxes.

It can be created by:

- `overflow` not `visible`.
- `display: flow-root`.
- `position: absolute` or `fixed`.
- `display: flex` or `grid`.

Use cases:

- Contain floats.
- Prevent margin collapse.
- Isolate layout interactions.

```css
.container {
  display: flow-root;
}
```

### Other Contexts

- IFC: inline formatting context for inline text layout.
- FFC: flex formatting context.
- GFC: grid formatting context.

## Layout System Overview

Common layout choices:

- Normal flow: document layout and text content.
- Flexbox: one-dimensional alignment and distribution.
- Grid: two-dimensional page or component layout.
- Positioning: overlays, fixed headers, tooltips.
- Multi-column and container queries: specialized responsive layouts.

Good layout design starts from content and constraints, not from CSS tricks.

## Flexbox In Depth

Flexbox is one-dimensional. It lays items along a main axis and cross axis.

```css
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}
```

Important properties:

- Container: `display`, `flex-direction`, `justify-content`, `align-items`, `gap`.
- Item: `flex-grow`, `flex-shrink`, `flex-basis`, `align-self`, `order`.

Common pitfall:

`min-width: auto` can prevent flex items from shrinking. Use `min-width: 0` for
text truncation in flex children.

```css
.title {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

## Grid In Depth

Grid is two-dimensional and works well for page sections and complex component
layouts.

```css
.dashboard {
  display: grid;
  grid-template-columns: 240px minmax(0, 1fr);
  grid-template-rows: auto 1fr;
  gap: 16px;
}
```

Useful functions:

- `minmax()`.
- `repeat()`.
- `auto-fit` and `auto-fill`.
- `fr` units.

Flexbox vs Grid:

- Use Flexbox for content distribution along one axis.
- Use Grid when rows and columns both matter.

## Responsive Design

Responsive design combines fluid layout, media queries, flexible units, and
content-aware breakpoints.

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: clamp(12px, 2vw, 24px);
}
```

Production practice:

- Prefer content breakpoints over device-name breakpoints.
- Test text expansion and localization.
- Preserve touch target sizes.
- Avoid layout shifts during image or font loading.

## CSS Variables and Theming

CSS variables cascade and can be updated at runtime.

```css
:root {
  --color-bg: #fff;
  --color-text: #111;
}

[data-theme='dark'] {
  --color-bg: #111;
  --color-text: #fff;
}

body {
  background: var(--color-bg);
  color: var(--color-text);
}
```

Use tokens for colors, spacing, typography, radius, and shadows. Keep semantic
tokens separate from raw palette values.

## CSS Animation and Performance

The most performant properties to animate are usually `transform` and `opacity`
because they can often be handled by compositing.

```css
.toast {
  transform: translateY(8px);
  opacity: 0;
  transition: transform 180ms ease, opacity 180ms ease;
}

.toast[data-open='true'] {
  transform: translateY(0);
  opacity: 1;
}
```

Avoid animating layout-heavy properties such as `width`, `height`, `top`, or
`left` when possible. Respect `prefers-reduced-motion`.

## Selectors and Specificity

Specificity roughly follows:

1. Inline styles.
2. IDs.
3. Classes, attributes, pseudo-classes.
4. Elements and pseudo-elements.

Engineering practice:

- Keep specificity low.
- Avoid deep descendant selectors.
- Use naming conventions or component scoping.
- Use cascade layers when appropriate.

## Modern CSS Features

Important modern features:

- Container queries.
- Cascade layers.
- `:has()`.
- `clamp()`, `min()`, `max()`.
- Logical properties.
- Subgrid where supported.

Use progressive enhancement. Check browser support before using a feature in
critical paths.

## CSS Engineering

Common approaches:

- BEM.
- CSS Modules.
- CSS-in-JS.
- Utility-first CSS.
- Design tokens.

There is no universal best choice. Evaluate runtime cost, SSR compatibility,
theming, build tooling, team convention, and design-system maturity.

## Interview Self-check

1. Explain the box model and `box-sizing`.
2. What is BFC, and when do you create one?
3. Flexbox vs Grid: how do you choose?
4. Why does `min-width: 0` matter in flex layouts?
5. How do you design responsive breakpoints?
6. How do CSS variables support theming?
7. Which CSS properties are best for animation performance?
8. How does specificity work?
9. What are container queries useful for?
10. How do you prevent layout shift?
11. How would you choose a CSS engineering strategy?
12. How do you support dark mode and accessibility together?
13. How do stacking contexts cause z-index bugs?
14. How do logical properties help internationalization and writing modes?
15. When can `will-change` make performance worse?
16. How do you debug a layout that only breaks with long translated text?
17. How do cascade layers change specificity governance?
18. What are the trade-offs of CSS-in-JS in SSR applications?
19. How do you design design tokens that support theming without leaking raw palette values?
20. How would you audit a component for responsive, accessible, and motion-safe behavior?

## Production Scenarios

### Scenario 1: Text Overflows in Internationalized UI

Test with long translations, larger font settings, and right-to-left layouts.
Use flexible containers, `min-width: 0`, logical properties, and content-aware
breakpoints instead of hard-coded dimensions.

### Scenario 2: Modal Appears Behind Another Layer

Inspect stacking contexts created by `position`, `z-index`, `transform`,
`opacity`, `filter`, and containment. Centralize overlay portals and z-index
tokens so component-local fixes do not create new global conflicts.

### Scenario 3: Animation Causes Jank

Trace rendering cost. Prefer `transform` and `opacity`, avoid layout-triggering
properties, remove unnecessary layer promotion, and honor `prefers-reduced-motion`
for accessibility.

### Scenario 4: Dark Mode Looks Inconsistent

Separate semantic tokens from raw colors, cover component states such as hover,
focus, disabled, and error, and test contrast in both themes before release.

## Summary

CSS interview answers should show layout judgment. Explain the formatting model,
choose Flexbox or Grid based on constraints, keep responsive behavior
content-driven, and connect animation or theming choices to performance and
maintainability.
