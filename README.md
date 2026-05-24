# fetchsmith 🔨

> Zero-dependency, TypeScript-first HTTP client with a fluent API, middleware pipeline, and smart retry logic.

[![npm version](https://img.shields.io/npm/v/fetchsmith.svg)](https://www.npmjs.com/package/fetchsmith)
[![bundle size](https://img.shields.io/bundlephobia/minzip/fetchsmith)](https://bundlephobia.com/package/fetchsmith)
[![CI](https://github.com/yourusername/fetchsmith/actions/workflows/ci.yml/badge.svg)](https://github.com/yourusername/fetchsmith/actions)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-blue)](https://www.typescriptlang.org/)
[![license](https://img.shields.io/npm/l/fetchsmith)](./LICENSE)

---

## Why fetchsmith?

Most HTTP libraries are either too heavy (Axios ~13kb) or too bare-bones (raw fetch). fetchsmith hits the sweet spot:

- **Zero dependencies** — just a thin wrapper over native `fetch`
- **Fluent builder API** — readable, chainable request construction
- **Full TypeScript generics** — response types flow through automatically
- **Middleware pipeline** — composable interceptors for auth, logging, caching
- **Smart retry** — exponential backoff, configurable status codes
- **~2kb gzipped** — negligible bundle impact

---

## Install

```bash
npm install fetchsmith
# or
pnpm add fetchsmith
# or
yarn add fetchsmith
```

Requires Node 18+ or any modern browser with native `fetch`.

---

## Quick start

```typescript
import { createClient } from 'fetchsmith';

const api = createClient('https://api.example.com');

// GET with type inference
interface User {
  id: number;
  name: string;
  email: string;
}

const users = await api.get<User[]>('/users').query({ page: 1, limit: 20 });

// POST with body
const newUser = await api
  .post<User>('/users')
  .body({ name: 'Alice', email: 'alice@example.com' })
  .data();

// PUT / PATCH / DELETE work the same way
await api.delete('/users/42').send();
```

---

## Core concepts

### Client configuration

```typescript
const api = createClient({
  baseUrl: 'https://api.example.com',
  headers: { 'X-Api-Version': '2' },
  timeout: 8_000,      // 8 seconds
  retries: 3,          // retry up to 3 times
  retryDelay: 500,     // 500ms base delay (doubles each attempt)
  retryOn: [429, 503], // status codes that trigger a retry
});
```

### Immutable client cloning

Clients are immutable — modifier methods return a new instance, keeping the original unchanged. This is great for building scoped clients.

```typescript
const base = createClient('https://api.example.com');

// Authenticated variant — doesn't affect `base`
const authed = base.withAuth(getToken());

// Per-service clients
const usersApi = base.withBaseUrl('https://users.example.com');
const paymentsApi = base.withBaseUrl('https://payments.example.com');
```

### Request builder

Every HTTP method returns a `RequestBuilder` you can chain before calling `.send()` or `.data()`.

```typescript
const response = await api
  .get<Product[]>('/products')
  .query({ category: 'books', inStock: true })
  .header('Accept-Language', 'en-US')
  .timeout(5_000)
  .retry(2)
  .send(); // returns full ResponseContext<Product[]>

console.log(response.status);  // 200
console.log(response.headers); // Headers
console.log(response.data);    // Product[]
```

Use `.data()` when you only need the response body:

```typescript
const products = await api.get<Product[]>('/products').data();
```

Or `await` the builder directly (sugar for `.data()`):

```typescript
const products = await api.get<Product[]>('/products');
```

### Response types

```typescript
// JSON (default)
const data = await api.get<User>('/me').data();

// Plain text
const html = await api.get('/page').as('text').data(); // string

// Blob (e.g. file download)
const blob = await api.get('/export').as('blob').data(); // Blob

// ArrayBuffer (binary data)
const buf = await api.get('/binary').as('arrayBuffer').data();

// Discard body
await api.delete('/session').as('void').send();
```

---

## Middleware

fetchsmith uses a Koa-style `(ctx, next)` middleware pipeline. Middleware is async and can run logic before and after the request.

```typescript
import { createClient, type Middleware } from 'fetchsmith';

const timingMiddleware: Middleware = async (ctx, next) => {
  const start = Date.now();
  const response = await next();
  console.log(`${ctx.method} ${ctx.url} → ${Date.now() - start}ms`);
  return response;
};

const api = createClient('https://api.example.com').use(timingMiddleware);
```

### Built-in middleware

#### `logger` — request/response logging

```typescript
import { createClient, logger } from 'fetchsmith';

const api = createClient('https://api.example.com').use(logger());

// Custom log function
const api2 = createClient('https://api.example.com').use(
  logger({ log: (msg, data) => myLogger.info(msg, data), logBody: true })
);
```

#### `authToken` — dynamic token injection

Perfect for JWT tokens that can expire.

```typescript
import { createClient, authToken } from 'fetchsmith';

const api = createClient('https://api.example.com').use(
  authToken(async () => {
    const token = await tokenStore.getOrRefresh();
    return token;
  })
);
```

#### `cache` — in-memory response caching

```typescript
import { createClient, cache } from 'fetchsmith';

const api = createClient('https://api.example.com').use(
  cache({ ttl: 30_000 }) // cache GET responses for 30 seconds
);
```

#### `dedupe` — deduplicates in-flight requests

Prevents parallel identical requests from hitting the server twice.

```typescript
import { createClient, dedupe } from 'fetchsmith';

const api = createClient('https://api.example.com').use(dedupe());

// Only ONE network request is made, both get the same result
const [a, b] = await Promise.all([
  api.get('/config').data(),
  api.get('/config').data(),
]);
```

#### `rateLimit` — client-side rate limiting

```typescript
import { createClient, rateLimit } from 'fetchsmith';

const api = createClient('https://api.example.com').use(
  rateLimit({ max: 10, windowMs: 1000 }) // max 10 req/second
);
```

### Composing middleware

```typescript
const api = createClient({
  baseUrl: 'https://api.example.com',
  middleware: [
    logger(),
    authToken(() => getAccessToken()),
    cache({ ttl: 10_000 }),
    dedupe(),
  ],
});
```

---

## Error handling

```typescript
import { FetchSmithError, TimeoutError, NetworkError } from 'fetchsmith';

try {
  const user = await api.get<User>('/users/999').data();
} catch (err) {
  if (err instanceof TimeoutError) {
    console.error('Request timed out:', err.message);
  } else if (err instanceof NetworkError) {
    console.error('Network failure:', err.message);
  } else if (err instanceof FetchSmithError) {
    console.error(`HTTP ${err.status}:`, err.message);
    console.log('Response body:', err.response?.data);
  }
}
```

---

## Real-world example

```typescript
// src/api/client.ts
import { createClient, logger, authToken, cache, dedupe } from 'fetchsmith';
import { getSession } from './auth';

export const api = createClient({
  baseUrl: import.meta.env.VITE_API_URL,
  timeout: 10_000,
  retries: 2,
  retryDelay: 500,
  retryOn: [429, 502, 503, 504],
  middleware: [
    logger(),
    authToken(async () => {
      const session = await getSession();
      if (!session) throw new Error('Not authenticated');
      return session.accessToken;
    }),
    dedupe(),
    cache({ ttl: 30_000 }),
  ],
});

// src/api/posts.ts
export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

export const getPosts = (page = 1) =>
  api.get<Post[]>('/posts').query({ page, limit: 20 }).data();

export const getPost = (id: number) =>
  api.get<Post>(`/posts/${id}`).data();

export const createPost = (data: Omit<Post, 'id'>) =>
  api.post<Post>('/posts').body(data).data();

export const updatePost = (id: number, data: Partial<Post>) =>
  api.patch<Post>(`/posts/${id}`).body(data).data();

export const deletePost = (id: number) =>
  api.delete(`/posts/${id}`).as('void').send();
```

---

## API Reference

### `createClient(config?)`

Creates a new `FetchSmith` client instance.

| Option | Type | Default | Description |
|---|---|---|---|
| `baseUrl` | `string` | — | Base URL prepended to all requests |
| `headers` | `Record<string, string>` | `{}` | Default headers for all requests |
| `timeout` | `number` | `10000` | Request timeout in milliseconds |
| `retries` | `number` | `0` | Number of retry attempts |
| `retryDelay` | `number` | `300` | Base delay between retries (doubles each attempt) |
| `retryOn` | `number[]` | `[429,502,503,504]` | Status codes that trigger a retry |
| `middleware` | `Middleware[]` | `[]` | Middleware pipeline |

### `RequestBuilder<T>` methods

| Method | Description |
|---|---|
| `.header(name, value)` | Set a request header |
| `.headers(record)` | Set multiple headers |
| `.query(record)` | Set query parameters |
| `.param(name, value)` | Set a single query param |
| `.body(data)` | Set JSON body |
| `.text(data, contentType?)` | Set plain-text body |
| `.auth(token)` | Set `Authorization: Bearer <token>` |
| `.timeout(ms)` | Override timeout |
| `.retry(count, delay?)` | Override retry count |
| `.retryOn(statuses)` | Override retry status codes |
| `.as(type)` | Set response type (`json`, `text`, `blob`, `arrayBuffer`, `void`) |
| `.use(middleware)` | Add middleware for this request only |
| `.send()` | Execute → `ResponseContext<T>` |
| `.data()` | Execute → `T` (body only) |

---

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](./CONTRIBUTING.md) and open an issue before submitting large PRs.

```bash
git clone https://github.com/yourusername/fetchsmith
cd fetchsmith
npm install
npm test
npm run build
```

---

## License

[MIT](./LICENSE) © Your Name
