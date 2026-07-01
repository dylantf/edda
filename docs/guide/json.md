# JSON

`Edda.Json` is a thin layer over [`saga_json`](https://github.com/dylantf/saga_json).
It gives you a `json` response constructor on one side and a
`body_json` decoder on the other; everything else is the `saga_json`
trait machinery doing the work.

Application records round-trip through HTTP by hand-writing `ToJson` and
`FromJson` impls with `SagaJson.Encode` and `SagaJson.Decode`.

This is part of Edda's core API: `json`, `body_json`, `BodyError`, and
`body_error_response`. These helpers intentionally stay small; application
specific envelopes, auth, and validation policies belong in ordinary routes or
effect handlers.

## Encoding: `json`

```saga
pub fun json : Int -> a -> Response where {a: ToJson}
```

Serializes the value, sets `Content-Type: application/json`, and
returns a `Response` with the given status.

```saga
import Edda.Json (json)
import SagaJson.Encode as Encode
import SagaJson.Encode (ToJson)

record User {
  id: Int,
  name: String,
  email: String,
}

impl ToJson for User {
  to_json u =
    Encode.object [
      ("id", to_json u.id),
      ("name", to_json u.name),
      ("email", to_json u.email),
    ]
}

fun show_user : Request -> Response
show_user _ =
  json 200 (User { id: 1, name: "Alice", email: "alice@example.com" })
```

Lists, `Maybe`, tuples, and the primitive types all have built-in
`ToJson` impls, so:

```saga
json 200 [user1, user2, user3]
```

works without ceremony.

For larger custom shapes, keep composing the primitive builders or reach for
the post-processing combinators — see the
[`saga_json` encoding guide](https://github.com/dylantf/saga_json/blob/main/docs/guide/encoding.md).

## Decoding: `body_json`

```saga
pub type BodyError =
  | NoBody
  | NotUtf8
  | JsonError J.Error

pub fun body_json : Request -> Result a BodyError where {a: FromJson}
```

`BodyError` keeps body-level problems (missing, non-UTF-8) distinct
from JSON-level problems (malformed, wrong shape), so the response
message can say something useful.

Saga needs a type annotation on the call so it can pick the right
`FromJson` impl — `body_json` is generic in `a` and there's nothing
else for inference to latch onto:

```saga
case (body_json req : Result CreateUser BodyError) {
  Err e    -> body_error_response e
  Ok input -> ...
}
```

Write decoders with `SagaJson.Decode`:

```saga
import Std.Fail (Fail)
import SagaJson as J
import SagaJson.Decode as Decode
import SagaJson.Decode (FromJson)

record CreateUser {
  name: String,
  email: String,
}

impl FromJson for CreateUser needs {Fail J.Error} {
  from_json j = CreateUser {
    name: Decode.at "name" Decode.string j,
    email: Decode.at "email" Decode.string j,
  }
}
```

`body_error_response` is a reasonable 400 mapping:

| `BodyError`                              | Response                                                        |
| ---------------------------------------- | --------------------------------------------------------------- |
| `NoBody`                                 | `400 request body is required`                                  |
| `NotUtf8`                                | `400 request body is not valid UTF-8`                           |
| `JsonError (InvalidJson msg)`            | `400 invalid json: <msg>`                                       |
| `JsonError (InvalidShape exp found path)`| `400 invalid shape at /<path>: expected <exp>, found <found>`   |

The `InvalidShape` path is the path-tracking feature of `saga_json` —
nested decoders prepend their field name onto the error path, so the
client sees exactly where the payload went wrong (`/address/zip`,
not just "wrong shape").

If you want a structured error envelope (JSON instead of text), skip
`body_error_response` and write your own mapping from `BodyError` to
`Response`.

## Two error-handling shapes

Routes that decode bodies tend to fall into one of two patterns. Both
work; pick the one that reads better at the call site.

### Inline match

Keep the failure path in the route body. Best when only a handful of
routes parse bodies and each wants its own response shape:

```saga
fun create_user : Request -> Response
create_user req = case (body_json req : Result CreateUser BodyError) {
  Err e    -> body_error_response e
  Ok input -> {
    let u = persist input
    json 201 u
  }
}
```

### Per-app effect

Declare a `Body` effect, have routes declare it in `needs`, and the
boundary handler maps decode failures to a response once. Best when
many routes share the same body type and the same 400 behavior:

```saga
effect Body {
  fun decode_create_user : Unit -> CreateUser
}

fun create_user : Request -> Response needs {Body}
create_user _ = {
  let input = decode_create_user! ()
  let u = persist input
  json 201 u
}

# at the boundary
app req = req |> choose [
  route POST "/users" create_user,
] with {
  decode_create_user () = case (body_json req : Result CreateUser BodyError) {
    Ok v  -> resume v
    Err e -> body_error_response e
  }
}
```

The route reads as happy-path code; the failure path lives in one
place at the boundary. This is just
[an opt-in effect handler](middleware.md#2-opt-in-effect-handlers)
applied to body decoding — there's nothing Edda-specific about it.

## End-to-end example

A small CRUD-ish demo lives in [`src/Demo/JsonApi.saga`](../../src/Demo/JsonApi.saga)
in the repo. It exercises encoding, decoding, both error-handling
shapes, and the path-aware 400 messages. Worth reading once if any of
the above feels abstract.
