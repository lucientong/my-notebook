Language: English | [中文](../前端知识库/05-浏览器原理与Web安全.md)

# Browser Internals and Web Security

---

## Table of Contents

### Browser Internals

1. [Browser Architecture](#browser-architecture)
2. [Rendering Pipeline](#rendering-pipeline)
3. [Browser Cache](#browser-cache)

### Network and Cross-origin

4. [Same-origin Policy and CORS](#same-origin-policy-and-cors)
5. [HTTPS and TLS](#https-and-tls)

### Web Security and Self-check

6. [XSS](#xss)
7. [CSRF](#csrf)
8. [Clickjacking and Injection](#clickjacking-and-injection)
9. [Interview Self-check](#interview-self-check)

---

## Browser Architecture

Modern browsers use a multi-process architecture:

- Browser process: tabs, navigation, permissions, network coordination.
- Renderer process: HTML/CSS/JS execution and page rendering.
- GPU process: compositing and graphics.
- Network service: request handling and cache.
- Utility processes: isolated tasks such as audio or decoding.

The advantage is isolation and stability. The cost is inter-process
communication and higher memory usage.

Interview answer:

"A modern browser isolates pages and sensitive work into processes. This improves
security and crash isolation, but it means rendering, networking, and GPU work
must coordinate through process boundaries."

## Rendering Pipeline

High-level pipeline:

1. Parse HTML into the DOM.
2. Parse CSS into the CSSOM.
3. Combine DOM and CSSOM into a render tree.
4. Calculate layout.
5. Paint pixels.
6. Composite layers.

JavaScript can block parsing when it is a normal synchronous script. `defer`
waits until HTML parsing is complete and preserves order. `async` downloads in
parallel and executes as soon as it is ready.

### Reflow, Repaint, and Composite

- **Reflow/Layout**: geometry changes, such as width, font size, or DOM insertion.
- **Repaint**: visual changes without layout, such as color or background.
- **Composite**: combine layers, often cheaper than layout and paint.

Production practice:

- Batch DOM reads and writes.
- Avoid forced synchronous layout.
- Use transforms and opacity for animations.
- Use `will-change` carefully; too many layers increase memory.

## Browser Cache

### Strong Cache

The browser can reuse a resource without asking the server.

```http
Cache-Control: public, max-age=31536000, immutable
```

Use this for hashed static assets.

### Negotiated Cache

The browser asks the server whether the resource changed.

```http
ETag: "abc123"
If-None-Match: "abc123"
```

If unchanged, the server returns `304 Not Modified`.

### Frontend Deployment Strategy

- HTML: no-cache or short cache.
- Hashed JS/CSS: long cache and immutable.
- API: cache only when business semantics allow it.
- Service Worker: design an explicit update strategy to avoid stale releases.

## Same-origin Policy and CORS

The same-origin policy restricts one origin from reading resources from another
origin. Origin is scheme, host, and port.

CORS is a browser-enforced protocol that lets servers opt into cross-origin
access.

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

Requests with custom headers, non-simple methods, or certain content types
trigger a preflight `OPTIONS` request.

Production practice:

- Do not use wildcard origin with credentials.
- Keep allowed origins explicit.
- Cache preflight responses when safe.
- Handle CORS at the edge or gateway consistently.

## HTTPS and TLS

TLS provides encryption, integrity, and authentication.

Simplified handshake:

1. Client sends supported versions and cipher suites.
2. Server returns certificate and selected parameters.
3. Client verifies certificate chain and hostname.
4. Both sides derive session keys.
5. Application data is encrypted with symmetric keys.

HSTS tells browsers to use HTTPS automatically for future requests.

## XSS

Cross-site scripting happens when untrusted input is executed as code in the
user's browser.

Types:

- Stored XSS.
- Reflected XSS.
- DOM-based XSS.

Defenses:

- Escape output by context.
- Avoid dangerous sinks such as `innerHTML`.
- Sanitize rich text.
- Use CSP.
- Store sensitive tokens in HttpOnly cookies.

```javascript
// Prefer textContent for plain text.
element.textContent = userInput;
```

## CSRF

CSRF tricks a logged-in browser into sending an unwanted request with existing
credentials.

Defenses:

- SameSite cookies.
- CSRF tokens.
- Verify Origin or Referer for state-changing requests.
- Use idempotent semantics correctly.

CSRF is less effective against token-in-header APIs, but XSS can still steal or
abuse tokens if they are accessible to JavaScript.

## Clickjacking and Injection

Clickjacking embeds a page in a frame and tricks users into clicking hidden UI.

```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'self'
```

SQL injection is primarily a backend issue, but frontend engineers should never
assume client validation is a security boundary. Always validate and parameterize
on the server.

## Interview Self-check

1. What happens after typing a URL and pressing Enter?
2. How do layout, paint, and compositing differ?
3. How do strong cache and negotiated cache differ?
4. What triggers a CORS preflight?
5. Why is wildcard CORS unsafe with credentials?
6. How do you defend against XSS?
7. How do you defend against CSRF?
8. What is CSP, and how do you roll it out safely?
9. How do `async` and `defer` differ?
10. What is cross-origin isolation, and when is it needed?
11. How do you prevent Service Worker stale-cache problems?
12. How would you debug a white screen in production?
13. How do preload, prefetch, and preconnect differ, and when can each hurt performance?
14. What browser behaviors can force synchronous layout?
15. How do you choose between strong cache, negotiated cache, and no-store?
16. Why is `Access-Control-Allow-Origin: *` invalid with credentials?
17. How do COOP and COEP affect SharedArrayBuffer and third-party resources?
18. What data should a frontend security incident report include?
19. How do XSS and CSRF defenses change when auth moves from localStorage tokens to HttpOnly cookies?
20. How would you phase in a stricter CSP without breaking analytics or legacy scripts?

## Production Scenarios

### White Screen Debugging

Check release version, HTML and asset cache consistency, JS runtime errors,
chunk loading failures, CSP violations, network failures, and API dependency
errors. Use monitoring data to segment by browser, region, release, and route.

### CSP Rollout

Start with `Content-Security-Policy-Report-Only`, collect violations, remove
inline scripts, add nonces or hashes where necessary, then enforce gradually.

### Service Worker Update

Version cache names, clean old caches, notify users when a new version is ready,
and avoid serving old HTML with new assets or new HTML with old assets.

### Cross-origin Isolation Rollout

Audit third-party scripts, iframes, workers, and CDN headers before enabling
COOP/COEP. Roll out by route or cohort, monitor resource load failures, and keep
a rollback path because one missing `Cross-Origin-Resource-Policy` header can
break critical embeds.

### Suspicious XSS Report

Capture the affected route, release, user input source, sink, CSP violation, and
browser details. Patch the sink with contextual escaping or sanitization, rotate
exposed credentials if needed, and add regression tests around the vulnerable
rendering path.

## Summary

Browser knowledge should be explained as a pipeline: navigation, networking,
parsing, JavaScript execution, rendering, caching, and security boundaries. In
interviews, connect mechanisms to user-visible failures and production defenses.
