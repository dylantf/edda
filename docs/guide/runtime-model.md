# Runtime model

Underneath, an Edda app is a tree of BEAM processes. `saga_http`'s
`serve` runs one process accepting connections on the listening
socket and spawns a _new_ process for every accepted connection —
that's where your `app req` runs, where the route runs, where any
`with` handlers around the route get installed. When the request
finishes (or the connection closes, or the handler panics), that
process exits. The next request gets a fresh one.

This is how `saga_http` works, which in turn is how the BEAM has worked since the late 1980s.

## What falls out

Three consequences of one-process-per-request that matter for how you
think about an Edda app:

**Requests are isolated.** No shared mutable state between concurrent
handlers. A panic in one handler kills its own process and nothing
else — the accept loop keeps running, every other in-flight request
keeps running. There is no "global crash" failure mode in this
architecture; the only way one bad request can take down another is
if you wire them together through shared resources you control.

**Concurrency is cheap.** BEAM processes are not OS threads. They
start in microseconds, occupy a couple of KB of memory each, and
scale to hundreds of thousands per node on commodity hardware.
`saga_http` spawns one per request without ceremony because the
platform was built for that exact pattern. You don't need a worker
thread pool, an event loop, or `async`/`await` colouring on your
functions — the runtime multiplexes processes across CPU cores for
you.

**Per-request state is automatic.** Anything you bind inside `app
req` — a request ID, a DB transaction handle, a logger with the
trace ID baked in — is local to that request's process. It cannot
leak into another request. This is why [ambient context
handlers](middleware.md#3-ambient-context-handlers) work the way they
do: the handler closes over the request because the handler _is_
running in a process dedicated to that request.

The rest of this page is patterns that build on these three.
Supervision can restart a failed handler because the failed handler
was its own process. Background spawns survive past the response
because they're independent processes from the start. Resource
cleanup with `finally` is bounded by the request process's lifetime.
None of it requires Edda — it falls out of having request handlers
running on the BEAM.

## The actor effects, briefly

Saga exposes the BEAM's process machinery as ordinary effects
(`Process`, `Actor`, `Monitor`, `Timer`). The standard library
provides handlers (`beam_actor`, `Std.Supervisor`) that map those
effects onto real BEAM processes and supervision. Edda doesn't wrap
any of it — because it's effects and handlers, it composes with
routes the same way every other Saga pattern does.

The hello-world `main` already discharges the actor effects that
`serve` needs:

```saga
main () = {
  let cfg = { default_config | port: 8080 }
  case serve cfg (to_handler app) {
    Err e -> dbg ("startup failed", e)
    Ok h  -> await_shutdown h
  }
} with {beam_actor, console}
```

`beam_actor` is the handler that connects the abstract effects to the
concrete runtime. The server, the accept loop, and each request
handler are all real BEAM processes underneath. The effect row
disappears at this boundary; everything inside `serve` runs against
real processes.

## Panics in routes

A route that panics doesn't take the server with it: the per-request
process dies and the client gets a dropped connection. That's
sometimes what you want, but usually not — clients see a connection
error instead of a status code.

For a clean 500 instead, wrap the app with `with_panic_recovery`. See
[error handling](error-handling.md#with_panic_recovery-for-the-unexpected)
for the pattern. `catch_panic` handles both Saga `panic`s and native
BEAM exceptions (`badarith`, `badmatch`, ...), so a single wrap covers
both.

## Supervised retries inside a route

Some operations are flaky for boring reasons — a transient network
hiccup, a database deadlock, a rate-limit blip. `Std.Supervisor`'s
`supervised` retries a computation a fixed number of times:

```saga
import Std.Supervisor (supervised)

fun show_item : Request -> Response needs {Fail}
show_item req = case supervised 3 (fun () -> fetch_from_upstream id) {
  Ok value  -> json 200 value
  Err _why  -> fail! (Internal "upstream unreachable")
}
```

`supervised` catches `fail!` from the inner computation and retries
up to N times before giving up. For exponential backoff between
retries, the saga-website [supervision
guide](https://saga.run/guide/supervision) has a `supervise_backoff`
recipe that adds a `Timer` effect and a doubling delay.

The route's own `Fail` row stays intact — the inner supervisor only
catches the _inner_ computation's failure, not the route's.

## Background work per request

Fire-and-forget work after the response goes out — sending an audit
event, enqueueing an email, warming a cache — is a `spawn!` away:

```saga
fun create_user : Request -> Response needs {Process}
create_user req = {
  let u = persist input
  spawn! (fun () -> {
    send_welcome_email u
    log_audit "user.created" u.id
  })
  json 201 u
}
```

The spawned process is independent of the request handler. The
response goes out as soon as `json 201 u` returns; the email and
audit log happen on their own time. If the child crashes, the request
is unaffected.

If you want the child to be supervised (retried on crash), spawn into
`supervise`:

```saga
spawn! (fun () -> supervise (fun () -> send_welcome_email u))
```

The supervisor lives inside the child process, so a crash and retry
happens locally without involving the request handler at all.

## Resource cleanup across restarts

When a supervised computation acquires resources, pair `supervise`
with `run_scoped` so each retry gets a fresh resource and the old one
is released:

```saga
fun process_message : Message -> Unit needs {Fail, Scope}
process_message msg = {
  let db = acquire_scoped! (fun () -> connect db_url) disconnect
  do_work db msg
}

# spawned background worker
spawn! (fun () -> {
  supervise (fun () -> process_message msg)
} with run_scoped)
```

If `do_work` calls `fail!`, the supervisor catches it, the `finally`
block in `run_scoped` closes the old DB connection, and the next
attempt acquires a fresh one. The cleanup story is the same whether
the failure is a `fail!`, an abort from another handler, or a panic
— `finally` runs in all three cases.

This is the BEAM "let it crash" story translated to Saga: write the
happy path, let supervisors handle restarts, let `finally` handle
cleanup. Each piece is its own small handler; they compose because
they share the same mechanism.

## What Edda contributes

Nothing in particular. The router is a pure function; the boundary
helper is a pure function; the request handlers are pure functions
(modulo their `needs` row). Everything in this page is the language
and standard library doing the work — Edda just gives the request
handler somewhere natural to live.

The framework-level question to keep an eye on as Edda evolves: what
patterns get repeated enough across apps that they're worth promoting
to library helpers? Per-request panic recovery is an obvious
candidate. Per-request scoping (auto-`run_scoped` around every
handler) might be another. Supervision per-route isn't, because the
right restart policy is too app-specific. The principle is that any
helper Edda ships has to read as a thin wrapper over a Saga pattern
that already works without it — not a new abstraction.
