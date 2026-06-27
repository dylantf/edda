# Edda — Design Notes

A small web framework for Saga, sitting on top of [`saga_http`](https://github.com/dylantf/saga_http).
This is a living document — design decisions, open questions, and rationale
captured as we go.

## Goals

- **Minimal-ish.** Closer to Express or Hono than to Phoenix. More than a
  bare router, but not opinionated about databases, ORMs, templating, etc.
- **Composable parts.** Easy to plug pieces together. Few hard requirements
  about how an app is structured.
- **Lean on Saga's strengths.** Effects, handlers, row polymorphism,
  exhaustive pattern matching — the framework should feel like a natural
  use of the language, not a transplant of a JS/Python framework.

## The boundary with `saga_http`

`saga_http`'s entry point is:

```saga
pub fun serve : Config -> (Request -> Response) -> Result ShutdownHandle String
  needs {Process, Actor SupMsg, Monitor, Timer, Server}
```

The handler is **pure**: `Request -> Response`, no `needs` row. All
effects must be discharged before control returns to `serve`. Inside
the closure we hand to `serve`, we're free to attach as many `with`
stacks as we like — per-request, per-route, per-group, anything goes.

### Request and Response wrapping

We define our own `Edda.Request` rather than passing `SagaHttp.Request`
through. Reasons:

- `Edda.Request` carries a `params : Dict String String` field, populated
  by the router as `:name` segments match. `SagaHttp.Request` has no
  notion of route params.
- It carries `original_path` alongside `path` so that `group` and `mount`
  can strip the matched prefix from `path` while the original remains
  accessible to handlers that need it (logging, correlation).
- It's our extension point for future additions (parsed query, decoded
  body helpers) without changes upstream.

We currently **do not** wrap `Response` — saga_http's `text`, `bytes`,
and `stream` constructors are perfectly usable. We can add wrappers later
if we want sugar (e.g. JSON, redirects) that doesn't fit upstream.

### The boundary helpers

```saga
pub fun from_http  : Http.Request -> Request
pub fun to_handler : (Request -> Response) -> Http.Request -> Response
```

The simplest usage is one line:

```saga
serve config (to_handler app)
```

For ambient context, users skip `to_handler` and write their own boundary
(see [Ambient context handlers](#3-ambient-context-handlers) below).

## Routes and composition

Saga records cannot carry effect rows, so a route can't be a record of
`{ method, path, handler }`. Function types **can** carry rows, so routes
are functions composed via combinators in the Giraffe/Hono style.

### The `Skip` effect

A route handler is a function that either produces a response or signals
"I don't match — try the next one." That's an effect:

```saga
effect Skip {
  fun skip : Unit -> a
}
```

A route's type is therefore:

```saga
Request -> Response needs {Skip, ..e}
```

where `..e` is whatever the inner handler needs.

### The core combinators

```saga
type Method = GET | POST | PUT | DELETE | PATCH | OPTIONS | HEAD | Custom String

# Attach a method + path matcher to a handler.
fun route : Method -> String -> (Request -> Response needs {..e})
         -> Request -> Response needs {Skip, ..e}

# Try each route in order; if one skips, fall through to the next.
fun choose : List (Request -> Response needs {Skip, ..e})
          -> Request -> Response needs {..e}
```

`choose` discharges `Skip` by trying the next route in its list when one
skips. If nothing matches, it falls back to `not_found` (404).

```saga
let app = choose [
  route GET  "/users"     list_users,
  route POST "/users"     create_user,
  route GET  "/users/:id" show_user,
]
```

### Grouping and mounting

Two flavors of nesting:

**`group`** — inline grouping with shared effects. Sub-routes share the
parent's effect row.

```saga
fun group : String -> List (Request -> Response needs {Skip, ..e})
         -> Request -> Response needs {Skip, ..e}

let app = choose [
  group "/users" [
    route GET  "/"     list_users,
    route POST "/"     create_user,
    route GET  "/:id"  show_user,
  ],
]
```

If the prefix matches but no sub-route matches, `group` skips so the
outer `choose` can keep trying later routes. It is prefix composition,
not namespace ownership.

**`mount`** — attach a fully-discharged sub-app at a prefix. Useful when
a sub-app uses a different handler stack (admin DB pool, mock auth in
tests) or is its own module.

```saga
fun mount : String -> (Request -> Response)
         -> Request -> Response needs {Skip}

fun admin_app : Request -> Response
admin_app req = (choose [...]) req with { admin_db, log, audit }

let app = choose [
  mount "/admin" admin_app,
  mount "/api"   api_app,
]
```

### Path stripping

Both `group` and `mount` strip the matched prefix from `req.path` before
handing the request down. Sub-app authors think in relative paths.
`req.original_path` always carries the unmodified path.

### Path parameters

Stringly-typed. Pattern segments starting with `:` capture into the
params dict:

```saga
route GET "/users/:id" show_user

fun show_user : Request -> Response
show_user req = case param "id" req {
  Just id -> text 200 $"showing user {id}"
  Nothing -> text 400 "missing id"
}
```

Params accumulate across nesting:

```saga
group "/users/:id" [
  route GET "/posts" list_user_posts,   # `id` visible in inner handler
]
```

#### Future exploration

Two angles worth prototyping after v1:

- **Typed combinator DSL** (Servant-ish): `route GET (lit "/users" / int "id") show_user`
  with the handler typed as `Int -> Request -> Response`. Slick but
  Saga lacks HLists, so variadic-args plumbing needs design work. Also
  interacts awkwardly with grouping — each `group` would change the
  inner handler's expected arg list.
- **Path-as-effects**: describe the path as a sequence of effect
  operations (`lit!`, `next_int!`) that can be run with two handlers —
  one to extract the pattern for registration, another to parse a real
  request. Conceptually elegant; needs care so the "register" handler
  doesn't run the full route body.

Users can pattern-match on `req.path` themselves as an escape hatch at
any time.

## Middleware: three shapes

There's no `Middleware` type in Edda. What people call middleware in
other frameworks splits cleanly into three patterns here, each just
function composition or effect handling. The framework provides the
primitives; users compose them.

### 1. Wrap functions

A wrap is a function that takes a handler and returns a wrapped handler.
Uniform pre/post around every request flowing through it.

```saga
fun with_logging : (Request -> Response needs {..e})
                -> Request -> Response needs {Stdio, ..e}
with_logging inner req = {
  println $"--> {method_str req.method} {req.original_path}"
  let resp = inner req
  println $"<-- {resp.status}"
  resp
}
```

Wraps can short-circuit (don't call `inner`), modify the response,
perform any effects they need, and compose with each other:

```saga
with_logging (with_timing app) req
```

Use wraps for cross-cutting concerns that should apply uniformly:
request logging, timing, CORS headers, gzip, panic recovery,
security headers, body-size guards, or a header guard like
`require_api_key`.

### 2. Opt-in effect handlers

Routes declare capabilities they need via `needs`; the boundary
provides handlers. Different routes can need different effects.

```saga
fun account : Request -> Response needs {Auth}
account _ = {
  let user = current_user! ()
  text 200 $"account page for {user}"
}
```

Handlers can short-circuit by not calling `resume`:

```saga
current_user () = case find_header "x-user" req.headers {
  Just "alice" -> resume "Alice"
  Just "bob"   -> resume "Bob"
  _            -> text 401 "unauthorized"   # don't resume → 401 is the result
}
```

Use opt-in effects for capabilities specific routes need: DB access,
email sending, feature flags, structured logging that emits domain
events, auth that varies per route, anything you'd want to swap in tests.

### 3. Ambient context handlers

Same shape as opt-in effects, but the handler is installed *per request*
inside the user's boundary function, so it can close over the request:

```saga
fun handle : Http.Request -> Response
handle hr = {
  let req = from_http hr
  let rid = gen_request_id ()
  app req with {
    request_id   () = resume rid,
    current_user () = case find_header "x-user" req.headers {
      Just "alice" -> resume "Alice"
      Just "bob"   -> resume "Bob"
      _            -> text 401 "unauthorized"
    },
  }
} with console
```

Routes pull values from these handlers:

```saga
fun whoami : Request -> Response needs {ReqCtx}
whoami _ = {
  let rid = request_id! ()
  text 200 $"your request id: {rid}"
}
```

Use ambient context for request-scoped values: current user, request id,
parsed JWT claims, feature flag set, anything that's "the same for this
request, different across requests."

### When to reach for which

| Shape | Lives at | Use when |
| --- | --- | --- |
| Wrap function | `fun outer inner = ... ; outer (inner app) req` | Uniform behaviour for everything under it; cross-cutting; no per-route variation |
| Opt-in effect | `with { ... }` at the boundary | A capability some routes need; swap for tests; varies by environment |
| Ambient context | `with { ... }` inside the boundary, closing over `req` | Request-scoped values pulled by routes that opt in |

The smell test: "should every request get this?" → wrap. "Should the
route declare it needs this?" → opt-in effect. "Does this value depend
on the request itself?" → ambient context.

The three compose freely. A real app might have wraps for
logging/timing/CORS, opt-in effects for DB and auth, and ambient
context for the request ID and current user.

## Capability-based routing

Saga's effect system gives us capability-based routing for free. Each
route's `needs` clause is its capability declaration, enforced at
compile time:

```saga
fun list_users : Request -> Response needs {Db, Log}
fun read_pii   : Request -> Response needs {Db, Log, Auth, PII}
```

The second route can't accidentally skip auth, can't accidentally log
PII outside the PII handler, can't read from anywhere except what its
declared effects allow. The first route physically cannot access `PII`
because it didn't ask for it.

This is honest at the **route** level — pure routes and effectful routes
coexist in the same `choose` list, each declaring only what it actually
uses. The router doesn't force a uniform effect set across the app.

(See [Notes on the compiler interaction](#notes-on-the-compiler-interaction)
below for why this took a few rounds to land.)

## JSON

Lives in `Edda.Json`. Built on `saga_json`'s `ToJson` / `FromJson` traits,
so a record with `deriving (ToJson, FromJson)` round-trips through HTTP
without a hand-written codec.

### Encoding: `json`

```saga
pub fun json : Int -> a -> Response where {a: ToJson}
```

Sets `Content-Type: application/json` and serializes via `E.serialize`.

```saga
record User { id: Int, name: String } deriving (ToJson)

fun show : Request -> Response
show _ = json 200 (User { id: 1, name: "Alice" })
```

Lists, `Maybe`, tuples, and the primitive types all have built-in
`ToJson` impls, so `json 200 [u1, u2, u3]` works without ceremony.

### Decoding: `body_json` + `BodyError`

```saga
pub type BodyError =
  | NoBody
  | NotUtf8
  | JsonError J.Error

pub fun body_json : Request -> Result a BodyError where {a: FromJson}
pub fun body_error_response : BodyError -> Response
```

`BodyError` keeps body-level problems (missing, not UTF-8) distinct
from JSON-level ones (malformed, wrong shape) so the 400 message can
say something useful. Saga needs an annotation on the call to pick the
right `FromJson` impl:

```saga
case (body_json req : Result CreateUser BodyError) {
  Err e -> body_error_response e
  Ok input -> json 201 (created_from input)
}
```

`body_error_response` is a reasonable default: it flattens the
`InvalidShape` path into a JSON-pointer-ish string (`/email`, `/address/zip`)
so clients see exactly where their payload went wrong. Apps with a
structured error envelope can write their own mapping.

### Two error-handling shapes

Routes that decode bodies tend to end up in one of two patterns,
demonstrated in `Demo.JsonApi`:

**Inline match.** Keep the failure path in the route body. Best when a
handful of routes parse bodies and each wants its own response shape.

```saga
fun create_user : Request -> Response
create_user req = case (body_json req : Result CreateUser BodyError) {
  Err e -> body_error_response e
  Ok input -> ...
}
```

**Per-app effect.** Declare a `Body` effect, route declares it in
`needs`, the boundary handler maps decode failures to a response once.
Best when many routes share the same body type and the same 400
behavior.

```saga
effect Body { fun decode_create_user : Unit -> CreateUser }

fun create_user : Request -> Response needs {Body}
create_user _ = {
  let input = decode_create_user! ()
  ...  # happy path only
}

# at the boundary
app req = ... with {
  decode_create_user () = case (body_json req : Result CreateUser BodyError) {
    Ok v -> resume v
    Err e -> body_error_response e
  }
}
```

Edda intentionally ships only the primitive (`body_json`) and the
default mapping (`body_error_response`). The effect pattern is just
the [opt-in effect handler](#2-opt-in-effect-handlers) shape applied
to body decoding — there's nothing framework-specific to add.

## Endpoint specs and OpenAPI

`Edda.Spec` is an experimental sidecar, not the future primary routing API.
The primary API remains the small function-combinator surface in `Edda`:
`route`, `choose`, `group`, and `mount`.

The reason to keep `Edda.Spec` in-tree for now is practical: it lets us
prototype type-driven route contracts against the real router and explore
whether one route definition can serve both dispatch and OpenAPI generation.
That experiment may eventually become a separate package, stay as an optional
sidecar module, or feed improvements back into the core API. Until then, app
authors should treat it as exploratory and write ordinary routes for the stable
path.

The typed response promise should stay honest. Plain `responds` is metadata
only; it records a status/content pair but does not prove the handler returns
that shape. Typed helpers such as `responds_success_json` derive the success
schema from the same builder phantom that `performed_by` unifies with the
handler's `TypedResponse a`, so that specific path keeps schema and handler
return type coupled.

## Open questions

### Router data structure

Currently a linear list matched top-to-bottom (via `choose`). Fine for
small/medium apps. Could upgrade to a trie of path segments for faster
lookup and natural prefix grouping. Defer until measured.

### Better request IDs

The demo uses `monotonic_ms ()` which is a large negative number —
correct but ugly. Real apps want UUIDv7, an atomic counter via `Std.Ref`,
or a Hex package via FFI. Not a framework concern, but worth a recipe
in docs.

### A `to_handler_with` helper

Right now, ambient context requires the user to write their own boundary
function. A `to_handler_with` that takes a "per-request handler installer"
might be worth providing if the pattern is common enough. Defer until we
have multiple real apps to see what shape feels right.

## Notes on the compiler interaction

Several rounds of Saga compiler work happened while building this
prototype. Each unblocked a piece of the design we wanted but couldn't
express. Worth recording because they shaped the shape of things.

1. **Indirect call to effectful function inside `with`** — calling
   `r req with { skip () = ... }` where `r` came from a list. The CPS
   evidence wasn't threaded at the call site. Fixed; this is what made
   `choose` work.
2. **Indirect call to effectful function as bare/tail expression** —
   `inner req` inside a wrap function, where `inner` is row-polymorphic
   and there's no `with` immediately around the call. Crashed with
   badarity (or silently no-op'd if it was the tail expression).
   Fixed; this is what made wrap functions work.
3. **Row widening for list literals** — a `List (T needs {..e})` couldn't
   hold elements with different concrete rows. Fixed; this is what made
   capability-based routing land at the route level instead of being
   forced up to the sub-app level. Followed by a runtime fix for
   partial-application closures whose row was solved narrowly but
   widened at the use site.
4. **Multi-arm handler dispatch on ADT constructors** — a handler with
   several arms keyed on variant patterns (`fail (NotFound m) = ...`,
   `fail (BadRequest m) = ...`) silently installed only the last arm;
   calls with other variants crashed with `case_clause`. Fixed; this is
   what made the typed-`Fail` error pattern in `Demo.ErrorMiddleware`
   readable instead of a single dispatch table.
5. **`catch_panic` over BEAM exceptions** — Saga's `panic` was caught
   cleanly, but native BEAM exceptions (`badarith` from `10 / 0`, etc.)
   weren't surfaced as `Err`, so `with_panic_recovery` crashed instead
   of producing a 500. Fixed; the panic-recovery wrap is now safe to
   put at the top of any app.
6. **Internal modules of a library dep** — a library's exposed module
   (`SagaJson`) internally imported a non-exposed sibling
   (`SagaJson.Parser`), and consumers couldn't resolve the transitive
   import. Fixed at the build-map level; libraries can have private
   internal modules without exposing them.

What we did right by accident: the design tried hard to express the
"natural" shape (heterogeneous-effect routes in a single list, wraps
that compose, handlers that close over the request) and we kept hitting
real bugs rather than redesigning around them. Each fix made the
framework strictly more expressive; we didn't paint ourselves into a
corner.

What this means for the framework today: **the design works as
intended** with the natural signatures. No artificial annotations, no
warnings, no workarounds. Pure routes are pure. Effectful routes
declare what they use. They mix freely.
