# Getting started

Edda is an effect-based web framework for Saga. It sits on top of
[`saga_http`](https://github.com/dylantf/saga_http) and gives you a
router, request/response wrappers, and a few conventions for installing
effects on your routes.

> **Status: very work-in-progress.** Edda is exploratory. The API will
> change, probably radically.

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

import Std.Actor (beam_actor)
import Std.IO (console, println)
import Edda (Request, choose, route, GET, to_handler)
import SagaHttp.Http (serve, await_shutdown, default_config, text, Response)

fun home : Request -> Response
home _ = text 200 "hello from edda"

fun app : Request -> Response
app req = choose [
  route GET "/" home,
] req

main () = {
  let cfg = { default_config | port: 8080 }

  case serve cfg (to_handler app) {
    Err e -> dbg ("startup failed", e)
    Ok h -> {
      println $"server at http://localhost:{cfg.port}"
      await_shutdown h
    }
  } with {beam_actor, console}
}
```

Three things worth noticing:

- **`app : Request -> Response`** — pure. No `needs` row. All effects
  in your routes have to be discharged somewhere before control returns
  to `serve`. Inside `app` you're free to install effect handlers at
  any level you like.
- **`to_handler app`** — adapts Edda's `Request -> Response` to the
  shape `serve` expects (which takes a `saga_http` request).
- **`choose [...]`** — the router. Each route is a function; `choose`
  tries them top to bottom.

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
