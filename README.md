# @joaquimserafim/fetchx

Tiny, dependency-free `fetch` wrapper for **Node** and **browsers**. Adds the
things the platform `fetch` makes you write by hand: timeouts, signal merging,
a typed JSON helper, and a client factory with shared defaults.

- **Zero runtime dependencies.** ~1 kB gzipped (ESM).
- **Isomorphic.** Works wherever `globalThis.fetch` exists (Node 18+, modern browsers, edge runtimes, workers).
- **Type-safe.** First-class TypeScript types, dual ESM/CJS build.
- **Composable.** `fetchx` is a thin shell over `fetch`; `fetchx.json` and `createClient` build on top.

## Install

```sh
pnpm add @joaquimserafim/fetchx
# or: npm i @joaquimserafim/fetchx
# or: yarn add @joaquimserafim/fetchx
```

## Quick start

```ts
import { fetchx, createClient, HttpError } from "@joaquimserafim/fetchx";

// 1. Plain request with a timeout (default 10s).
const res = await fetchx("https://api.example.com/health", { timeout: 2000 });
// → Response

// 2. Typed JSON, errors include the parsed response body.
type User = { id: number; name: string };
const user = await fetchx.json<User>("https://api.example.com/users/1");
// → User                          e.g. { id: 1, name: "Ada" }

// 3. A reusable client with a baseUrl and shared headers.
const api = createClient({
    baseUrl: "https://api.example.com",
    headers: { authorization: `Bearer ${token}` },
    timeout: 5000,
});

const me = await api.json.get<User>("/me");
// → User
await api.json.post("/orders", { sku: "abc", qty: 1 });
// → unknown                       (pass a generic to type the body)
```

## API

### `fetchx(url, options?)`

Drop-in replacement for `fetch` with a timeout. Returns a `Response` (or a
`{ response, durationMs }` object when `withDuration: true`).

```ts
const res = await fetchx(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(payload),
    timeout: 3000,             // ms; defaults to DEFAULT_TIMEOUT_MS (10_000)
    signal: controller.signal, // optional — merged with the internal timeout signal
    fetchImpl: myFetch,        // optional — snapshot of `fetch` to bypass monkey-patches
    withDuration: true,        // optional — resolve to { response, durationMs } instead
});
// → { response: Response, durationMs: number }   (because withDuration: true)
// → Response                                     (when withDuration is omitted)
```

- **Caller `signal` is preserved.** It's merged with the timeout signal via
  `AbortSignal.any` (Node 20.3+, modern browsers) or an event-based fallback.
  Either signal can abort the request.
- **Timeout reason is distinguishable.** When the timeout fires, the abort
  reason is an `Error` with `name === "TimeoutError"` — match on that to tell
  apart "we timed out" from "the caller cancelled".
- **`fetchImpl`** is read at call time, so dependency injection or test mocks
  work without retaining a stale reference.

### `fetchx.json<T>(url, options?)`

Convenience for JSON APIs. Attached to `fetchx`; wraps the same underlying call.

- Sets `accept: application/json` unless the caller already did.
- When `json` is provided, serializes it and sets `content-type: application/json`.
- Resolves to the parsed body typed as `T`. Empty responses (status 204/205,
  or empty body) resolve to `undefined`.
- Rejects with [`HttpError`](#httperror) on non-2xx, carrying the parsed body
  when available.

```ts
const user = await fetchx.json<User>("https://api.example.com/users/1");
// → User                          e.g. { id: 1, name: "Ada" }

const created = await fetchx.json<{ id: string }>("https://api.example.com/users", {
    method: "POST",
    json: { name: "Ada", age: 36 }, // serialized + content-type set
});
// → { id: string }                e.g. { id: "usr_42" }

// 204 No Content (or any empty body):
const ack = await fetchx.json("https://api.example.com/ping");
// → undefined
```

### `createClient(defaults?)`

Build a client with shared defaults. Per-request options override the defaults;
headers are merged (request wins on conflicts).

```ts
const api = createClient({
    baseUrl: "https://api.example.com",
    headers: { authorization: "Bearer …" },
    timeout: 5000,
    fetchImpl: customFetch, // optional
});

// Untyped — returns Response, same as `fetchx`.
await api.get("/health");                                  // → Response
await api.post("/events", JSON.stringify({ kind: "ping" }), {
    headers: { "content-type": "application/json" },
});                                                        // → Response

// Typed JSON — returns the parsed body, throws HttpError on non-2xx.
const me = await api.json.get<User>("/me");                // → User
const order = await api.json.post<{ id: string }>("/orders", { sku: "abc", qty: 1 });
// → { id: string }
await api.json.put("/orders/123", { qty: 2 });             // → unknown
await api.json.patch("/orders/123", { qty: 3 });           // → unknown
await api.json.delete("/orders/123");                      // → unknown  (undefined for 204)
```

**URL resolution** uses `new URL(path, baseUrl)`. Watch the trailing slash on
`baseUrl` — it follows standard `URL` semantics:

```ts
createClient({ baseUrl: "https://api.example.com/v1/" }).get("users");
// → https://api.example.com/v1/users

createClient({ baseUrl: "https://api.example.com/v1/" }).get("/users");
// → https://api.example.com/users   (leading slash resets the path)
```

Absolute URLs and `URL` instances are passed through unchanged.

### `HttpError`

Thrown by `fetchx.json` and `createClient` json helpers when the response status
is not 2xx.

```ts
try {
    await api.json.get("/admin");
} catch (err) {
    if (err instanceof HttpError) {
        err.status;     // number, e.g. 403
        err.statusText; // string, e.g. "Forbidden"
        err.body;       // parsed JSON body, or raw text, or undefined
        err.response;   // the original Response (headers, etc.)
    }
}
```

### `Result<T, E>`, `tryAsync`, `fetchx.try`, `fetchx.json.try`

Skip the `try`/`catch` ceremony for failures you already know how to handle.
Each `.try` variant resolves to a discriminated `Result` instead of rejecting:

```ts
export type Result<T, E = Error> =
  | { readonly ok: true;  readonly data: T }
  | { readonly ok: false; readonly error: E };
```

```ts
import { fetchx, tryAsync, HttpError } from "@joaquimserafim/fetchx";

const res = await fetchx.try("https://api.example.com/health", { timeout: 2000 });
if (res.ok) {
    res.data;     // Response
} else {
    res.error;    // Error (TimeoutError, network error, caller abort)
}
// → { ok: true, data: Response } | { ok: false, error: Error }

const u = await fetchx.json.try<User>("https://api.example.com/users/1");
if (u.ok) {
    u.data;       // User
} else {
    u.error;      // Error | HttpError (HttpError extends Error)
}
// → { ok: true, data: User } | { ok: false, error: Error }
```

**`tryAsync<T>(promise)`** is the generic primitive — wraps any promise into a
`Result`. Useful for `createClient` calls, where there's no per-method `.try`:

```ts
const api = createClient({ baseUrl: "https://api.example.com" });

const me = await tryAsync(api.json.get<User>("/me"));
if (me.ok) console.log(me.data);
else       console.error(me.error);
```

Non-`Error` rejections (`throw "boom"`, `Promise.reject(undefined)`) are
coerced into `new Error(String(reason))` so `res.error` is always an `Error`
instance — no defensive `instanceof` checks on the failure branch.

### `DEFAULT_TIMEOUT_MS`

The default timeout used when `timeout` is omitted (`10_000` ms). Exported
mainly for tests and for callers that want to align their own defaults.

## Recipes

### Distinguishing timeout from caller cancellation

```ts
try {
    await fetchx(url, { timeout: 2000, signal });
} catch (err) {
    if (err instanceof Error && err.name === "TimeoutError") {
        // we timed out
    } else {
        // caller aborted, or network error
    }
}
```

### Measuring round-trip time

```ts
const { response, durationMs } = await fetchx(url, { withDuration: true });
// response  → Response
// durationMs → number  (ms, rounded to 0.01)
metrics.histogram("api.latency", durationMs);
```

Only the success path returns `{ response, durationMs }`. Failures still
reject — wrap and measure in your caller if you need failure timings.

### Replacing `fetch` for testing

```ts
const stub = vi.fn().mockResolvedValue(new Response("{}"));
await fetchx(url, { fetchImpl: stub });
```

## License

MIT © [@joaquimserafim](https://github.com/joaquimserafim)
