# Saga compiler bug: indirect call to effectful function value misses hidden args

## Summary

When an effectful function value (one whose type carries a `needs` row) is
pulled out of a list via pattern match and called from inside another
effectful function, the call site emits a plain 1-arg call instead of a
CPS-aware call. At runtime the BEAM closure expects more arguments (CPS
continuation / evidence) than were passed, and the process crashes with a
`badarity` / "function called with 1 argument(s), but expects 3" error.

Type-level tracking is correct — the program compiles cleanly and the row
types unify the way the source suggests. The divergence is in codegen at
the call site `r input` inside the dispatch function.

## Minimal reproducer (no HTTP, no library deps)

```saga
module Main

import Std.IO (console, println)

effect Skip {
  fun skip : Unit -> a
}

fun route : String -> (String -> String needs {..e})
         -> String -> String needs {Skip, ..e}
route pattern h input =
  if input == pattern then h input
  else skip! ()

fun choose : List (String -> String needs {Skip, ..e})
          -> String -> String needs {..e}
choose routes input = case routes {
  [] -> "no match"
  r :: rest -> r input with {
    skip () = choose rest input
  }
}

fun greet : String -> String
greet _ = "hello!"

fun bye : String -> String
bye _ = "goodbye!"

fun r1 : String -> String needs {Skip}
r1 input = route "/" greet input

fun r2 : String -> String needs {Skip}
r2 input = route "/bye" bye input

main () = {
  let result = choose [r1, r2] "/bye"
  println result
} with console
```

```
$ saga run
Runtime error: function called with 1 argument(s), but expects 3

  Stack trace:
    main:choose/2
```

Expected output: `goodbye!`

## What works vs. what doesn't

This matrix was built by progressively peeling layers off the failing case.

| Form | Result |
| --- | --- |
| `route p h x with {...}` — direct call, all args, inline handler | ✅ works |
| `r x with {...}` where `r` is a named effectful function | ✅ works |
| `call_indirect f x` with `f` an effectful function passed as parameter, called as `f x with {...}` | ✅ works |
| `choose [r1, r2] "/bye"` — named effectful funs in a list, pulled via `r :: rest` | ❌ badarity (1 vs 3) |
| `choose [route p h] x` — partial application in list | ❌ badarity (1 vs 3) |
| `choose [fun x -> route p h x] x` — lambdas in list | ❌ different error: `evidence_tag_not_found 'Main.Skip'` |

### Important: it is **not** the simple "indirect call" case

```saga
fun call_indirect : (String -> String needs {Skip}) -> String -> String
call_indirect f input = f input with { skip () = "default" }

main () = {
  println (call_indirect maybe_skip "nope")
} with console
```

This works fine. So a *single* effectful function passed through a single
parameter and called is OK. The bug needs the list-via-pattern-match shape
in `choose`.

## Hypothesis

Effectful functions are CPS-rewritten with extra hidden parameters (per
the FFI docs). At a direct call site to a named effectful function, the
compiler emits a CPS-style call that passes these hidden args. When the
function comes from a *list element pulled by pattern match*, the codegen
treats the call as a plain function value invocation and omits the hidden
args.

Working theory: there are two codegen paths for "call a function" —
one for direct/named calls (CPS-aware) and one for "fn value in a
variable" (plain). The pattern-bound name `r :: rest` ends up in the
second path even though its type has a `needs` row.

The lambda-in-list variant fails differently (`evidence_tag_not_found`)
which suggests the lambda captures evidence at creation time rather than
at call time. Different bug, possibly related.

## Suggested next steps

1. **Look at the generated Core Erlang for `choose`.** Specifically the
   call site `r input` inside the `case` arm. Compare to a direct call
   site like `r1 input` (which works).
2. **Check the call-site codegen branch** that handles applying a variable
   of function type. Does it inspect the type's effect row and emit a
   CPS call when the row is non-empty?
3. **Confirm the BEAM arity** of the closure produced by `r1` (which has
   `needs {Skip}` in its signature). The "expects 3" in the error implies
   1 visible + 2 hidden args.
4. The lambda-in-list variant (`evidence_tag_not_found`) is a useful
   second data point — same indirection shape, different error. Likely
   the same root cause expressed through a different codegen path.

## Why this matters

This blocks a route-combinator pattern that's central to the Edda web
framework design: build a `List (Request -> Response needs {Skip, ..e})`,
dispatch by `choose` over that list. Every alternative we've sketched
runs into one of the variants above.
