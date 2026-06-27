---
title: Edda
---

## Types

### Method

```saga
type Method =
  | GET
  | POST
  | PUT
  | DELETE
  | PATCH
  | OPTIONS
  | HEAD
  | Custom String
```

Standard HTTP methods, with an escape hatch for non-standard verbs.

### RouteMatch

```saga
record RouteMatch {
  pattern: String,
  path: String,
  params: List (String, String)
}
```

A route, group, or mount segment that matched while routing the request.

### RequestBodyError

```saga
type RequestBodyError =
  | MissingBody
  | BodyNotUtf8
```

Errors from raw request body helpers.

### ContentTypeError

```saga
type ContentTypeError =
  | MissingContentType
  | UnexpectedContentType String
```

Errors from content-type checks.

### SameSite

```saga
type SameSite =
  | SameSiteLax
  | SameSiteStrict
  | SameSiteNone
```

SameSite values for outgoing cookies.

### CookieOptions

```saga
record CookieOptions {
  path: Maybe String,
  domain: Maybe String,
  max_age: Maybe Int,
  http_only: Bool,
  secure: Bool,
  same_site: Maybe SameSite
}
```

Options for outgoing `Set-Cookie` headers. Names and values are emitted as
provided; no validation or encoding is performed yet.

### CorsConfig

```saga
record CorsConfig {
  allow_origins: List String,
  allow_methods: List Method,
  allow_headers: List String,
  allow_credentials: Bool,
  max_age: Maybe Int
}
```

CORS policy for `with_cors`. Use `"*"` in `allow_origins` for wildcard origin.

### Request

```saga
record Request {
  method: Method,
  path: String,
  original_path: String,
  query: Maybe String,
  matches: List RouteMatch,
  params: List (String, String),
  headers: List (String, String),
  body: Maybe BitString
}
```

The framework's view of an HTTP request.

`path` reflects the current matcher's view — inside a `group`, it has the
matched prefix stripped. `original_path` is always the unmodified path
from the wire, useful for logging and correlation. `query` is the raw
origin-form query string when present.

`matches` records the nested route/group/mount patterns that matched.
`params` is ordered, so duplicate names from nested matches are preserved.

## Effects

### Skip

```saga
effect Skip {
  fun skip : Unit -> a
}
```

## Values

### not_found

```saga
fun not_found : Response
```

### no_content

```saga
fun no_content : Response
```

A 204 response with an empty body.

### default_cookie_options

```saga
fun default_cookie_options : CookieOptions
```

Default cookie options: no attributes.

### default_cors_config

```saga
fun default_cors_config : CorsConfig
```

A permissive CORS config for local/simple APIs.

## Functions

### method_str

```saga
fun method_str : Method -> String
```

### param

```saga
fun param : String -> Request -> Maybe String
```

Look up a captured path parameter by name.

If nested routes reuse a parameter name, the innermost capture wins.

### header

```saga
fun header : String -> Request -> Maybe String
```

Look up the first request header by name, case-insensitively.

### header_values

```saga
fun header_values : String -> Request -> List String
```

Look up every request header value for `name`, case-insensitively.

### query_params

```saga
fun query_params : Request -> List (String, String)
```

Parse the raw query string into ordered `(name, value)` pairs.
Duplicate keys are preserved. Values are not percent-decoded yet.

### query_param

```saga
fun query_param : String -> Request -> Maybe String
```

Look up the first query parameter value by name.

### cookies

```saga
fun cookies : Request -> List (String, String)
```

Parse Cookie headers into ordered `(name, value)` pairs.
Duplicate names are preserved. Values are not percent-decoded yet.

### cookie

```saga
fun cookie : String -> Request -> Maybe String
```

Look up the first cookie value by name.

### body_bytes

```saga
fun body_bytes : Request -> Result BitString RequestBodyError
```

Return the buffered request body bytes, or `MissingBody`.

### body_text

```saga
fun body_text : Request -> Result String RequestBodyError
```

Decode the buffered request body as UTF-8 text.

### require_content_type

```saga
fun require_content_type : String -> Request -> Result Unit ContentTypeError
```

Require the request Content-Type to match `expected`. Parameters such as
`; charset=utf-8` are allowed.

### html

```saga
fun html : Int -> String -> Response
```

Build an HTML response with `Content-Type: text/html; charset=utf-8`.

### redirect

```saga
fun redirect : Int -> String -> Response
```

Build a redirect response with the given status and `Location`.

### with_header

```saga
fun with_header : String -> String -> Response -> Response
```

Append a response header without removing existing values.

### content_type

```saga
fun content_type : String -> Response -> Response
```

Set or replace the response Content-Type header.

### octet_stream

```saga
fun octet_stream : Int -> BitString -> Response
```

Build a binary response with `application/octet-stream`.

### set_cookie

```saga
fun set_cookie : String -> String -> Response -> Response
```

Add a `Set-Cookie` response header with default options.

### set_cookie_with

```saga
fun set_cookie_with : String -> String -> CookieOptions -> Response -> Response
```

Add a `Set-Cookie` response header with explicit options.

### delete_cookie

```saga
fun delete_cookie : String -> Response -> Response
```

Expire a cookie with default options.

### delete_cookie_with

```saga
fun delete_cookie_with : String -> CookieOptions -> Response -> Response
```

Expire a cookie with explicit options. Use matching `path`/`domain` options
when deleting scoped cookies.

### with_cors

```saga
fun with_cors : CorsConfig -> (Request -> Response needs {..e}) -> Request -> Response needs {..e}
```

Wrap an app with CORS response headers and preflight handling.

### route

```saga
fun route : Method -> String -> (Request -> Response needs {..e}) -> Request -> Response needs {Skip, ..e}
```

Build a route. If method matches and the pattern fully consumes the path
(capturing any `:name` segments into params), the handler runs; otherwise
the route skips to let the next route try.

### choose

```saga
fun choose : List (Request -> Response needs {Skip, ..e}) -> Request -> Response needs {..e}
```

Try each route in order. If every route skips, return `not_found`.

### group

```saga
fun group : String -> List (Request -> Response needs {Skip, ..e}) -> Request -> Response needs {Skip, ..e}
```

Match a path prefix and run the inner routes against the remainder.
Captured `:name` params accumulate and are visible to sub-routes.

If no inner route matches, the group skips so the outer `choose` can continue.

### mount

```saga
fun mount : String -> (Request -> Response) -> Request -> Response needs {Skip}
```

Mount a sub-app whose effects are already handled. Captured `:name`
params accumulate and are visible to the sub-app.

### from_http

```saga
fun from_http : Http.Request -> Request
```

Lift a SagaHttp.Http.Request into an Edda.Request.

### to_handler

```saga
fun to_handler : (Request -> Response) -> Http.Request -> Response
```

Adapt an Edda app to the `Request -> Response` shape `SagaHttp.serve`
expects. Use as: `serve config (to_handler app)`.
