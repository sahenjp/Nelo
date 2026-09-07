<p align="center">
  <img src="./assets/nelo-icon.svg" alt="Nelo" width="128" height="128">
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="./assets/nelo-wordmark-on-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="./assets/nelo-wordmark-on-light.svg">
    <img src="./assets/nelo-wordmark-on-light.svg" alt="Nelo — Every request owns its work." width="520">
  </picture>
</p>

<p align="center">
  <strong>Manage the lifetime of work owned by each request.</strong>
</p>

<p align="center">
  <img alt="Experimental" src="https://img.shields.io/badge/status-experimental-6d7178">
  <img alt="Version 0.2.0" src="https://img.shields.io/badge/version-0.2.0-2864dc">
  <img alt="Strict TypeScript" src="https://img.shields.io/badge/TypeScript-strict-3178c6">
  <a href="./LICENSE"><img alt="Apache-2.0" src="https://img.shields.io/badge/license-Apache--2.0-5bc8ad"></a>
</p>

<p align="center">
  English · <a href="./README.ja.md">日本語</a> · <a href="https://nelo.lattee.jp">Website</a>
</p>

Nelo is a Web Standards framework that treats each request as the owner of the work it starts. A
handler may return a `Response` while tasks are still running, resources are still open, a response
body is still being delivered, or explicitly deferred work has been transferred beyond the response.
Nelo keeps those lifetimes explicit.

> [!NOTE]
> **AI usage disclosure:** AI tools were used for most of the development of Nelo.

> Returning a `Response` is not the same as completing the request lifetime.

## Quick start

```ts
import { Nelo } from "@lasder/nelo";
import { serve } from "@lasder/nelo/node";

const app = new Nelo();

app.get("/users/:id", async (context) => {
  const user = context.fork("load-user", (signal) => fetchUser(context.params.id!, { signal }));
  return context.json(await user);
});

const server = serve(app, { port: 3000 });
await server.listen();
```

## Lifetime model

```text
Request lifetime
├── Handler scope
│   ├── middleware
│   ├── context.fork()
│   ├── context.forkScope() → nested task/resource lifetime
│   ├── context.deadline()
│   └── context.use()
├── Delivery scope
│   ├── Response.body
│   ├── context.delivery.fork()
│   ├── context.delivery.forkScope()
│   └── context.delivery.use()
└── Deferred scope
    └── context.defer() → explicit post-response ownership transfer
```

The handler scope closes after the handler finishes. The delivery scope remains active until the
body completes, fails, is cancelled, or the transport reports a disconnect. Deferred work is an
explicit transfer beyond the response and is tracked by adapters that support it. Resources are
released once in reverse acquisition order.

## Core API

| API                                      | Purpose                                                   |
| ---------------------------------------- | --------------------------------------------------------- |
| `app.fetch(request)`                     | Run routing, middleware, the handler, and owned delivery. |
| `context.fork(name, operation)`          | Start an eager task owned by the request.                 |
| `context.forkScope(name, operation)`     | Own a nested task/resource lifetime as one task.          |
| `context.signal`                         | Forward cooperative cancellation to request work.         |
| `context.deadline(duration)`             | Create a disposable signal with a shorter request budget. |
| `context.defer(name, operation)`         | Explicitly transfer best-effort work beyond the response. |
| `context.use(name, acquire, cleanup?)`   | Acquire and release a handler-owned resource.             |
| `context.delivery.fork(name, operation)` | Start work owned by response delivery.                    |
| `context.delivery.forkScope(...)`        | Own a nested lifetime through response delivery.          |
| `context.delivery.use(...)`              | Keep a resource or cleanup attached to delivery.          |

Deadlines accept milliseconds or values such as `750ms`, `2s`, `1m`, and `1h`. They preserve parent
cancellation, abort with a typed `deadline` reason on expiry, and are disposed automatically when
the handler scope closes.

`forkScope()` is additive to the existing `fork()` API. It is useful when one operation owns several
tasks, resources, or deadlines. Child lifetimes inherit cancellation and remain visible in
`handlerTree` or `deliveryTree` diagnostics. The previous low-level `forkChild()` method remains as
a deprecated compatibility alias.

### Deferred work

`context.defer(name, operation)` explicitly transfers best-effort work beyond the response. The
operation gets its own `AbortSignal`; its failure is observable as `NELO_DEFERRED_002`. A runtime
without deferred-work support fails explicitly with `NELO_DEFERRED_001` instead of silently creating
unowned background work.

The Node adapter implements process-tracked deferred work. `server.close()` first drains active HTTP
exchanges, then waits for registered deferred tasks within the remaining grace period. At grace
expiry remaining deferred work receives `server_shutdown`. This is in-memory process tracking, not a
durable queue, retry system, or exactly-once delivery guarantee.

## Security boundary

Nelo does not implicitly trust proxy forwarding headers. The Node adapter's `protocol: "https"`
option declares the public Fetch URL scheme but does not enable TLS; terminate TLS in a trusted
external server or proxy. Application-specific request-body limits remain the application's
responsibility. See [SECURITY.md](./SECURITY.md) before deploying an Internet-facing adapter.

## Lvau integration

[`examples/lvau-service`](./examples/lvau-service/mod.ts) runs the Lvau file-encryption CLI as
request-owned work. Client cancellation terminates the child process, temporary plaintext is removed
with the handler scope, uploads are bounded, and the password is read from a protected local file.

See [the integration guide](./docs/integrations/lvau.md) for setup and security boundaries.

## Brand assets

- [`assets/nelo-icon.svg`](./assets/nelo-icon.svg) — primary icon for documentation and product
  surfaces.
- [`assets/favicon.svg`](./assets/favicon.svg) — compact browser and site icon.

Both assets are flat SVGs with transparent backgrounds and can be referenced directly from websites.

## Development

```sh
git clone https://github.com/sahenjp/Nelo.git
cd Nelo
npm ci
npm run format:check
npm run lint
npm run typecheck
npm test
npm run build
npm run check:package
npm run check:tarball
```

See [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution and security expectations.

## License

Nelo is available under the [Apache License 2.0](./LICENSE).
