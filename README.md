# Chamber

A framework-agnostic component playground for testing React, Vue, and Web Components in isolation.

## Why?

Existing tools like Storybook run components in a shared context, making true isolation impossible. Chamber uses an iframe + Shadow DOM architecture to test components without framework pollution.

## Architecture

Chamber uses a two-process architecture:

- **Lab** (React app): The control plane. Manages UI and orchestration.
- **Stage** (Iframe): The execution plane. Renders components in complete isolation.

Communication happens via MessageChannel API for type-safe, bi-directional messaging.

## Project Structure

```
chamber/
├── apps/
│   ├── lab/          # React host app
│   └── stage/        # Isolated iframe sandbox
└── packages/
    ├── protocol/     # Message types & communication
    └── config/       # Shared TS/lint config
```

## Getting Started

```bash
pnpm install
pnpm dev
```

Opens:

- Lab at http://localhost:5173
- Stage at http://localhost:5174

## Roadmap

- [x] MessageChannel protocol
- [x] Iframe sandbox setup
- [ ] React/Vue/WC strategy adapters
- [ ] AST-based prop controls
- [ ] Performance monitoring

## Tech Stack

- TypeScript (strict mode)
- React 19 (Lab)
- Vite (bundler)
- pnpm workspaces + Turbo

## License

MIT
