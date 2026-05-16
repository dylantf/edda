# Edda — Design Notes

A small web framework for Saga, sitting on top of [`saga_http`](https://github.com/dylantf/saga_http).
This is a living document — design decisions, open questions, and rationale
captured as we go.

## Goals

- **Minimal-ish.** Closer to Express or Hono than to Phoenix. More than a bare
  router, but not opinionated about databases, ORMs, templating, etc.
- **Composable parts.** Easy to plug pieces together. Few hard requirements
  about how an app is structured.
- **Lean on Saga's strengths.** Effects, handlers, row polymorphism, exhaustive
  pattern matching — the framework should feel like a natural use of the
  language, not a transplant of a JS/Python framework.

## The `serve` constraint

`saga_http`'s entry point is:

```saga
pub fun serve : Config -> (Request -> Response) -> Result ShutdownHandle String
  needs {Process, Actor SupMsg, Monitor, Timer, Server}
```

The handler is **pure**: `Request -> Response`, no `needs` row. This is a hard
boundary — the function we hand to `serve` has all its effects already
discharged.

For Edda, that means the framework's job at the seam is:

```saga
fun to_serve : App -> (Request -> Response)
to_serve app = fun req -> {
  dispatch app req            # has needs {Db, Log, Auth, ...}
} with { db, log, auth, ... }  # user-supplied handlers, attached here
```

The user attaches concrete handlers at `main`-time. The framework itself never
names specific handlers — it just funnels requests into whatever stack the user
provides.

### Consequence: effects must be discharged before HTTP

All a route's effects must be handled by the time control returns to `serve` —
that's the only constraint. The closure handed to `serve` is free to do
whatever it wants internally, including attaching different handler stacks to
different routes.

So per-route and per-group handler swapping are both possible: mock auth in a
test sub-app, a different DB pool for `/admin`, a stricter logger for `/api/v2`
— all just different `with` stacks at dispatch time.

What this means for the framework: routes (or route groups) likely want to
carry their own handler attachment, not inherit a single app-wide stack. We'll
need to decide where users declare those stacks — on the route itself, on a
group/sub-app, or at the call to `to_serve` — when we get to the route data
structure question below.

## Core ideas

### 1. Middleware as handlers

Effect handlers naturally express what middleware does in other frameworks.
Pre/post behaviour, short-circuiting, and response rewriting all fall out of
the existing handler machinery:

- Run code before `resume` → pre-middleware
- Run code after `resume` → post-middleware
- Don't `resume` → short-circuit (return early, e.g. auth failure)
- Use `return` → transform the final response (gzip, CORS headers, etc.)

```saga
handler timing for Timing needs {Clock, Log} {
  measure label = {
    let start = now! ()
    resume ()
    log! $"{label}: {now! () - start}ms"
  }
}
```

Whether Edda exposes a dedicated `Middleware` type on top of this is still
open. A type might still be worth having as a template — to standardise the
request/response shape, give middleware authors something concrete to
implement against, or to support patterns handlers don't cover cleanly. We'll
revisit once we've sketched a few real middleware examples.

### 2. Capability-based routing

A route's `needs` clause _is_ its capability declaration, enforced by the
compiler:

```saga
fun list_users : Request -> Response needs {Db, Log}
fun read_pii   : Request -> Response needs {Db, Log, Auth, PII}
```

The second route cannot accidentally skip auth, cannot accidentally log PII,
cannot read from anywhere except what its declared effects allow. The first
route physically cannot access `PII` because it didn't ask for it.

This falls out of Saga's effect system for free — no framework code required.
The framework's contribution is mostly to write docs that frame it this way
and to keep the `dispatch` boundary honest.

## Routes and composition

Saga records cannot carry effect rows, so a `Route` cannot be a record of
`{ method, path, handler }` where `handler` has an open effect row. **Function
types, however, can carry rows.** So routes are functions, composed via
combinators in the style of Giraffe (F#) or Hono.

### The `Skip` effect

A route handler is a function from `Request` to `Response` that may also
signal "I don't match — try the next one." That's an effect:

```saga
effect Skip {
  fun skip : Unit -> a
}
```

A route is therefore a function of type:

```saga
Request -> Response needs {Skip, ..e}
```

where `..e` is whatever the inner application code needs.

### The core combinators

```saga
type Method = GET | POST | PUT | DELETE | PATCH | OPTIONS | HEAD | Custom String

# Attach a method + path matcher to a handler
fun route : Method -> String -> (Request -> Response needs {..e})
         -> Request -> Response needs {Skip, ..e}

# Try each route in order; if one skips, fall through to the next.
fun choose : List (Request -> Response needs {Skip, ..e})
          -> Request -> Response needs {..e}
```

`choose` discharges `Skip` by trying the next route in its list when one
skips. If nothing matches, returns a 404 (or some configurable fallback).

User code:

```saga
let app = choose [
  route GET  "/users"     list_users,
  route POST "/users"     create_user,
  route GET  "/users/:id" show_user,
]
```

`get`/`post`/etc. lowercase sugar functions are a possible future nicety,
but `route METHOD path handler` covers it for v1 and stays out of the way
of users who want a closed `Method` ADT.

### Grouping and mounting

Two flavors of nesting, both useful:

**`group`** — inline grouping with shared effects. Sub-routes share the
parent's effect row; `..e` propagates through.

```saga
fun group : String -> List (Request -> Response needs {Skip, ..e})
         -> Request -> Response needs {Skip, ..e}

let app = choose [
  group "/users" [
    route GET  "/"     list_users,
    route POST "/"     create_user,
    route GET  "/:id"  show_user,
  ],
  group "/posts" [...],
]
```

**`mount`** — mount a fully built sub-app whose effects are already
discharged. Useful when a sub-app needs a different handler stack (admin DB
pool, mock auth in tests), or when the sub-app is its own module/library.

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

Both `group` and `mount` strip the matched prefix before handing the request
down. Inside `group "/users" [...]`, a sub-route's pattern matches against
the remainder, not the full path. This is the Express model and keeps
sub-app authors thinking in relative paths.

The original path is still available via `req.target` — that's the parsed
RFC 9112 form from saga_http and is never modified. We document it as the
escape hatch when handlers need the unstripped path (logging, correlation
IDs).

## Open questions

These are the things we want to pin down before going much further.

### Path parameters

We'll ship **stringly-typed** params in v1: `req.params |> Dict.get "id"`
returns `Maybe String`. Familiar, keeps the router and combinators simple,
and doesn't constrain the future design space.

Params accumulate across nesting — a `group "/users/:id"` followed by
`route GET "/posts"` exposes `id` in `req.params` to the inner handler.

#### Future exploration

Both of these are worth prototyping but should not block v1:

- **Typed combinator DSL** (Servant-ish): `route GET (lit "/users" / int "id") show_user`
  where `show_user : Int -> Request -> Response`. Slick when it works,
  but Saga lacks HLists so the variadic-args plumbing needs design work.
  Also interacts awkwardly with grouping — each `group` would change the
  expected handler arg list.
- **Path-as-effects**: describe the path as a sequence of effect operations
  (`lit!`, `next_int!`) that can be run with two different handlers — one
  to extract the pattern string for registration, another to parse a real
  request. Conceptually elegant; needs care around the "register" handler
  not running the full body.

Users can always pattern-match on `req.path` or segments themselves as an
escape hatch, regardless of which option we ultimately ship.

### JSON

We'll lean on `saga_json` (https://github.com/dylantf/saga_json). At minimum we want:

- `json : Int -> a -> Response where {a: ToJson}` — encode a value as a JSON
  response
- `body_json : Request -> Result a Error where {a: FromJson}` — decode the
  request body

Final shape depends on what `saga_json` exposes.

### The router itself

Likely options:

- Linear list of routes, matched top-to-bottom. Simplest. Fine for small
  apps.
- Trie of path segments. Faster lookup, supports prefix-based grouping
  naturally. Saga's pattern matching makes this pleasant to write.

We can start with the linear list and switch if it becomes a bottleneck or
if we need prefix grouping for middleware purposes.
