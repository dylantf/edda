# Middleware

Edda doesn't have a `Middleware` type. What people call middleware in
other frameworks splits into three patterns here, each one falling out
of ordinary Saga features. The framework provides the primitives; you
compose them.

## The three shapes

| Shape           | Lives at                           | Use when                                                                                   |
| --------------- | ---------------------------------- | ------------------------------------------------------------------------------------------ |
| Wrap function   | `fun outer inner = ...`            | Uniform behavior for everything under it; cross-cutting; no per-route variation.           |
| Opt-in effect   | `with { ... }` at the boundary     | A capability some routes need; varies by environment; you might want to swap it in tests.  |
| Ambient context | `with { ... }` inside the boundary | Request-scoped values pulled by routes that opt in (current user, request ID, JWT claims). |

A real app uses all three: a wrap for logging or panic recovery,
opt-in effects for DB and auth, ambient context for the request ID.

## 1. Wrap functions

A wrap takes a handler and returns a wrapped handler. Same shape going
in as coming out, so they compose by ordinary function application.

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

Reach for a wrap when the behavior should be **uniform** across every
request: logging, timing, gzip, CORS, panic recovery, security
headers, body-size limits.

Wraps can short-circuit (don't call `inner`), modify the response, or
perform any effects of their own. Compose them with normal function
application:

```saga
with_logging (with_panic_recovery app) req
```

For the two most common pure wrapping cases, Edda provides small helpers:

```saga
let app =
  inner
  |> map_request normalize_path
  |> map_response (with_header "X-Frame-Options" "DENY")
```

`map_request` rewrites the `Request` before the inner handler runs.
`map_response` rewrites the `Response` after the inner handler returns.

Edda ships `with_cors` as one of these wraps:

```saga
let cors = { default_cors_config |
  allow_origins: ["https://app.example.com"],
  allow_methods: [GET, POST],
  allow_credentials: True,
  expose_headers: ["X-Request-Id"],
}

with_cors cors app req
```

`with_cors` handles preflight `OPTIONS` requests itself and adds
`Access-Control-*` headers to normal responses. With credentials enabled and a
wildcard origin policy, Edda echoes the request origin instead of emitting `*`.

### Example: panic recovery

```saga
fun with_panic_recovery : (Request -> Response needs {..e})
                       -> Request -> Response needs {..e}
with_panic_recovery inner req = case catch_panic (fun () -> inner req) {
  Ok resp -> resp
  Err msg -> text 500 $"internal error: {msg}"
}
```

Catches both Saga `panic` and native BEAM exceptions, so it's safe to
put at the top of an app.

## 2. Opt-in effect handlers

Routes declare the capabilities they need; the boundary provides the
handlers. Different routes can declare different sets — capability-
based routing falls out of the type system.

```saga
fun me : Request -> Response needs {Auth}
me _ = {
  let user = current_user! ()
  text 200 $"you are {user}"
}
```

The `Auth` effect is just a regular Saga effect:

```saga
effect Auth {
  fun current_user : Unit -> String
}
```

Handlers can short-circuit by not calling `resume`:

```saga
app req = req |> choose [
  route GET "/me" me,
  route GET "/account" account,
] with {
  current_user () = case find_header "authorization" req.headers {
    Just "Bearer alice-token" -> resume "Alice"
    _ -> text 401 "unauthorized"   # no resume → 401
  }
}
```

Reach for opt-in effects when a capability is **route-specific** —
some routes need DB access, some don't; some need auth, some are
public; some emit domain events, some are pure.

### Bracket pattern: code before and after `resume`

When the handler should _do_ something around the route (timing,
tracing, holding a lock), capture `resume`'s result and run code
before and after:

```saga
handler timing for Timing needs {Stdio} {
  measure label = {
    let start = Time.monotonic_ms ()
    let result = resume ()
    let elapsed = Time.monotonic_ms () - start
    println $"[timing] {label}: {elapsed}ms"
    result
  }
}
```

Routes opt in by calling `measure! "label"` somewhere in their body.

### Typed errors via `Fail`

A common opt-in-effect use case is a domain-specific `Fail` effect
whose variants get mapped to HTTP statuses at the boundary — letting
routes read as happy-path code and keeping the status mapping in one
exhaustive handler. See the [error handling guide](error-handling.md)
for the full pattern.

## 3. Ambient context handlers

Same shape as opt-in effects, but the handler is installed _per
request_ inside the user's boundary function, so it can close over the
incoming request:

```saga
fun handle : Http.Request -> Response
handle hr = {
  let req = from_http hr
  let rid = gen_request_id ()
  app req with {
    request_id   () = resume rid,
    current_user () = case find_header "x-user" req.headers {
      Just u -> resume u
      Nothing -> text 401 "unauthorized"
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

Reach for ambient context when the value is **request-scoped** —
current user, request id, parsed JWT claims, feature-flag set,
anything that's "the same for this request, different across
requests."

You can install the context at the root boundary, or at a mounted
sub-app boundary. The session demo uses this to parse the signed
session cookie once for `/session` and provide it through a local
effect:

```saga
effect SessionContext {
  fun current_session_user : Unit -> Maybe String
}

fun secret : Request -> Response needs {SessionContext}
secret _ = case current_session_user! () {
  Just user -> text 200 $"welcome {user}"
  Nothing -> redirect 303 "/session"
}

fun app req =
  choose [
    route GET "/secret" secret,
  ] req with {
    current_session_user () = resume (session_user req)
  }
```

This keeps `Request` lean: the parsed value is available to routes
that opt in, but it is not baked into every request record.

## The smell test

- Should _every_ request get this? → **wrap function.**
- Should the route _declare_ it needs this? → **opt-in effect.**
- Does this value depend on the _request itself_? → **ambient context.**

The three compose freely. A real app might have wraps for
logging/timing/CORS, opt-in effects for DB and auth and a typed
`Fail`, and ambient context for the request ID and current user.
