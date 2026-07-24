Language: English | [中文](../前端知识库/10-移动端与跨端开发.md)

# Mobile and Cross-platform Development

---

## Table of Contents

1. [Mobile Adaptation](#mobile-adaptation)
2. [Mobile Gestures and Interaction](#mobile-gestures-and-interaction)
3. [Hybrid App](#hybrid-app)
4. [React Native In Depth](#react-native-in-depth)
5. [Flutter In Depth](#flutter-in-depth)
6. [Mini Programs](#mini-programs)
7. [Electron and Tauri](#electron-and-tauri)
8. [Cross-platform Solution Comparison](#cross-platform-solution-comparison)
9. [Mobile Performance Optimization](#mobile-performance-optimization)
10. [Interview Self-check](#interview-self-check)

---

## Mobile Adaptation

Mobile adaptation handles different screen sizes, pixel densities, safe areas,
input methods, and browser behavior.

Common techniques:

- Responsive layout with CSS media queries.
- Fluid units such as `%`, `vw`, `rem`, and `clamp()`.
- Device pixel ratio aware images.
- Safe-area handling for notches.
- Touch-friendly spacing and target sizes.

```css
.page {
  padding: max(16px, env(safe-area-inset-top)) 16px
    max(16px, env(safe-area-inset-bottom));
}
```

Production practice:

- Test real devices, not only desktop emulation.
- Handle landscape mode.
- Avoid fixed heights around virtual keyboards.
- Reserve space to prevent layout shift.

## Mobile Gestures and Interaction

Mobile interaction differs from desktop:

- Touch events and pointer events.
- Tap delay history and click synthesis.
- Scroll momentum.
- Pull-to-refresh.
- Swipe and drag.
- Virtual keyboard resize behavior.

Use Pointer Events when possible because they unify mouse, touch, and pen input.

Production risks:

- Gesture conflict with native scrolling.
- Passive listener issues.
- Accidental text selection.
- Poor accessibility for custom gestures.

## Hybrid App

Hybrid apps combine native shells with WebView content.

Core pieces:

- WebView container.
- JavaScript bridge.
- Native capability modules.
- URL routing and permissions.
- Offline package or hot update strategy.

Trade-offs:

- Faster feature delivery and shared web code.
- Lower access to native performance and platform-specific UX.
- Bridge design and version compatibility become critical.

Bridge calls should be versioned, permission-aware, and observable.

## React Native In Depth

React Native uses React to describe native UI. It is not a WebView by default.

Key concepts:

- JavaScript runtime.
- Native UI components.
- Bridge or newer JSI/TurboModules/Fabric architecture.
- Metro bundler.

Strengths:

- React mental model.
- Native UI feel.
- Large ecosystem.
- Shared business logic across platforms.

Challenges:

- Native module maintenance.
- Platform differences.
- App store release cycles.
- Performance tuning for large lists and animations.

## Flutter In Depth

Flutter uses Dart and renders UI through its own engine.

Strengths:

- Consistent UI across platforms.
- High-performance rendering.
- Rich widget system.
- Strong tooling.

Challenges:

- Different language and ecosystem.
- Larger runtime footprint.
- Native integration still needed for platform APIs.
- UI may need extra work to feel fully platform-native.

## Mini Programs

Mini programs run inside host platforms such as WeChat. They provide a controlled
runtime, platform APIs, and distribution channel.

Characteristics:

- Constrained runtime.
- Platform-specific components and APIs.
- Package size limits.
- Review and release process.
- Strong dependency on host platform rules.

Engineering practice:

- Separate business logic from platform adapters.
- Watch package size.
- Handle platform API failures gracefully.
- Test across host app versions.

## Electron and Tauri

Electron packages Chromium and Node.js for desktop apps. Tauri uses the system
WebView and a Rust backend, often producing smaller apps.

Comparison:

- Electron: mature ecosystem, larger package size, consistent Chromium behavior.
- Tauri: smaller footprint, stronger native shell model, more platform WebView
  variation.

Use desktop wrappers when the product needs filesystem access, tray, native
menus, offline desktop behavior, or deep OS integration.

## Cross-platform Solution Comparison

Selection criteria:

- UI performance requirement.
- Native capability requirement.
- Team language and framework expertise.
- Release speed.
- Platform consistency.
- Long-term maintenance.
- Ecosystem maturity.

Rule of thumb:

- H5/PWA: content-heavy and fast iteration.
- Hybrid: existing native app plus web business pages.
- React Native: React team needing native UI.
- Flutter: highly customized UI and consistent cross-platform rendering.
- Mini program: platform ecosystem distribution.
- Electron/Tauri: desktop product.

## Mobile Performance Optimization

Mobile constraints are stricter: CPU, memory, battery, network, and WebView
behavior.

Optimization areas:

- Reduce JavaScript bundle size.
- Avoid long tasks.
- Lazy-load non-critical modules.
- Compress and resize images.
- Use GPU-friendly animations.
- Reduce bridge calls in hybrid or RN apps.
- Cache carefully for unstable networks.
- Monitor cold start and first screen.

For React Native:

- Use `FlatList` for large lists.
- Avoid excessive re-rendering.
- Move animations to native/UI thread where possible.
- Optimize image size and caching.

## Interview Self-check

1. How do you handle mobile viewport adaptation?
2. What are safe areas, and why do they matter?
3. How do touch and pointer events differ?
4. What is a Hybrid app bridge?
5. React Native vs Hybrid WebView: how do they differ?
6. Flutter vs React Native: how would you choose?
7. What are mini program constraints?
8. Electron vs Tauri: how do you choose?
9. How do you optimize mobile first-screen performance?
10. How do you reduce bridge overhead?
11. How do you handle virtual keyboard layout issues?
12. How do you design a cross-platform architecture that remains maintainable?
13. How do viewport units, safe areas, and virtual keyboards differ across mobile browsers?
14. How do you prevent custom gestures from breaking native scrolling and accessibility?
15. How do you version a Hybrid bridge so old native shells and new web pages remain compatible?
16. How do React Native's legacy bridge and newer JSI/Fabric architecture change performance trade-offs?
17. How do you decide whether a performance issue belongs to JavaScript, native UI, bridge, or network?
18. How do mini program package limits affect architecture?
19. How do Electron and Tauri differ in security hardening responsibilities?
20. What rollout and rollback constraints are unique to app-store distributed clients?

## Production Scenarios

### Scenario 1: Keyboard Covers Input

Test real devices, avoid fixed viewport assumptions, use scroll-into-view
behavior, account for safe areas, and handle platform-specific keyboard resize
events.

### Scenario 2: Hybrid Page Is Slow

Measure WebView startup, HTML load, JS execution, bridge calls, image weight,
and network waterfall. Cache the shell, prefetch critical resources, and reduce
bridge round trips.

### Scenario 3: Choosing a Cross-platform Stack

Start from product requirements and team capability. Compare native feature
needs, animation complexity, release process, hiring, ecosystem risk, and
maintenance cost before choosing technology.

### Scenario 4: Bridge Method Breaks Older Native Clients

Version bridge APIs, feature-detect capabilities, keep backward-compatible
fallbacks, and log native shell version with every bridge error. Web releases
must respect the slowest supported app-store client still in the wild.

### Scenario 5: React Native Screen Drops Frames

Profile JavaScript thread, UI thread, image decoding, list virtualization, and
bridge traffic separately. Reduce re-renders, tune `FlatList`, move animations to
the native/UI thread, and avoid sending large payloads across the bridge.

### Scenario 6: Mobile Web Page Fails on Specific Browsers

Reproduce on real devices, check viewport behavior, input method resize, passive
listeners, CSS feature support, memory pressure, and WebView vendor version.
Apply progressive enhancement instead of assuming desktop browser parity.

## Summary

Mobile and cross-platform development is mostly about constraints and trade-offs.
Strong interview answers compare solutions honestly and connect adaptation,
interaction, performance, native capability, and release governance.
