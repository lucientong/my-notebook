Language: English | [中文](../前端知识库/06-前端框架与工程化综合.md)

# Frontend Frameworks and Engineering

---

## Table of Contents

### Frameworks and Architecture

1. [Vue In Depth](#1-vue-in-depth)
2. [Next.js and SSR](#2-nextjs-and-ssr)
3. [Micro-frontends](#3-micro-frontends)
4. [Cross-platform Development](#4-cross-platform-development)

### Engineering and Performance

5. [Build Tools](#5-build-tools)
6. [Package Managers](#6-package-managers)
7. [Frontend Monitoring](#7-frontend-monitoring)
8. [Advanced Frontend Engineering](#8-advanced-frontend-engineering)
9. [Interview Self-check](#9-interview-self-check)
10. [Production Scenarios](#10-production-scenarios)

---

## 1. Vue In Depth

Vue 3 uses Proxy-based reactivity and the Composition API. Compared with Vue 2,
it improves type support, tree shaking, composition, and large-application
maintainability.

### Reactivity

Vue tracks dependencies when reactive values are read and triggers effects when
they change.

```javascript
import { reactive, computed } from 'vue';

const state = reactive({ count: 1 });
const doubled = computed(() => state.count * 2);
```

Key concepts:

- `reactive`: deep reactive object.
- `ref`: reactive wrapper for primitive values or object references.
- `computed`: cached derived value.
- `watch`: side effect in response to changes.

### Composition API vs Options API

Composition API groups logic by feature instead of by option type. It is better
for complex components, reusable composables, and TypeScript.

Options API is still readable for simple components and teams that prefer strong
conventions.

## 2. Next.js and SSR

Rendering modes:

- **CSR**: browser renders after JavaScript loads.
- **SSR**: server renders HTML per request.
- **SSG**: static HTML generated at build time.
- **ISR**: static pages regenerated over time.
- **RSC**: React Server Components reduce client JavaScript for server-only UI.

Trade-offs:

- SSR improves first paint and SEO but increases server complexity.
- SSG is fast and cache-friendly but less dynamic.
- CSR is simple for authenticated dashboards but can hurt initial load.
- RSC reduces client bundle size but changes data and component boundaries.

Production practice:

- Keep server-only code out of client bundles.
- Design cache keys and invalidation explicitly.
- Monitor server latency and hydration errors.
- Use streaming when slow data sources would block the full page.

## 3. Micro-frontends

Micro-frontends split a frontend system by team or domain, enabling independent
development and deployment.

Implementation options:

- Module Federation.
- Runtime container frameworks.
- iframe isolation.
- Build-time integration.
- Web Components.

Core problems:

- Style isolation.
- JavaScript sandboxing.
- Shared dependency governance.
- Routing ownership.
- Cross-app communication.
- Independent release and rollback.

Interview answer:

"I would choose micro-frontends only when organizational autonomy justifies the
runtime and governance cost. The technical design must include shared dependency
policy, observability, version compatibility, and fallback behavior."

## 4. Cross-platform Development

Common options:

- React Native: native UI with JavaScript/React.
- Flutter: custom rendering engine and Dart.
- Hybrid app: WebView plus native bridge.
- Mini programs: platform-specific constrained runtime.
- Uni-app/Taro-like solutions: multi-target compilation.

Selection criteria:

- Required native capability.
- Performance and animation needs.
- Team skill set.
- Platform constraints.
- Release process.
- Long-term maintenance cost.

## 5. Build Tools

### Webpack

Webpack builds a dependency graph and transforms modules through loaders and
plugins. It is mature and highly configurable.

### Vite

Vite uses native ES modules during development and Rollup for production build.
It is fast because it avoids bundling the entire app during dev startup.

Trade-off:

- Webpack is powerful for complex legacy setups.
- Vite is simpler and faster for modern ESM-first applications.
- Migration should be measured by build time, dev startup, HMR speed, and plugin
  compatibility.

## 6. Package Managers

### npm, Yarn, and pnpm

`pnpm` stores packages in a content-addressable store and links them into
projects, saving disk space and making dependency resolution stricter.

Monorepo concerns:

- Workspace dependency boundaries.
- Shared tooling.
- Incremental builds.
- Versioning strategy.
- Ownership and CI scalability.

## 7. Frontend Monitoring

Monitoring should be actionable, not just comprehensive.

Collect:

- JavaScript errors.
- Promise rejections.
- Resource loading failures.
- Web Vitals and custom performance metrics.
- API latency and error rate.
- User behavior breadcrumbs.
- Release version, route, browser, region, and user segment.

Alerting should be based on user impact, such as error-rate increase, Core Web
Vitals regression, white-screen rate, or conversion drop.

## 8. Advanced Frontend Engineering

Important practices:

- Linting and formatting.
- Type checking.
- Unit, integration, and E2E testing.
- CI gates.
- Bundle analysis.
- Feature flags.
- Gray release and rollback.
- Static asset versioning.
- Docker or artifact-based deployment.

### Rollback-safe Frontend Release

Keep HTML and static assets compatible across versions. Use hashed assets, avoid
deleting old assets immediately, and make feature flags backward compatible
during rollout.

## 9. Interview Self-check

1. Why is Vue 3 faster or more scalable than Vue 2?
2. How do `ref` and `reactive` differ?
3. SSR vs SSG vs CSR: how do you choose?
4. What problem does React Server Components solve?
5. How do micro-frontends isolate styles and JavaScript?
6. Why is Vite faster than Webpack in development?
7. What are pnpm's advantages?
8. What should frontend monitoring collect?
9. How do you design actionable alerts?
10. How do you measure whether an engineering migration succeeded?
11. How do you avoid asset-version mismatch during gray release?
12. How do you govern shared dependency drift in micro-frontends?
13. How do Vue's `ref`, `reactive`, `computed`, `watch`, and `watchEffect` differ in ownership and timing?
14. How do you design permission-based dynamic routes without leaking unauthorized UI?
15. What are the operational costs of React Server Components?
16. How do you decide whether Module Federation is worth adopting?
17. How do you migrate from Webpack to Vite without blocking feature delivery?
18. How do you enforce package boundaries in a monorepo?
19. What is the difference between collecting telemetry and creating actionable observability?
20. How do feature flags, static assets, and backend compatibility interact during rollback?

## 10. Production Scenarios

### Scenario 1: Micro-frontend Dependency Drift

Define shared dependency ownership, allowed version ranges, compatibility tests,
and release dashboards by sub-application. For critical runtime dependencies
like React or Vue, prefer explicit governance over ad hoc sharing.

### Scenario 2: Build Tool Migration

Measure baseline startup, HMR, build time, bundle output, plugin compatibility,
and sourcemap quality. Migrate one package or route first, run both pipelines in
CI temporarily, and keep rollback simple until production metrics are stable.

### Scenario 3: Monitoring Produces Noise

Group alerts by user impact and release. Keep raw telemetry for investigation,
but alert on white-screen rate, conversion drop, Web Vitals regression, or error
budget burn instead of every individual JavaScript exception.

### Scenario 4: Rollback Fails Because Assets Were Deleted

Keep multiple release artifacts available, cache HTML conservatively, make flags
backward compatible, and verify that old HTML can still load old chunks after a
new deployment has started.

## Summary

Modern frontend engineering is not only about frameworks. Strong candidates can
explain rendering mode choices, build pipelines, package management,
observability, release safety, and organizational trade-offs.
