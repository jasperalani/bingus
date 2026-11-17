# Bingus: framework plan

Draft v0.2. The repo is empty, so nothing here is constrained by existing code.

Bingus is a web application framework in TypeScript, running on **Bun**, built to learn how web frameworks work. That goal drives most of the choices below: where there is a trade-off between "what a production framework would do" and "what teaches the most", this plan picks the second.

**Decided so far**

- Runtime and toolchain: Bun (D2, D11).
- Backend HTTP framework first, in the Express / Koa / Hono / Elysia family, designed so that full-stack features can be layered on later (D1).
- The router and cookie handling are written by hand. Bun's built-in `routes` table and cookie helpers are not used (D4).

Still open: D3 and D5 onward. The recommendations stand as defaults until you say otherwise.

**What choosing Bun changes:** Bun's server already speaks `Request` in, `Response` out, and Bun runs, tests and type-checks TypeScript itself. So the transport layer and the tooling get much thinner, and more of the project is spent on framework logic. The cost is that you see less of what happens between the socket and the `Request` object. That part is kept as an optional side quest.

---

## 1. Goals and non-goals

**Goals**

- Understand every layer between "a request arrived" and "a route handler ran" by writing it.
- Zero runtime dependencies in the core. If bingus needs a cookie parser, you write the cookie parser.
- Use Bun as the runtime and toolchain, but not as the framework (see D4).
- Every module small enough to read in one sitting.
- Good enough to build a small real app on, because using it is how you find out what is wrong with it.

**Non-goals**

- Production readiness, security hardening beyond the basics, or competing on benchmarks.
- Compatibility with Express, Koa or Hono middleware.
- Running on Node. The core will be close to portable anyway, since it is written against web-standard types, but nothing will be tested there.

---

## 2. Decisions to make

D1 to D4 shape everything else. The rest can be decided when you reach the relevant milestone.

### D1. What kind of framework is it?

| Option | Examples | What you would be building |
|---|---|---|
| A. Backend HTTP framework | Express, Koa, Hono, Elysia | Server, router, middleware, request/response handling |
| B. Full-stack framework | Next, Remix, SvelteKit | Everything in A, plus rendering, file-based routing, a dev server, and client-side code delivery |
| C. Frontend UI framework | React, Solid, Vue | Reactivity, a component model, DOM rendering. No server at all |

**Decided: A, structured so it can grow into B.** A is the foundation B sits on, so nothing is wasted. C shares almost no code with A and is best treated as a second project.

### D2. Which request/response model?

**Decided by Bun: web-standard `Request` / `Response`.** `Bun.serve` takes a `fetch(request)` function that returns a `Response`, so the whole framework reduces to building one such function:

```ts
Bun.serve({ fetch: app.fetch, port: 3000 });
```

Two consequences worth knowing up front:

- The "server adapter" is a few lines, not a module.
- Everything is testable by calling `app.fetch(new Request(...))` directly, with no port and no network.

### D3. How low do you start?

| Option | Notes |
|---|---|
| `Bun.serve` | Bun parses HTTP and hands you a `Request`. You start at "a request arrived" |
| `Bun.listen` (raw TCP) | You parse HTTP/1.1 yourself: request line, headers, chunked encoding, keep-alive |

**Recommendation: `Bun.serve` for the main line.** Writing an HTTP parser is a great exercise, but it teaches protocols more than frameworks. Listed as a side quest in section 3.

### D4. Which Bun built-ins are allowed?

This is the decision that choosing Bun creates. Bun already includes several things a framework normally provides: `Bun.serve` has a built-in `routes` table with path params, per-method handlers and wildcards, plus cookie helpers, static file responses and WebSockets. Using them would leave little to build.

**Decided: use Bun for I/O and tooling, write the framework logic yourself.** The router and cookies in particular are hand-written.

| Use Bun's version | Write your own |
|---|---|
| `Bun.serve` with only the `fetch` handler | Routing (do not use `routes`) |
| `Bun.file` for reading files from disk | Cookie parsing, serialising and signing |
| `bun test`, `bun --watch`, `bun install` | Static file middleware logic (MIME, ETag, ranges, path safety) |
| Built-in TypeScript and JSX transpiling | Body parsing limits, error handling, middleware |
| Web Crypto for HMAC | |

A useful extra exercise once your router works: benchmark it against Bun's built-in `routes` and read up on why theirs is faster.

### D5. Handler and middleware model

| Option | Shape | Notes |
|---|---|---|
| Express-style | `(req, res, next)` | Callback-based, mutates `res`. Historically important, awkward with async, and a poor fit for `Request`/`Response` |
| Onion (Koa/Hono-style) | `async (ctx, next)` | Code before `await next()` runs on the way in, code after runs on the way out |
| Lifecycle hooks (Fastify/Elysia-style) | `onRequest`, `beforeHandle`, `afterHandle`... | Explicit phases, more machinery |

**Recommendation: onion, with handlers returning a `Response`.** The compose function is about 20 lines and is the single most instructive piece of code in the project.

### D6. Router algorithm

| Option | Notes |
|---|---|
| Linear scan | Loop over routes, test each pattern. Around 30 lines. O(routes) per request |
| Radix trie | Tree keyed on path segments. O(path length) per request. What find-my-way (Fastify) and Hono's TrieRouter do |
| Single compiled regex | All routes merged into one regex. Fast and hard to follow |

**Recommendation: build linear first, then a trie behind the same interface, then benchmark both.** Doing it twice is the point.

### D7. How far to push the type system

In rough order of difficulty:

1. Path params inferred from the route string (`"/users/:id"` gives `{ id: string }`), using template literal types.
2. Context variables typed across middleware (an auth middleware adds `ctx.user`, and handlers after it see the type).
3. Validated input typed from a schema.
4. An end-to-end typed client generated from route definitions (Hono RPC / Elysia Eden / tRPC style).

**Recommendation: 1 is a milestone, 2 and 3 are stretch goals, 4 is out of scope for now.** Design the API as plain runtime code first and add types after. Type-level programming is a separate skill from framework design and is easier to learn when the runtime behaviour is already settled.

### D8. Route definition style

Code-defined (`app.get("/path", handler)`) or file-based (`routes/users/[id].ts`).

**Recommendation: code-defined.** File-based routing is a layer that scans a directory and calls the code-defined API, so it can be added later without changing the core.

### D9. Rendering (only if you go full-stack)

| Option | Notes |
|---|---|
| None | JSON and plain text only |
| Tagged template (`` html`<p>${x}</p>` ``) | Small, teaches escaping and XSS |
| Your own JSX runtime | A `jsx()` function plus `renderToString`. Teaches what JSX compiles to |
| Use React/Preact | Teaches integration, not rendering |

**Recommendation: tagged template first, own JSX runtime as a stretch goal.** Bun transpiles JSX out of the box, so pointing it at your own runtime is a `tsconfig.json` setting and not a build pipeline. Defer until the core is done.

### D10. API style

Functions and plain objects, or classes and decorators (NestJS style).

**Recommendation: functions.**

### D11. Tooling

Bun covers nearly all of it, which keeps dev dependencies close to zero.

| Job | Tool |
|---|---|
| Run TypeScript | `bun run src/index.ts`, no build step |
| Dev loop | `bun --watch` |
| Tests | `bun test` (Jest-style API from `bun:test`) |
| Type checking | `bun check`, or `bun test --check` to gate tests on types. `tsc --noEmit` is the fallback on older Bun versions |
| Packages | `bun install` |
| Bun API types | `@types/bun` as a dev dependency |
| CI | GitHub Actions with `oven-sh/setup-bun` |
| Layout | Single package until there is a real second package |

Things to know:

- Bun does not check types when it runs or tests your code. Type checking is always a separate step, so wire it into the `test` script or CI from day one.
- With TypeScript 6.0 and later, `@types` packages are no longer picked up automatically. Add `"types": ["bun"]` to `compilerOptions` in `tsconfig.json`.
- Use `"moduleResolution": "bundler"` so the type checker resolves imports the way Bun does.
- ESM only.

---

## 3. Parts to build

### Request lifecycle

This is the path a request takes, and the order of the list below follows it.

```
socket
  -> Bun.serve                 (Bun parses HTTP, builds a Request)
  -> app.fetch(request)
       -> middleware chain     (onion: in ...)
            -> router          (method + path -> handler + params)
            -> handler(ctx)    (returns Response)
       <- middleware chain     (... and back out)
       <- error boundary       (thrown errors -> Response)
  <- Bun.serve                 (writes the Response)
<- socket
```

Line counts below are rough guesses for a clean first version, not targets.

### Tier 0: the core

This is the minimum that counts as a framework.

| # | Part | What it does | What you learn | Size |
|---|---|---|---|---|
| 1 | App and `serve` | Holds routes and middleware, exposes `fetch(request)`, thin wrapper over `Bun.serve` | The "a server is a function" model | ~100 |
| 2 | Router | Matches method and path to a handler, extracts params, handles wildcards, 404 vs 405 | Path matching, precedence rules, trailing slashes, URL decoding | ~60 (linear), ~200 (trie) |
| 3 | Middleware compose | Chains middleware into one function | The onion model, async control flow, the "`next()` called twice" bug | ~30 |
| 4 | Context | Per-request object: params, query, headers in; `json()`, `text()`, `html()`, `redirect()` out | API ergonomics, lazy parsing | ~150 |
| 5 | Error handling | `HttpError` class, top-level catch, custom `onError` and `notFound` | Where errors can come from, what leaks to the client | ~60 |
| 6 | Body parsing | JSON, URL-encoded forms, text, with a size limit | Content-Type negotiation, why limits matter | ~80 |

### Tier 1: what makes it pleasant to use

| # | Part | What it does | What you learn |
|---|---|---|---|
| 7 | Route groups and mounting | Prefixes, sub-apps, per-group middleware | Composition, how routers nest |
| 8 | Cookies | Parse, serialise, sign | The cookie spec, attributes (`HttpOnly`, `SameSite`), HMAC signing |
| 9 | Static files | Serve a directory, reading with `Bun.file` | MIME types, ETags, conditional requests, range requests, path traversal attacks |
| 10 | Built-in middleware | Logger, CORS, security headers, compression | What each header is for, preflight requests |
| 11 | Test helper | `app.request("/path")` sugar over `app.fetch` | Why the `fetch`-shaped core pays off |
| 12 | Typed route params | Infer params from the path string | Template literal types, conditional types, `infer` |
| 13 | Typed context | Middleware extends the context type | Generics that accumulate through a chain |
| 14 | Validation | Check body, query and params against a schema, with inferred types | Schema design, parse-don't-validate |
| 15 | Streaming and SSE | Stream a response body, server-sent events | `ReadableStream`, flushing, connection lifetime |

### Tier 2: full-stack

Only relevant if D1 heads toward option B.

| # | Part | What you learn |
|---|---|---|
| 16 | HTML tagged template | Escaping, XSS, safe-by-default APIs |
| 17 | JSX runtime and `renderToString` | What JSX compiles to, tree walking, async components |
| 18 | File-based routing | Directory scanning, naming conventions, dynamic import |
| 19 | Sessions | Session stores, session cookies, CSRF |
| 20 | Multipart uploads | `multipart/form-data`, streaming to disk |
| 21 | Dev experience | An error page with a stack trace, restart vs hot reload (`bun --watch` and `--hot` do the heavy lifting) |
| 22 | CLI | `bingus dev`, `bingus new`, argument parsing |

### Tier 3: side quests

Each is a project in its own right. Pick them up for interest, not completeness.

- HTTP/1.1 parser on raw TCP with `Bun.listen`, as an alternative to `Bun.serve`
- A Node adapter (`node:http` to `Request`/`Response` and back), for the streams and backpressure lesson that Bun otherwise hides
- WebSocket routing on top of Bun's built-in WebSocket server
- Client-side hydration or islands, bundled with `Bun.build`
- Typed RPC client generated from routes
- OpenAPI generation from route definitions
- Dependency injection container

---

## 4. Scope

Three sizes. Each contains the one before it.

| Size | Contents | Rough code size | Comparable to |
|---|---|---|---|
| Small | Tier 0 | 400 to 700 lines | A minimal Koa |
| Medium | Tier 0 + Tier 1 | 2,000 to 3,000 lines | A small Hono |
| Large | Through Tier 2 | 5,000+ lines | A very small Remix |

**Recommendation: commit to Small, plan for Medium, treat Large as a menu.** Small is achievable in a handful of focused sessions and already teaches the core ideas. The decision about Large is better made after you have used Medium to build something.

---

## 5. Milestones

Each milestone ends with something that runs and has tests.

| # | Milestone | Done when |
|---|---|---|
| M0 | Repo setup: `bun init`, `tsconfig.json`, scripts, CI, README | `bun test` and the type check pass on one trivial test, locally and in CI |
| M1 | Hello world: `app.fetch` wired into `Bun.serve` | `curl localhost:3000` returns a response |
| M2 | Router v1 (linear): methods, params, wildcards, 404 and 405 | Router tests pass, including precedence and trailing-slash cases |
| M3 | Middleware compose, context, error handling | A logger middleware times requests, a thrown error becomes a 500 |
| M4 | Body parsing, response helpers, cookies | A JSON POST round-trips, a cookie can be set and read |
| M5 | **Example app** built on bingus (a todo API or link shortener, `bun:sqlite` for storage) | It works, and you have a list of everything that annoyed you |
| M6 | Router v2 (trie) and a benchmark against v1 and against Bun's built-in `routes` | Same tests pass on both of yours, benchmark numbers recorded |
| M7 | Typed route params and typed context | Wrong param names fail type checking |
| M8 | Route groups, static files, built-in middleware | The example app serves a frontend and uses CORS |
| M9 | Validation | The example app rejects bad input with a useful 400 |
| | **Checkpoint:** decide whether to go full-stack | |
| M10+ | Tier 2, in whatever order is interesting | |

M0 to M4 is the Small scope. M5 is the most important milestone on the list and the one most tempting to skip.

---

## 6. API sketch

A target to aim at, assuming the recommendations above. Expect it to change at M5.

```ts
import { createApp, HttpError } from "bingus";
import { logger } from "bingus/middleware";

const app = createApp();

app.use(logger());

app.get("/", (c) => c.text("hello from bingus"));

app.get("/users/:id", (c) => {
  const user = db.find(c.params.id);          // c.params.id is typed as string
  if (!user) throw new HttpError(404, "no such user");
  return c.json(user);
});

app.post("/users", async (c) => {
  const body = await c.req.json();
  return c.json(db.create(body), 201);
});

app.onError((err, c) => c.json({ error: err.message }, 500));

Bun.serve({ fetch: app.fetch, port: 3000 });
```

---

## 7. Repo layout

```
bingus/
  src/
    index.ts            public exports
    app.ts
    compose.ts
    context.ts
    errors.ts
    body.ts
    cookies.ts
    router/
      types.ts          the Router interface
      linear.ts
      trie.ts
    middleware/
      logger.ts
      cors.ts
      static.ts
  test/                 *.test.ts, run by bun test
  examples/
    todo-api/
  bench/
  docs/
    decisions/          one short file per decision: what, why, what was rejected
  package.json
  tsconfig.json
  PLAN.md
  README.md
```

---

## 8. Working practices

- **Read, then build.** Before each part, read how one existing framework does it, then close the tab and write your own. Good small reads: `koa-compose` (middleware), `path-to-regexp` (Express routing), `find-my-way` (Fastify's trie router), Hono's router directory (several algorithms side by side). Hono and Elysia are the closest relatives to what bingus will be, since both are built on `Request`/`Response`.
- **Tests against `fetch`.** Almost everything can be tested by calling `app.fetch(new Request(...))` and inspecting the `Response`. No ports, no network.
- **Keep a decisions log.** A few lines in `docs/decisions/` each time you pick one option over another. In a learning project, this log is worth as much as the code.
- **Use it.** Rebuild the example app at each checkpoint. API problems are invisible until you are the user.
- **Break it on purpose.** For each part, write down one way it could be attacked or misused (path traversal, huge bodies, header injection, `next()` called twice) and add a test.

---

## 9. Open questions

1. D3: do you want the raw TCP parser on the main path, or as a side quest? Default is side quest.
2. D5: onion middleware with handlers returning a `Response`, or one of the other models? Needs an answer before M3.
3. Is there a framework you already know well? Its design is a useful thing to compare against, or deliberately differ from.

---

## Sources

- [Bun docs: HTTP server (`Bun.serve`)](https://bun.sh/docs/api/http)
- [Bun docs: type checking (`bun check`)](https://bun.com/docs/runtime/check.md)
- [Bun docs: test runner](https://bun.sh/docs/test)
- [Bun docs: TypeScript 6 and `types`](https://bun-docs.pages.dev/docs/typescript-6)
