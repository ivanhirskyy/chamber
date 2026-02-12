# Architecture

## Overview

Chamber is a component testing tool that uses iframe isolation to prevent framework pollution. The architecture separates concerns into two processes:

- **Lab**: React-based UI for controls and orchestration
- **Stage**: Isolated iframe that renders components

## Why Iframe + Shadow DOM?

Existing tools (Storybook, Styleguidist) run components in a shared context. This causes:

- CSS leaks between components
- Global state pollution
- Framework version conflicts
- Inaccurate performance measurements

Chamber solves this with:

1. **Iframe**: Process-level isolation in modern browsers
2. **Shadow DOM**: Style encapsulation within the iframe
3. **MessageChannel**: Type-safe bi-directional communication

## Communication Protocol

We use `MessageChannel` instead of `window.postMessage` for:

- Dedicated port-to-port communication (no message collision)
- Transferable objects support
- Better performance

### Message Flow

```typescript
// packages/protocol/index.ts
enum KernelAction {
  INIT_CONNECTION = "INIT_CONNECTION",
  RENDER_COMPONENT = "RENDER_COMPONENT",
  UPDATE_PROPS = "UPDATE_PROPS",
  REPORT_ERROR = "REPORT_ERROR",
}

interface IKernelMessage<T> {
  id: string;           // UUID for request-response
  action: KernelAction;
  payload: T;
}
```

Host creates MessageChannel and transfers Port 2 to Stage:

```typescript
// Lab side
const channel = new MessageChannel();
iframe.contentWindow.postMessage(
  { action: "INIT_CONNECTION" },
  "*",
  [channel.port2]
);

// Stage side
window.addEventListener("message", (event) => {
  const port = event.ports[0];
  port.onmessage = handleMessage;
});
```

## Framework Strategy Pattern

Each framework has different mounting APIs. We abstract this:

```typescript
interface IRenderStrategy {
  mount(container: ShadowRoot, module: unknown, props: Record<string, unknown>): Promise<void>;
  update(props: Record<string, unknown>): void;
  unmount(): void;
}
```

Implementations:

- `ReactStrategy`: Uses `ReactDOM.createRoot()`
- `VueStrategy`: Uses `createApp().mount()`
- `WebComponentStrategy`: Uses `customElements.define()`

Factory lazy-loads strategies to minimize bundle size:

```typescript
class RenderStrategyFactory {
  static async getStrategy(framework: string): Promise<IRenderStrategy> {
    switch (framework) {
      case "react":
        const { ReactStrategy } = await import("./ReactStrategy");
        return new ReactStrategy();
      // ...
    }
  }
}
```

## Security

### Sandbox Attributes

Stage iframe uses:

```html
<iframe sandbox="allow-scripts allow-same-origin">
```

This prevents:
- Cookie access
- localStorage access
- Top-level navigation

### Origin Validation

In production, replace `postMessage("*")` with specific origin:

```typescript
iframe.contentWindow.postMessage(msg, "http://localhost:5174");
```

Stage validates `event.origin` before processing messages.

## Performance Budget

Target: 60fps in Lab UI, even with slow components.

Strategies:

1. **Iframe Isolation**: Component runs in separate process (Chromium architecture)
2. **Debounced Updates**: Prop changes throttled to 16ms
3. **Web Workers**: AST parsing offloaded from main thread (future)

## Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| Stage iframe crashes | Component disappears | Host detects missing heartbeat, shows error |
| Invalid ESM URL | Component fails to load | Catch in `import()`, display error |
| Strategy throws | Render fails | Try-catch in `StageManager`, report to Host |

## Decision Records

### ADR-001: MessageChannel over postMessage

**Context**: Need bi-directional communication

**Decision**: Use MessageChannel

**Rationale**: Dedicated ports avoid message collisions, support transferable objects

**Trade-off**: Slightly more complex setup

### ADR-002: Iframe over Web Worker

**Context**: Need to render visual components in isolation

**Decision**: Iframe with Shadow DOM

**Rationale**: Web Workers can't access DOM

**Trade-off**: Requires origin management for security

## Future Work

- AST parser for auto-generating prop controls
- Performance timeline integration
- Accessibility audit runner
- Stress test suite (render 1000+ instances)
