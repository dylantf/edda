---
title: Edda.Json
---

JSON request/response helpers for Edda.

Built on top of saga_json. The `ToJson` / `FromJson` traits do the work;
this module is just the glue between them and Edda's `Request` / `Response`.

## Types

### BodyError

```saga
type BodyError =
  | NoBody
  | NotUtf8
  | JsonError J.Error
  deriving (Debug)
```

Why decoding a JSON body failed. Keeps body-level problems
(missing body, non-UTF8) distinct from JSON-level ones (malformed,
wrong shape) so the response can say something useful.

## Functions

### json

```saga
fun json : Int -> a -> Response where {a: ToJson}
```

Encode a value as a JSON response body with the given status.
Sets `Content-Type: application/json` automatically.

### body_json

```saga
fun body_json : Request -> Result a BodyError where {a: FromJson}
```

Decode the request body as JSON into a value of type `a`.

The call site needs a type annotation to pick the right `FromJson` impl:
`body_json req : Result User BodyError`

### body_error_response

```saga
fun body_error_response : BodyError -> Response
```

A reasonable default 400 response for `BodyError`. Most apps will
want this directly; apps with structured error envelopes can write
their own mapping.

