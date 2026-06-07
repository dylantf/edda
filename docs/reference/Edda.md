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

### Request

```saga
record Request {
  method: Method,
  path: String,
  original_path: String,
  params: Dict String String,
  headers: List (String, String),
  body: Maybe BitString
}
```

The framework's view of an HTTP request.

`path` reflects the current matcher's view — inside a `group`, it has the
matched prefix stripped. `original_path` is always the unmodified path
from the wire, useful for logging and correlation.

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

### route

```saga
fun route : Method -> String -> Request -> Response -> Request -> Response needs {Skip, ..e}
```

Build a route. If method matches and the pattern fully consumes the path
(capturing any `:name` segments into params), the handler runs; otherwise
the route skips to let the next route try.

### choose

```saga
fun choose : List (Request -> Response needs {Skip, ..e}) -> Request -> Response
```

### group

```saga
fun group : String -> List (Request -> Response needs {Skip, ..e}) -> Request -> Response needs {Skip, ..e}
```

Match a path prefix and run the inner routes against the remainder.
Captured `:name` params accumulate and are visible to sub-routes.

### mount

```saga
fun mount : String -> Request -> Response -> Request -> Response needs {Skip}
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
fun to_handler : Request -> Response -> Http.Request -> Response
```

Adapt an Edda app to the `Request -> Response` shape `SagaHttp.serve`
expects. Use as: `serve config (to_handler app)`.
