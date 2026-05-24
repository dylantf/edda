# Error handling

Most real apps end up wanting two distinct things:

1. **Expected errors** — "this id doesn't exist", "the body is
   missing a field", "this resource is in conflict." Each one maps
   cleanly to an HTTP status. You want routes to read as happy-path
   code and the status mapping to live in one place.
2. **Unexpected errors** — bugs, divide-by-zero, a library throwing
   something you didn't anticipate. You want them caught and turned
   into a 500 rather than crashing the request handler.

Edda doesn't add machinery for either. Both fall out of effects: a
typed `Fail` effect for the first, a `with_panic_recovery` wrap
around the app for the second.

## Typed `Fail` for expected errors

Define a domain error type with a variant per failure mode, then a
`Fail` effect that takes one:

```saga
type AppError =
  | NotFound String
  | BadRequest String
  | Conflict String
  | Internal String

effect Fail {
  fun fail : AppError -> a
}
```

The polymorphic return type `a` is what makes `fail!` flexible — not
a guarantee that it aborts. `fail!` is just an effect call carrying an
`AppError`; what happens when you call it is *entirely* up to whoever
installed the `Fail` handler:

- A handler that **doesn't resume** turns `fail!` into an early
  exit. The route never sees control again, and whatever the handler
  returns becomes the result of the handled computation. This is the
  shape the boundary uses below.
- A handler that resumes with some value lets the route continue.
  This is how you'd implement "treat NotFound as Just Nothing" or
  similar recovery.
- A handler that catches the failure and returns `Result a AppError`
  lets the caller pattern-match inline rather than abort.

The same `fail!` site participates in all three. The choice lives in
the handler, not in the call. That's the thing effects give you that
exceptions can't: error propagation as a policy, not a control-flow
primitive baked into the call.

### Routes read as happy-path code

Routes declare `needs {Fail}` and call `fail!` wherever they hit a
non-success case. Given a boundary handler that aborts (below), the
rest of the function reads as if errors don't exist:

```saga
fun show_item : Request -> Response needs {Fail}
show_item req = case param "id" req {
  Nothing -> fail! (BadRequest "missing id")
  Just id -> case lookup id {
    Ok value      -> text 200 value
    Err DbMissing -> fail! (NotFound $"item {id}")
    Err DbTimeout -> fail! (Internal "database timeout")
  }
}
```

No `Result` plumbing, no `?` operator, no try/catch around every call
— *because* the handler we install below chooses to abort on each
`fail!`. With a different handler, the same `fail!` calls would
behave differently without the route changing.

### The boundary maps variants to statuses

A single handler at the boundary catches every `AppError` variant and
turns it into a response. This handler *doesn't call `resume`*, so
each `fail!` becomes an early exit and the response becomes the
result of the handled `choose`:

```saga
app req = choose [
  route GET "/items/:id" show_item,
  route GET "/conflict"  always_conflict,
  ...
] req with {
  fail (NotFound msg)   = text 404 msg
  fail (BadRequest msg) = text 400 msg
  fail (Conflict msg)   = text 409 msg
  fail (Internal msg)   = text 500 msg
}
```

Two things to notice:

- **One mapping, one place.** Every route's failures flow through
  these four arms. Adding a new route doesn't change the boundary;
  changing a status code doesn't require touching the routes.
- **Compiler-checked exhaustiveness.** Add a new variant to
  `AppError` and the pattern match here becomes non-exhaustive — the
  compiler tells you exactly where to handle it. You cannot
  accidentally drop an error variant on the floor.

This is the same shape as any other [opt-in effect
handler](middleware.md#2-opt-in-effect-handlers); the typed-error use
case just happens to be common enough to deserve its own page.

## Lifting `Result` into `Fail`

A lot of Saga code returns `Result a Error` — `Std.Dict.get`,
database calls, parsers. Pattern-match once and `fail!` on the error
branch:

```saga
fun show_item : Request -> Response needs {Fail}
show_item req = case fake_lookup id {
  Ok value      -> text 200 value
  Err DbMissing -> fail! (NotFound $"item {id}")
  Err DbTimeout -> fail! (Internal "database timeout")
}
```

This is the "translation layer" where one error vocabulary (`DbError`)
becomes another (`AppError`). Doing it inside the route keeps the
boundary handler ignorant of every dependency's error type — it only
knows about `AppError`.

If the lift gets repetitive, factor it into a helper:

```saga
fun lift_db : Result a DbError -> a needs {Fail}
lift_db r = case r {
  Ok x          -> x
  Err DbMissing -> fail! (NotFound "missing")
  Err DbTimeout -> fail! (Internal "database timeout")
}
```

Then the route shrinks:

```saga
fun show_item : Request -> Response needs {Fail}
show_item req = text 200 (lift_db (fake_lookup id))
```

The helper itself declares `needs {Fail}`, so calling it propagates
the requirement up to the route automatically.

The direction also goes the other way — `Fail` can be *lowered* back
into a `Result`. See the next section.

## Lowering `Fail` into `Result`

Mirror image of the section above. The boundary handler we'll use in
the [next section](#the-boundary-maps-variants-to-statuses) makes
`fail!` look like an early exit because *that handler* doesn't call
`resume`. A different handler can catch the failure and return it as
a value instead.

The standard library ships a generic `to_result` handler that catches
any `Fail` and returns the computation's result as
`Result a AppError`:

```saga
import Std.Fail (to_result)

fun show_item_safe : Request -> Response
show_item_safe req = case (show_item req with to_result) {
  Ok resp -> resp
  Err (NotFound msg) -> {
    log_audit "miss" req
    text 404 msg
  }
  Err other -> text 500 (debug other)
}
```

The route's `fail!` sites haven't changed. `show_item` still has
`needs {Fail}`. But because we installed `to_result` instead of an
aborting handler, the call returns `Result Response AppError` and the
caller decides what to do with each variant — log specially, fall
back, recover, whatever. The same machinery powers retry handlers,
"swallow this error class only," and inline pattern-matching on
recoverable failures.

So the two sections are duals:

| Direction                     | What you write                                    | When                                            |
| ----------------------------- | ------------------------------------------------- | ----------------------------------------------- |
| `Result a E` → `a needs {Fail}` | Pattern-match, `fail!` on `Err` (`lift_db` above) | Translating a dependency's error vocabulary into yours |
| `a needs {Fail}` → `Result a E` | Install `to_result` as the handler                | Caller wants to inspect/recover inline rather than abort |

The lesson: don't think of `fail!` as "early return." Think of it as
"emit a value of type `AppError` into whoever is listening." The
listener decides if that emission is fatal — and if it isn't, the
same emission shows up downstream as ordinary data.

## `with_panic_recovery` for the unexpected

The `Fail` effect handles errors you *anticipated*. For everything
else — bugs, native exceptions, divide-by-zero — wrap the app in a
panic recovery handler:

```saga
fun with_panic_recovery : (Request -> Response needs {..e})
                       -> Request -> Response needs {..e}
with_panic_recovery inner req = case catch_panic (fun () -> inner req) {
  Ok resp -> resp
  Err msg -> text 500 $"internal error: {msg}"
}
```

`catch_panic` is row-polymorphic, so the wrap composes with anything
inside it. Apply it at the top of an app:

```saga
pub fun app : Request -> Response
app req = {
  with_panic_recovery (choose [
    route GET "/items/:id" show_item,
    route GET "/oops"      oops,        # panic "intentional bug"
    route GET "/divide"    divide,      # 10 / 0
  ]) req with {
    fail (NotFound msg)   = text 404 msg
    fail (BadRequest msg) = text 400 msg
    ...
  }
}
```

Both Saga panics (`panic "..."`) and BEAM native exceptions
(arithmetic errors, badmatch, etc.) come back as `Err msg`. The route
never sees them; the client gets a 500 with a message instead of a
dropped connection.

## A useful split

Together, the two patterns give you a clean two-tier story:

| Kind of failure                | Mechanism                                      | Response shape                            |
| ------------------------------ | ---------------------------------------------- | ----------------------------------------- |
| Expected (typed domain errors) | `fail!` on a variant, mapped at the boundary   | Whatever status the variant maps to (400, 404, 409, ...) |
| Unexpected (bugs, exceptions)  | `with_panic_recovery` wrap around the app      | 500 with a generic message                |

The first is for cases you *want* the type system to make you handle.
The second is for cases the type system can't see.

## End-to-end example

The pattern lives in [`src/Demo/ErrorMiddleware.saga`](../../src/Demo/ErrorMiddleware.saga)
in the repo. It has both layers wired up against a small set of
routes that exercise every variant, plus a `/oops` route that
explicitly panics so you can see the recovery wrap kick in.
