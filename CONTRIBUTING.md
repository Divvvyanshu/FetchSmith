# Contributing to fetchsmith

Thanks for your interest in contributing! Here's how to get started.

## Development setup

```bash
git clone https://github.com/yourusername/fetchsmith
cd fetchsmith
npm install
```

## Scripts

| Command | Description |
|---|---|
| `npm test` | Run the test suite |
| `npm run typecheck` | Run TypeScript type checking |
| `npm run build` | Build the library to `dist/` |
| `npm run dev` | Build in watch mode |

## Project structure

```
src/
├── index.ts       — Public exports
├── types.ts       — Shared TypeScript types
├── client.ts      — FetchSmith client class
├── builder.ts     — RequestBuilder fluent API
├── executor.ts    — Fetch execution, retry, middleware pipeline
├── errors.ts      — Custom error classes
└── middleware.ts  — Built-in middleware (logger, cache, dedupe, etc.)
tests/
└── client.test.ts — Integration tests using Node's built-in test runner
```

## Pull Request guidelines

- Keep PRs focused on a single change
- Add tests for any new functionality
- Run `npm run typecheck && npm test` before opening a PR
- Follow the existing code style (TypeScript strict mode, ESM)

## Reporting bugs

Please open a GitHub Issue with a minimal reproduction case.

## Feature requests

Open an Issue describing the use case before implementing anything large — it helps ensure the feature fits the library's goals (minimal, composable, zero-dependency).
