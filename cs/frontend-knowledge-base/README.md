Language: English | [中文](../前端知识库/)

# Frontend Knowledge Base

This directory is the English interview-preparation version of `cs/前端知识库`.
It is designed for frontend, full-stack, and frontend infrastructure interviews in
international companies, where candidates are expected to explain mechanisms,
trade-offs, production experience, and debugging strategy in English.

## How to Use This Knowledge Base

Read each document in three passes:

1. **Build the mental model**: understand the core concepts and execution model.
2. **Practice trade-off answers**: explain when you would choose one approach over another.
3. **Rehearse production stories**: connect concepts to incidents, metrics, rollout, and monitoring.

For interview preparation, do not memorize paragraphs. Turn each section into a
two-minute answer with: **definition -> mechanism -> trade-off -> production example -> risk**.

## Recommended Reading Path

### Foundation

1. [JavaScript and TypeScript Core](./01-JavaScript-and-TypeScript-Core.md)
2. [Browser Internals and Web Security](./05-Browser-Internals-and-Web-Security.md)
3. [HTML5 and Web APIs](./08-HTML5-and-Web-APIs.md)
4. [CSS In Depth and Layout](./07-CSS-In-Depth-and-Layout.md)

### Framework and Application Development

5. [React Framework In Depth](./02-React-Framework-In-Depth.md)
6. [Frontend Frameworks and Engineering](./06-Frontend-Frameworks-and-Engineering.md)
7. [Frontend Performance Optimization](./09-Frontend-Performance-Optimization.md)

### Deployment, Full-stack, and Cross-platform

8. [Node.js Backend Development](./03-Nodejs-Backend-Development.md)
9. [Nginx and Web Servers](./04-Nginx-and-Web-Servers.md)
10. [Mobile and Cross-platform Development](./10-Mobile-and-Cross-Platform-Development.md)

## Difficulty Levels

- **L1 - Core syntax and APIs**: JavaScript scope, TypeScript types, DOM APIs, CSS layout, React component model.
- **L2 - Runtime mechanisms**: event loop, rendering pipeline, Fiber, browser cache, service worker, Node.js streams.
- **L3 - Architecture and trade-offs**: state management, SSR/SSG/CSR, micro-frontends, monorepo, deployment and rollback.
- **L4 - Production excellence**: observability, performance budgets, security hardening, incident diagnosis, gradual rollout.

## Review Status

The English documents are maintained as interview-ready mirrors of the Chinese
frontend knowledge base. As of the latest frontend review, each numbered
document includes:

- Core mechanisms from fundamentals to advanced production behavior.
- Senior interview prompts focused on trade-offs, debugging, compatibility,
  security, performance, and governance.
- Production scenarios that turn concepts into incident diagnosis and rollout
  decisions.

## Document Map

| No. | English Document | Chinese Source | Main Focus |
| --- | --- | --- | --- |
| 01 | [JavaScript and TypeScript Core](./01-JavaScript-and-TypeScript-Core.md) | [中文](../前端知识库/01-JavaScript与TypeScript核心.md) | JS runtime, async, prototypes, TS type system |
| 02 | [React Framework In Depth](./02-React-Framework-In-Depth.md) | [中文](../前端知识库/02-React框架深入.md) | component model, reconciliation, Fiber, hooks, performance |
| 03 | [Node.js Backend Development](./03-Nodejs-Backend-Development.md) | [中文](../前端知识库/03-Node.js后端开发.md) | Node runtime, middleware, auth, streams, deployment |
| 04 | [Nginx and Web Servers](./04-Nginx-and-Web-Servers.md) | [中文](../前端知识库/04-Nginx与Web服务器.md) | reverse proxy, cache, TLS, load balancing, rollout |
| 05 | [Browser Internals and Web Security](./05-Browser-Internals-and-Web-Security.md) | [中文](../前端知识库/05-浏览器原理与Web安全.md) | rendering, cache, CORS, XSS, CSRF, TLS |
| 06 | [Frontend Frameworks and Engineering](./06-Frontend-Frameworks-and-Engineering.md) | [中文](../前端知识库/06-前端框架与工程化综合.md) | Vue, Next.js, micro-frontends, build tools, CI/CD |
| 07 | [CSS In Depth and Layout](./07-CSS-In-Depth-and-Layout.md) | [中文](../前端知识库/07-CSS深入与布局.md) | box model, BFC, Flexbox, Grid, responsive design |
| 08 | [HTML5 and Web APIs](./08-HTML5-and-Web-APIs.md) | [中文](../前端知识库/08-HTML5与Web API.md) | semantic HTML, storage, workers, PWA, Canvas, WebRTC |
| 09 | [Frontend Performance Optimization](./09-Frontend-Performance-Optimization.md) | [中文](../前端知识库/09-前端性能优化.md) | metrics, loading, rendering, JavaScript, images, monitoring |
| 10 | [Mobile and Cross-platform Development](./10-Mobile-and-Cross-Platform-Development.md) | [中文](../前端知识库/10-移动端与跨端开发.md) | mobile adaptation, hybrid, React Native, Flutter, mini programs |

## English Interview Answering Tips

Use clear layers when answering:

- **Start with a concise definition**: "A closure is a function that keeps access to its lexical environment."
- **Explain the mechanism**: mention runtime data structures, scheduling, browser pipeline, or framework phases.
- **Compare trade-offs**: do not only say "A is better"; say what A optimizes and what it costs.
- **Add production practice**: talk about monitoring, rollback, compatibility, security, and measurable outcomes.
- **Close with boundaries**: mention cases where the technique is unnecessary or risky.

Useful phrases:

- "The key trade-off is..."
- "In production, I would guard this with..."
- "The failure mode I watch for is..."
- "I would measure it with..."
- "This is usually not worth it unless..."

## Maintenance Notes

- Keep file numbers aligned with `cs/前端知识库`.
- Add `Language: English | [中文](...)` at the top of every English document.
- When the Chinese version changes, update the English version in the same numbered file.
- Prefer practical interview English over literal translation.
- Avoid global index edits here; update root-level maps in a separate coordination step.
