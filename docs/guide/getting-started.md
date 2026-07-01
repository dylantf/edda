# Getting started

Edda is an effect-based web framework for Saga. It sits on top of
[`saga_http`](https://github.com/dylantf/saga_http) and gives you a
router, request/response wrappers, and a few conventions for installing
effects on your routes.

> **Status: early but usable.** The core router and `Edda.Json` APIs are the
> first stable slice. `Edda.Spec` is an experimental sidecar for type-driven
> route contracts and OpenAPI generation, not the primary routing API.

## What it gives you

- A router (`route`, `choose`, `group`, `mount`) with capturing path
  parameters (`/users/:id`).
- A pure `Request -> Response` boundary into `saga_http`, so your app
  decides where effects are discharged.
- Three composable middleware shapes — wrap functions, opt-in effect
  handlers, and ambient context handlers — all of which fall out of
  ordinary Saga features rather than a `Middleware` type.
- A small `Edda.Json` module that bridges to
  [`saga_json`](https://github.com/dylantf/saga_json) for typed
  encoding and decoding.

It is _not_ opinionated about databases, ORMs, templating, logging
formats, or app structure.

## How it works

The thing that makes Edda interesting isn't the router. It's that
almost everything you'd recognize as "framework features" in Express
or Phoenix or Rails falls out of Saga's effect system rather than
being invented by the framework.

A handler is a function `Request -> Response needs {..e}`. The `..e`
is whatever effects the handler needs. The router widens those rows
across a list of routes; the boundary discharges them by installing
handlers. That's almost the whole story.

Here is the same story told three ways — feature, language primitive,
what Edda has to add:

| You want                                                           | Saga gives you                                                                                           | Edda adds                                                                                                                              |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Middleware                                                         | Effect handlers around a computation                                                                     | Nothing — there's no `Middleware` type. Three patterns ([wraps, opt-in, ambient](middleware.md)) fall out of how you install handlers. |
| Request-scoped values (current user, request ID, JWT claims)       | A handler installed _per request_ that closes over the request                                           | A convention: write your own `from_http` boundary instead of `to_handler`. The handler is just a normal handler.                       |
| Typed errors that map to HTTP statuses                             | A `Fail` effect plus an exhaustive handler at the boundary                                               | Nothing. Use the fail effect with an abort-early handler and let errors bubble up. The compiler ensures every variant gets mapped.     |
| Scoped resources (open in route, auto-close at end)                | The `Scope` effect + `acquire_scoped! acquire release` pattern from the stdlib                           | Nothing. Install `run_scoped` once at the boundary; routes acquire whatever they want.                                                 |
| Resource cleanup that survives errors and panics                   | A handler arm with a `finally { ... }` block — guaranteed to run on normal completion, abort, _or_ panic | Nothing. A scoped DB-pool handler with Just Works inside a route, with no framework support.                                           |
| Background work per request (fire-and-forget, supervised children) | `beam_actor` + Saga's process/supervision effects                                                        | Nothing. The route spawns; supervision lives in the same handler stack.                                                                |
| Capability-based access control                                    | The `needs {..}` row on each route, checked at compile time                                              | A router that lets routes with different rows live in one `choose` list.                                                               |

The flexibility and compositional powers of effects mean the framework needs almost
nothing on its own -- the surface area you have to learn is mostly the
_language_, not the _framework_. New things you build using effects
compose with everything else without ceremony.

A more concrete example. Suppose you want every route in a sub-app to
get a database connection that's released when the route ends,
_including_ if the route panics mid-handler. Routes that need the
connection declare it as an effect; the boundary opens one connection
per request and serves it up.

```saga
effect Db {
  fun db : Unit -> Connection
}

fun list_items : Request -> Response needs {Db}
list_items _ = {
  let conn = db! ()
  ...
}

fun sub_app : Request -> Response
sub_app req = {
  let conn = acquire_scoped! open_db close_db

  choose [
    route GET "/items"     list_items,
    route GET "/items/:id" show_item,
  ] req
} with {
  run_scoped,
  db () = resume conn
}
```

Four moving parts, each from a different place:

- **`acquire_scoped!`** comes from `Std.Scope`. It opens the
  connection and registers `close_db` to run when the scope ends.
- **`run_scoped`** comes from the stdlib too. Its `finally` block
  fires `close_db` no matter how the route exits — normal return,
  `fail!`, panic, anything.
- **`effect Db` and the `with { db () = resume conn }` handler** are
  ordinary Saga. The route asks for a connection via `db! ()`; the
  handler supplies the one we just opened. No need to thread the connection manually through arguments.
- **`choose`** is the only Edda piece. The type system makes sure
  routes can't skip the cleanup, because every effect they declare
  has to be discharged somewhere up the stack.

The result is a framework whose source code is small, whose surface
area is small, but powerful.

## Install

Add Edda to your `project.toml`:

```toml
[deps]
edda = { git = "https://github.com/dylantf/edda" }
saga_http = { git = "https://github.com/dylantf/saga_http" }
saga_json = { git = "https://github.com/dylantf/saga_json" }
```

`saga_http` is the underlying server; `saga_json` is only needed if
you use `Edda.Json`. Both are workspace siblings during the prototype
phase.

## Hello world

```saga
module Main

import Std.IO (console, println)
import Edda (Request, choose, route, GET, create_app, use_routes, use_port, start, wait)
import SagaHttp.Http (text, Response)

fun home : Request -> Response
home _ = text 200 "hello from edda"

fun router : Request -> Response
router req = choose [
  route GET "/" home,
] req

main () = {
  let port = 8080
  let app =
    create_app
      |> use_routes router
      |> use_port port
  case start app {
    Err e -> dbg ("startup failed", e)
    Ok running -> {
      println $"Server running at http://localhost:{port}"
      wait running
    }
  }
} with console
```

Three things worth noticing:

- **`router : Request -> Response`** — this small router is pure. Apps that use
  decoded form/query/multipart helpers may need `RequestParserConfig`; the
  root `serve` helper supplies that for you.
- **`create_app |> use_routes router |> use_port port`** — builds the app root
  with defaults, then overrides the pieces this program cares about.
- **`start app`** — starts the server and returns after the socket is bound.
  The `Ok` branch is the natural place for startup logs or metrics.
- **`wait running`** — keeps the program alive until the server shuts down.
- **`serve app`** — starts the server, installs Edda's default parser policy,
  provides Saga's BEAM actor handler, and blocks until shutdown. Server events
  are discarded only if no handler was configured with `use_server_events`.
- **`choose [...]`** — the router. Each route is a function; `choose`
  tries them top to bottom.

## Demos

Run the example app with:

```sh
saga run
```

The demo server listens on `http://localhost:8080`.

- `/form` renders and submits an `application/x-www-form-urlencoded` form. The
  POST route decodes a `FormValues` value and preserves duplicate field names.
- `/form/multipart` renders and submits a buffered multipart form with text and
  file values.
- `/session` demonstrates a tiny cookie-backed login/logout flow.
- `/session/secret` is a protected route that redirects back to `/session`
  unless the demo session cookie is present.
- `/static/cats/cat1.jpg` is served by the static-directory helper.

## What's next

- [Routing](routing.md) — the full router API.
- [Middleware](middleware.md) — the three shapes and when to reach for
  which.
- [Error handling](error-handling.md) — typed `Fail` for early exit,
  plus `with_panic_recovery` for the unexpected.
- [JSON](json.md) — `Edda.Json` for typed request and response bodies.
- [Runtime model](runtime-model.md) — one BEAM process per request,
  what falls out of it, and the supervision / cleanup patterns it
  enables.
