Language: English | [中文](../前端知识库/02-React框架深入.md)

# React Framework In Depth

---

## Table of Contents

### Part 1: Component Architecture

1. [React Component Tree and Architecture](#1-react-component-tree-and-architecture)
2. [Component Communication Patterns](#2-component-communication-patterns)
3. [Data Flow and State Management](#3-data-flow-and-state-management)

### Part 2: Rendering Mechanism

4. [Rendering and Update Process](#4-rendering-and-update-process)
5. [Virtual DOM and Diffing](#5-virtual-dom-and-diffing)
6. [Fiber Architecture and Concurrent Rendering](#6-fiber-architecture-and-concurrent-rendering)

### Part 3: Lifecycle and Hooks

7. [Lifecycle Model](#7-lifecycle-model)
8. [Hooks Internals](#8-hooks-internals)
9. [Custom Hooks Best Practices](#9-custom-hooks-best-practices)

### Part 4: Design, Performance, and Practice

10. [React vs Vue vs Angular](#10-react-vs-vue-vs-angular)
11. [React Design Philosophy](#11-react-design-philosophy)
12. [Advanced Performance Optimization](#12-advanced-performance-optimization)
13. [Advanced Patterns](#13-advanced-patterns)
14. [Interview Self-check](#14-interview-self-check)
15. [Enterprise Scenarios](#15-enterprise-scenarios)
16. [React and Micro-frontends](#16-react-and-micro-frontends)

---

## 1. React Component Tree and Architecture

React applications are trees. Components describe UI, props flow downward, and
state changes trigger a new render calculation.

```jsx
function App() {
  return (
    <Layout>
      <Header />
      <Main />
      <Footer />
    </Layout>
  );
}
```

Internally, React represents the UI with Fiber nodes. A Fiber stores the element
type, props, state, links to child/sibling/parent, and effect information.

Key trees:

- **Current tree**: what is committed to the screen.
- **Work-in-progress tree**: the next version being prepared.
- **Host tree**: the actual DOM, native view hierarchy, or other renderer target.

Interview phrasing:

"React is declarative at the component level but operational internally. A state
update schedules work, React builds a work-in-progress Fiber tree, compares it
with the current tree, and commits the minimal host changes."

## 2. Component Communication Patterns

Common patterns:

- Parent to child: props.
- Child to parent: callback props.
- Sibling communication: lift state up.
- Cross-tree communication: Context.
- Global state: Redux, Zustand, Jotai, Recoil, or framework-level stores.
- Imperative escape hatch: refs.
- URL state: route params and query strings.
- Server state: React Query, SWR, or framework data APIs.

### Trade-offs

Context is convenient, but updating a large context value can re-render many
consumers. Server state should usually not be stored in a client-only global
store because it has cache invalidation, refetching, and stale-data semantics.

## 3. Data Flow and State Management

React encourages one-way data flow: state changes create a new render output.
Choose state ownership by lifetime and sharing scope:

- Local UI state: `useState` or `useReducer`.
- Derived state: compute during render instead of duplicating it.
- Shared UI state: context or a lightweight store.
- Server cache: React Query, SWR, or framework loader APIs.
- Persistent state: URL, localStorage, IndexedDB, or backend storage.

Production practice:

- Avoid duplicating the same source of truth.
- Keep server cache invalidation explicit.
- Use selectors to reduce re-renders in global stores.
- Keep business-critical state recoverable after refresh.

## 4. Rendering and Update Process

React rendering has two major phases:

1. **Render phase**: calculate the next UI. It can be paused, restarted, or
   discarded in concurrent rendering.
2. **Commit phase**: apply changes to the host environment. It must be completed
   consistently.

Because render can be repeated, render logic must be pure. Side effects belong
in Effects or event handlers.

```jsx
function SearchBox({ query, onChange }) {
  // Pure: output depends on props and state.
  return <input value={query} onChange={(e) => onChange(e.target.value)} />;
}
```

## 5. Virtual DOM and Diffing

Virtual DOM is a JavaScript representation of UI. It lets React compare two
trees and decide the host operations needed.

React's diff uses heuristics:

- Different element types produce different subtrees.
- Stable `key` values help match children across renders.
- The algorithm is optimized for common UI changes rather than solving the
  theoretical minimum edit distance.

```jsx
items.map((item) => <TodoItem key={item.id} item={item} />);
```

Index keys are risky when items can be inserted, deleted, or reordered because
state may be associated with the wrong item.

## 6. Fiber Architecture and Concurrent Rendering

Fiber turns rendering work into units that React can schedule. This enables:

- Time slicing.
- Prioritized updates.
- Interruptible rendering.
- Transitions for non-urgent updates.
- Suspense integration.

React 18 introduced concurrent capabilities, but concurrent rendering does not
mean every update is parallel. It means React can prepare work in a more
interruptible and priority-aware way.

Production implications:

- Do not mutate external state during render.
- Effects may run more than expected in development Strict Mode.
- Use `startTransition` for non-urgent UI updates.
- Keep urgent interactions responsive.

## 7. Lifecycle Model

Function components express lifecycle through rendering and Effects:

- Mount: component first appears.
- Update: props or state change.
- Unmount: component is removed.

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/profile', { signal: controller.signal });

  return () => controller.abort();
}, []);
```

`useLayoutEffect` runs after DOM mutation but before paint. Use it for layout
measurement or synchronous visual correction. Prefer `useEffect` for most side
effects because it does not block painting.

## 8. Hooks Internals

Hooks rely on call order. React associates hook state with the component's Fiber
and the order in which hooks are called.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const doubled = useMemo(() => count * 2, [count]);

  return <button onClick={() => setCount(count + 1)}>{doubled}</button>;
}
```

This is why hooks must not be called conditionally. Conditional calls change the
order and make React read the wrong hook state.

### Stale Closure

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((current) => current + 1);
  }, 1000);

  return () => clearInterval(id);
}, []);
```

Use functional updates, refs, or correct dependencies to avoid stale closures.

## 9. Custom Hooks Best Practices

Custom hooks extract reusable stateful logic.

```jsx
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const onOnline = () => setOnline(true);
    const onOffline = () => setOnline(false);

    window.addEventListener('online', onOnline);
    window.addEventListener('offline', onOffline);

    return () => {
      window.removeEventListener('online', onOnline);
      window.removeEventListener('offline', onOffline);
    };
  }, []);

  return online;
}
```

Good custom hooks have clear ownership, cleanup, stable return shapes, and
minimal hidden global behavior.

## 10. React vs Vue vs Angular

- **React**: library-first, flexible architecture, strong ecosystem, explicit
  state choices, JSX-centric.
- **Vue**: approachable, reactive by default, integrated single-file component
  experience.
- **Angular**: full framework, strong conventions, dependency injection,
  enterprise structure.

Trade-off answer:

"React gives teams more architectural freedom, which is powerful but requires
strong conventions. Angular gives more built-in structure, while Vue optimizes
for approachability and integrated ergonomics."

## 11. React Design Philosophy

React favors:

- Declarative UI.
- Component composition.
- One-way data flow.
- Explicit state ownership.
- Reconciliation instead of manual DOM mutation.

The production benefit is predictability: UI can be reasoned about as a function
of state. The cost is that performance requires understanding render frequency,
identity stability, and data ownership.

## 12. Advanced Performance Optimization

Optimize only after measuring.

Common tools:

- React DevTools Profiler.
- Browser Performance panel.
- Web Vitals.
- Bundle analyzer.

Common techniques:

- Split expensive components.
- Use `React.memo` with stable props.
- Use `useMemo` for expensive derived values.
- Use `useCallback` when function identity affects memoized children.
- Virtualize long lists.
- Defer non-urgent updates with transitions.
- Code split route-level and feature-level bundles.

Do not use `useMemo` or `useCallback` by default. They add dependency management
cost and can make code harder to read.

## 13. Advanced Patterns

- Compound components for flexible component APIs.
- Render props for behavior injection.
- Controlled and uncontrolled components.
- Error boundaries for failure containment.
- Suspense for async UI coordination.
- Portals for modals, tooltips, and overlays.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) return <Fallback />;
    return this.props.children;
  }
}
```

## 14. Interview Self-check

1. Why must hooks be called in the same order?
2. Is `setState` synchronous or asynchronous?
3. What problem does Fiber solve?
4. Explain render and commit phases.
5. What are the trade-offs of Virtual DOM?
6. When should `useLayoutEffect` be used?
7. How do `useMemo` and `useCallback` differ?
8. Why are stable keys important?
9. How would you design error boundaries to avoid a full-page white screen?
10. How would you optimize a long list?
11. Why must side effects not run during render in concurrent rendering?
12. How do you prevent stale closures?
13. When should you avoid `useMemo` and `useCallback` even if a component re-renders?
14. How do Context updates create performance problems, and how would you contain them?
15. How do Suspense, transitions, and route-level code splitting affect perceived performance?
16. How would you debug a hydration mismatch in an SSR React application?
17. How do controlled and uncontrolled components trade correctness for performance?
18. What data should be attached to React error reports to make incidents debuggable?
19. How do you govern shared React versions in a micro-frontend system?
20. How would you prove that a React performance optimization actually worked?

## 15. Enterprise Scenarios

### Scenario 1: White Screen Containment

Use route-level and widget-level error boundaries, show fallback UI, report the
error with component stack and release version, and keep critical navigation
outside risky feature boundaries.

### Scenario 2: Slow Dashboard

Profile render cost, split heavy widgets, virtualize large tables, memoize only
measured hotspots, cache server data, and lazy-load rarely used modules.

### Scenario 3: Complex Form

Separate field state from server submission state, validate at boundaries, keep
controlled components where correctness matters, and avoid re-rendering the
entire form on each keystroke.

### Scenario 4: Hydration Mismatch After SSR Release

Compare server and client render inputs: time, random values, locale, feature
flags, user agent branches, and data freshness. Add deterministic rendering,
move browser-only logic into Effects, and segment monitoring by route and release.

### Scenario 5: Context Provider Causes Global Re-renders

Profile consumers first. Split context by update frequency, memoize provider
values, use selectors or an external store for hot state, and avoid putting large
server-cache objects into a broad UI context.

## 16. React and Micro-frontends

React micro-frontends can be built with Module Federation, qiankun-like runtime
containers, iframes, or build-time composition.

Core concerns:

- Routing ownership.
- Style isolation.
- Shared dependency versions.
- Cross-app communication.
- Independent deployment and rollback.
- Monitoring across application boundaries.

Production answer:

"I would avoid micro-frontends unless team autonomy and deployment independence
justify the operational cost. If we adopt them, I would define shared dependency
governance, route ownership, observability tags, and rollback strategy before
implementation."

## Summary

Strong React interview answers connect the component model to the internal
rendering pipeline. Explain what React guarantees, where it gives flexibility,
and how you keep production systems fast, observable, and resilient.
