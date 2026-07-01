# Routing

Edda's router is a handful of combinators that compose ordinary
functions into an app. Each route is a `Request -> Response` function
that either produces a response or signals "I don't match — try the
next one."

This is part of Edda's core API: `Request`, `Method`, `route`, `choose`,
`group`, `mount`, request lookup helpers, `from_http`, and `to_handler`.

## The shape of a route

A route is just a function:

```saga
fun show_user : Request -> Response
show_user req = text 200 "ok"
```

Wrapped with `route`, it becomes a matcher:

```saga
route GET "/users/:id" show_user
```

`route` returns a function that runs the handler if the method and path pattern
match. If the path does not match, it `skip!`s. If the path matches but the
method does not, it reports the allowed method so `choose` can produce a `405`
if no later route handles the request.

Route paths are matched by segments. Empty segments are ignored, so `/users`
and `/users/` match the same pattern. A plain segment must match exactly, and
a segment starting with `:` captures the request segment into `req.params` and
adds an entry to `req.matches`. `route` only matches when the pattern consumes
the whole current `req.path`.

## `choose`

```saga
fun app : Request -> Response
app req = choose [
  route GET  "/",         home,
  route GET  "/users",    list_users,
  route POST "/users",    create_user,
  route GET  "/users/:id", show_user,
] req
```

`choose` tries each route in order. The first one that doesn't `skip!` wins. If
no path matches, you get the default `not_found` (404). If a path matches but no
route accepts the request method, you get `405 Method Not Allowed` with an
`Allow` header. `OPTIONS` gets an automatic 204 response for matched paths, and
`HEAD` can use `GET` routes with the body stripped.

Routes can declare effects they need. The router lets each route
declare its own set — they don't have to agree:

```saga
choose [
  route GET  "/health",  health,       # pure
  route GET  "/me",      me,           # needs {Auth}
  route GET  "/items/:id", show_item,  # needs {Db, Log}
]
```

`choose` widens the effect row across the list, so the surrounding
context has to discharge the union of everything any route inside it
needs. See [the middleware guide](middleware.md) for how to install
those handlers.

## Path parameters

Segments starting with `:` capture into the request's ordered `params` list:

```saga
route GET "/users/:id/posts/:post_id" show_post

fun show_post : Request -> Response
show_post req = case (param "id" req, param "post_id" req) {
  (Just uid, Just pid) -> text 200 $"user {uid}, post {pid}"
  _ -> text 400 "missing params"
}
```

`param` is `String -> Request -> Maybe String`. Captured values are strings;
convert as needed at the call site. If nested routes reuse a parameter name,
`param` returns the innermost value. Use `req.params` or `req.matches` directly
when ordering or duplicate names matter.

## Request helpers

`Request` stays lean: it stores the raw path/query/header/body facts and route
matching facts, while parsed convenience data is produced by helpers.

```saga
header      "authorization" req
query_param "page"          req
cookie      "session"       req
```

`header` returns the first matching header and `header_values` returns all of
them. `query_params` and `cookies` return ordered `(name, value)` lists, so
duplicate names are preserved. Query values from `query_params` are raw strings;
cookies decode valid percent escapes in names and values.

Use `query_values` when you want URL decoding:

```saga
case query_values req {
  Ok values -> form_get "page" values
  Err _ -> Nothing
}
```

`query_values` returns `FormValues`, the same text-only value container used by
`application/x-www-form-urlencoded` bodies. `multipart_values` returns
`MultipartFormValues`, whose values can be text or buffered files with per-part
headers.

## `group`: inline nesting

`group` matches a path prefix and runs inner routes against the
remainder. The sub-routes share the parent's effect row, so this is
the right tool when you want to factor out a common URL prefix without
changing how handlers are wired.

```saga
choose [
  group "/users" [
    route GET  "/",      list_users,
    route POST "/",      create_user,
    route GET  "/:id",   show_user,
  ],
]
```

If the prefix matches but none of the inner routes match, `group` skips and the
outer `choose` keeps trying later routes. In other words, `group` is prefix
composition; it does not claim the whole prefix unless you add an explicit
catch-all inside it.

Captured params accumulate across nesting, so:

```saga
group "/users/:id" [
  route GET "/posts", list_user_posts,   # can read `id`
]
```

works as expected.

Each successful `group`, `mount`, and `route` appends a `RouteMatch` to
`req.matches`. This is useful for route traces, breadcrumbs, or tooling that
wants to understand which nested patterns handled a request.

## `mount`: attach a sub-app

`mount` attaches an already-pure `Request -> Response` at a prefix.
Use it when a sub-app has its own effect stack that should stay
encapsulated — typically because it lives in its own module and
discharges its own handlers internally.

```saga
mount "/admin" admin_app
mount "/api"   api_app
```

A mounted sub-app sees the path with the prefix stripped. Inside
`admin_app`, requests to `/admin/users` arrive with `req.path =
"/users"`. The unmodified path stays available as `req.original_path`
for things like logging.

Unlike `group`, a mounted app is already a complete `Request -> Response`
application. If it matches the prefix, its own routing and 404 behavior own the
request.

### When to use which

- **`group`** when sub-routes need the parent's effects and should
  stay in the same file. Inner misses fall through to later outer routes.
- **`mount`** when a sub-app is its own thing — different effect
  stack, its own module, its own tests. Real-world example: a public
  API and an admin API that need different DB pools.

## Where the prefix goes

`group` and `mount` both strip the matched prefix from `req.path`
before handing the request down. Sub-app authors think in relative
paths.

| Field               | What it has                                   |
| ------------------- | --------------------------------------------- |
| `req.path`          | The path the current matcher sees (stripped). |
| `req.original_path` | The full path as it came in off the wire.     |

## Fallthrough, `405`, and `OPTIONS`

If nothing in a `choose` matches, the request falls through to
Edda's `not_found` (a 404 with body `"not found"`). To customize, add
a catch-all at the bottom:

```saga
choose [
  route GET "/health", health,
  ...
  fun req -> text 404 $"no route for {show req.method} {req.original_path}",
]
```

A plain lambda is a perfectly valid route — it doesn't have a method
or pattern, so it always matches. The router treats `Request ->
Response` as the most general possible route signature.

If the path matches but only with other methods, Edda returns `405 Method Not
Allowed` and an `Allow` header. `GET` routes imply `HEAD`, and matched paths get
automatic `OPTIONS` responses.
