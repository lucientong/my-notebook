Language: English | [中文](../前端知识库/09-前端性能优化.md)

# Frontend Performance Optimization

---

## Table of Contents

1. [Performance Metrics](#performance-metrics)
2. [Loading Performance](#loading-performance)
3. [Network Optimization](#network-optimization)
4. [Cache Strategy](#cache-strategy)
5. [Rendering Performance](#rendering-performance)
6. [JavaScript Performance](#javascript-performance)
7. [Image Optimization](#image-optimization)
8. [First-screen Optimization](#first-screen-optimization)
9. [Build Optimization](#build-optimization)
10. [Monitoring and Alerting](#monitoring-and-alerting)
11. [Interview Self-check](#interview-self-check)

---

## Performance Metrics

Measure before optimizing.

Important metrics:

- **FCP**: First Contentful Paint.
- **LCP**: Largest Contentful Paint.
- **INP**: Interaction to Next Paint.
- **CLS**: Cumulative Layout Shift.
- **TTFB**: Time to First Byte.
- **TBT**: Total Blocking Time.

Production metrics should be collected from real users, not only lab tools.

## Loading Performance

Main strategies:

- Reduce critical resources.
- Split code by route and feature.
- Preload critical assets.
- Defer non-critical scripts.
- Minimize render-blocking CSS.
- Use SSR/SSG/streaming when it improves user-perceived loading.

```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
<script type="module" src="/assets/app.js"></script>
```

Trade-off:

Preloading too much can compete with truly critical resources. Use it only when
the resource is needed very early.

## Network Optimization

Techniques:

- HTTP/2 or HTTP/3.
- CDN for static assets.
- Compression with gzip or Brotli.
- DNS prefetch and preconnect for important origins.
- Request batching or deduplication.
- Avoid unnecessary redirects.

```html
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
```

Production practice:

- Measure by region and network type.
- Monitor CDN hit rate.
- Avoid large request waterfalls.
- Keep API payloads shaped for the page.

## Cache Strategy

Recommended frontend asset policy:

- HTML: no-cache or short cache.
- Hashed JS/CSS: long cache with `immutable`.
- Images/fonts: long cache if versioned.
- API data: cache by business semantics.
- Service Worker: explicit version and update policy.

```http
Cache-Control: public, max-age=31536000, immutable
```

The key risk is version mismatch: old HTML referencing deleted assets, or new
HTML controlled by an old Service Worker.

## Rendering Performance

Rendering bottlenecks often come from layout thrashing, heavy paint, large DOM,
or expensive compositing.

Rules of thumb:

- Batch DOM reads and writes.
- Animate `transform` and `opacity`.
- Avoid unnecessary layer promotion.
- Use virtualization for long lists.
- Reserve image and ad dimensions to prevent CLS.

```css
.card {
  content-visibility: auto;
  contain-intrinsic-size: 240px;
}
```

Use modern CSS carefully and verify browser support.

## JavaScript Performance

Common causes:

- Large bundles.
- Long tasks.
- Expensive hydration.
- Heavy JSON parsing.
- Repeated computation during render.
- Unbounded event handlers.

Optimization options:

- Code splitting.
- Tree shaking.
- Web Workers.
- Debounce and throttle.
- Memoization for measured hotspots.
- Use requestIdleCallback for non-critical work where appropriate.

```javascript
const onScroll = throttle(() => {
  updateVisibleRange();
}, 100);
```

## Image Optimization

Images are often the largest page resources.

Best practices:

- Use responsive images with `srcset` and `sizes`.
- Use AVIF/WebP where supported.
- Lazy-load below-the-fold images.
- Reserve width and height.
- Use placeholders for important visual content.

```html
<img
  src="/hero-800.webp"
  srcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1200.webp 1200w"
  sizes="(max-width: 768px) 100vw, 800px"
  width="800"
  height="450"
  alt="Product dashboard"
>
```

## First-screen Optimization

First-screen performance is about the user's first useful experience.

Checklist:

- Reduce TTFB.
- Inline critical CSS when justified.
- Avoid blocking scripts.
- Prioritize the LCP resource.
- Stream HTML when server data is slow.
- Show skeletons only when they improve perceived progress.
- Avoid loading non-critical widgets before the main content.

## Build Optimization

Build-time optimization:

- Dependency analysis.
- Tree shaking.
- Minification.
- Route-level code splitting.
- Dynamic imports.
- Bundle visualization.
- Removing duplicate dependencies.
- Modern browser targets.

For CI:

- Cache package manager store.
- Cache build artifacts where safe.
- Run incremental builds in monorepos.
- Track build duration as an engineering metric.

## Monitoring and Alerting

Performance monitoring should connect metrics to releases and user impact.

Collect:

- Web Vitals.
- Route and release version.
- Device, browser, network type, region.
- Resource timing.
- Long tasks.
- API latency.

Alert on regressions:

- LCP p75 worsens significantly.
- INP p75 crosses threshold.
- CLS increases after release.
- White-screen or chunk-load error rate spikes.

## Interview Self-check

1. What are Core Web Vitals?
2. How do lab and field metrics differ?
3. How do you optimize LCP?
4. How do you reduce INP?
5. What causes CLS?
6. How do HTTP/2 and HTTP/3 help?
7. How do preload, prefetch, and preconnect differ?
8. How do you design cache policy for frontend assets?
9. How do you optimize a long list?
10. How do you reduce JavaScript bundle size?
11. How do you build actionable performance alerts?
12. How do you evaluate whether an optimization worked?
13. How do you diagnose whether an LCP issue is caused by TTFB, resource priority, or render blocking?
14. How do you reduce INP without making the UI feel delayed?
15. When can skeleton screens hurt perceived performance?
16. How do you design a performance budget that teams will actually follow?
17. How do you balance SSR, streaming, hydration cost, and CDN caching?
18. How do you identify duplicate dependencies and dead code in a modern bundle?
19. What field dimensions are necessary to make performance data actionable?
20. How do you prevent Service Worker or CDN cache policy from hiding a bad release?

## Production Scenarios

### Scenario 1: LCP Regression After Release

Check LCP element, image priority, server latency, CDN cache hit rate, CSS/JS
blocking, font loading, and whether the release added heavy above-the-fold work.

### Scenario 2: Slow Interaction

Use performance traces to identify long tasks around input. Split work, remove
unnecessary renders, debounce non-urgent updates, and move CPU-heavy logic to a
worker.

### Scenario 3: Bundle Size Growth

Inspect bundle analyzer output, find duplicate dependencies, replace heavy
libraries, split routes, and enforce performance budgets in CI.

### Scenario 4: INP Regression After Adding a Rich Widget

Find the interaction target, inspect long tasks around input, separate urgent
visual feedback from non-urgent work, move CPU-heavy processing to a worker, and
reduce unnecessary framework renders. Validate with field INP, not only local
traces.

### Scenario 5: Optimization Looks Good in Lab but Bad in Field

Segment field data by device class, browser, route, region, network type, cache
state, and release. Lab tools are controlled diagnostics; field data reveals
real user constraints and should drive priority.

### Scenario 6: CDN Cache Masks a Broken Release

Track release version in HTML, assets, and telemetry. Keep HTML revalidation
strict, purge or roll forward carefully, and verify that old and new asset
versions can coexist during rollout and rollback.

## Summary

Frontend performance is a measurement-driven discipline. Strong answers start
from metrics, identify bottlenecks in loading, network, rendering, JavaScript,
and assets, then explain how to verify improvement in production.
