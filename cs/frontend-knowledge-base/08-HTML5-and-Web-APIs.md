Language: English | [中文](../前端知识库/08-HTML5与Web API.md)

# HTML5 and Web APIs

---

## Table of Contents

1. [Semantic HTML](#semantic-html)
2. [Web Storage](#web-storage)
3. [Web Worker](#web-worker)
4. [Service Worker and PWA](#service-worker-and-pwa)
5. [Canvas and WebGL](#canvas-and-webgl)
6. [WebRTC](#webrtc)
7. [Fetch API and AbortController](#fetch-api-and-abortcontroller)
8. [Intersection Observer](#intersection-observer)
9. [Web Animations API](#web-animations-api)
10. [Other Important APIs](#other-important-apis)
11. [Interview Self-check](#interview-self-check)
12. [Production Scenarios](#production-scenarios)

---

## Semantic HTML

Semantic HTML describes meaning, not just appearance. It improves accessibility,
SEO, maintainability, and browser behavior.

```html
<main>
  <article>
    <h1>Frontend Performance Guide</h1>
    <p>Measure first, optimize second.</p>
  </article>
</main>
```

Production practice:

- Use heading levels in logical order.
- Use buttons for actions and anchors for navigation.
- Associate labels with form controls.
- Add accessible names to interactive elements.
- Do not replace native controls without a strong reason.

## Web Storage

### localStorage

Synchronous key-value storage that persists across sessions.

### sessionStorage

Synchronous key-value storage scoped to a tab session.

### IndexedDB

Asynchronous structured storage for larger data and offline scenarios.

Trade-offs:

- `localStorage` is simple but blocks the main thread and is not suitable for
  large data.
- IndexedDB is more complex but better for offline-first applications.
- Sensitive tokens should not be stored in JavaScript-accessible storage.

## Web Worker

Web Workers run JavaScript off the main thread.

```javascript
// main.js
const worker = new Worker('/worker.js');

worker.postMessage({ numbers: [1, 2, 3] });
worker.onmessage = (event) => {
  console.log(event.data);
};
```

Use workers for CPU-heavy parsing, image processing, compression, search, or
large data transformation. Communication uses message passing and structured
clone, so large payloads may need Transferable objects.

## Service Worker and PWA

A Service Worker is a programmable network proxy between the page and network.
It enables offline cache, push notifications, and background sync.

Lifecycle:

1. Register.
2. Install.
3. Activate.
4. Intercept fetch events.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request);
    }),
  );
});
```

Production risks:

- Serving stale HTML or assets.
- Cache version mismatch.
- Hard-to-debug update behavior.
- Offline logic hiding real network failures.

## Canvas and WebGL

Canvas is immediate-mode 2D drawing. WebGL exposes GPU-accelerated graphics.

Canvas fits charts, image editing, custom rendering, and games. DOM fits
accessible, interactive, document-like UI.

```javascript
const canvas = document.querySelector('canvas');
const ctx = canvas.getContext('2d');

ctx.fillStyle = '#1677ff';
ctx.fillRect(10, 10, 120, 80);
```

Production practice:

- Scale for device pixel ratio.
- Avoid unnecessary redraws.
- Use OffscreenCanvas where supported.
- Provide accessible alternatives for important content.

## WebRTC

WebRTC enables peer-to-peer audio, video, and data channels.

Core pieces:

- `getUserMedia`: capture camera and microphone.
- `RTCPeerConnection`: peer connection.
- ICE/STUN/TURN: connectivity traversal.
- Signaling: application-defined exchange of offers, answers, and candidates.

Trade-off:

WebRTC can reduce media latency, but operational complexity is high because NAT
traversal, TURN cost, device permissions, and browser compatibility must be
managed.

## Fetch API and AbortController

`fetch` returns a Promise and does not reject on HTTP error status by default.

```javascript
async function request(url, signal) {
  const response = await fetch(url, { signal });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  return response.json();
}

const controller = new AbortController();
request('/api/data', controller.signal);
controller.abort();
```

Production practice:

- Handle HTTP errors explicitly.
- Add cancellation for route changes or repeated search.
- Add timeout wrappers.
- Separate transport errors from business errors.

## Intersection Observer

Intersection Observer asynchronously observes element visibility relative to a
viewport or container.

```javascript
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      loadImage(entry.target);
      observer.unobserve(entry.target);
    }
  }
});
```

Use cases:

- Lazy loading.
- Infinite scroll.
- Visibility analytics.
- Triggering animations.

## Web Animations API

The Web Animations API gives JavaScript control over animations while using the
browser animation engine.

```javascript
element.animate(
  [
    { opacity: 0, transform: 'translateY(8px)' },
    { opacity: 1, transform: 'translateY(0)' },
  ],
  { duration: 180, easing: 'ease-out' },
);
```

Prefer CSS transitions for simple state changes. Use WAAPI when animation needs
runtime control, cancellation, sequencing, or integration with application
logic.

## Other Important APIs

- History API: SPA navigation.
- URL and URLSearchParams: robust URL parsing.
- Clipboard API: controlled clipboard access.
- Notification API: user-visible notifications.
- Performance API: timing and metrics.
- ResizeObserver: element-size observation.
- MutationObserver: DOM mutation observation.

## Interview Self-check

1. Why does semantic HTML matter?
2. localStorage vs sessionStorage vs IndexedDB: how do you choose?
3. When should you use Web Workers?
4. What problem does Service Worker solve?
5. How do you avoid stale Service Worker caches?
6. Canvas vs DOM: how do you choose?
7. What are the key pieces of WebRTC?
8. Why does `fetch` not reject on HTTP 500?
9. How do you cancel a request?
10. What is Intersection Observer useful for?
11. When would you use Web Animations API instead of CSS?
12. How do browser permissions affect API design?
13. How do you choose between DOM, Canvas, SVG, and WebGL for data visualization?
14. How do you design IndexedDB schemas and migrations for offline applications?
15. What are the failure modes of Service Worker cache-first strategies?
16. How do Transferable objects improve Worker communication?
17. How would you debug WebRTC connection failure across NAT or corporate networks?
18. How do ResizeObserver and MutationObserver differ from polling?
19. How do you make Canvas-based content accessible?
20. What API capability checks should run before enabling a browser feature?

## Production Scenarios

### Scenario 1: Offline Data Becomes Corrupted

Version IndexedDB schemas, migrate data transactionally, validate records at
read/write boundaries, and keep server reconciliation rules explicit. Offline
success should be measured by conflict rate, sync latency, and recoverability.

### Scenario 2: Web Worker Is Still Slow

Inspect message size and serialization cost. Use Transferable objects for large
buffers, keep worker protocols simple, batch small messages, and avoid copying
large objects back to the main thread for every frame.

### Scenario 3: WebRTC Works in Office Wi-Fi but Fails for Some Users

Check ICE candidate gathering, STUN/TURN availability, firewall restrictions,
permission prompts, codec support, and signaling reliability. Monitor connection
state, selected candidate pair, bitrate, packet loss, and TURN usage.

### Scenario 4: Canvas Chart Is Not Accessible

Provide semantic summaries, keyboard-accessible controls, text alternatives for
critical values, and a table or DOM representation when the chart communicates
business-critical information.

## Summary

HTML5 and Web APIs extend the browser from a document viewer into an application
runtime. Strong interview answers explain not only what each API does, but also
its lifecycle, permission model, performance impact, and production failure
modes.
