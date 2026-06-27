# Repository Guidelines

## Project Structure & Module Organization

Edda is a Saga library plus a small demo server. Core library modules live in
`lib/`: `Edda.saga` contains routing and request adaptation, `Json.saga`
contains JSON helpers, and `Spec.saga` contains the experimental OpenAPI route
schema builder. Demo applications live in `src/`, with `src/Main.saga` wiring
the demo server and `src/Demo/*.saga` showing routing, middleware, errors,
JSON, and specs. User-facing docs live in `docs/guide/` and generated-style API
references live in `docs/reference/`. Design rationale and open questions are
kept in `planning/design.md`. In-process regression tests live in `tests/` and
exercise the public Edda API without starting a server.

## Build, Test, and Development Commands

- `saga fmt <filename>`: format Saga source file. Run this before `saga build` when
  changing `.saga` files.
- `saga build`: compile the library, docs examples, and demo binary. Run this
  before submitting changes.
- `saga test`: run the `Std.Test` suite in `tests/`. Use this for router,
  request parser, and response helper regressions.
- `saga run`: run `src/Main.saga`, which starts the demo server defined in
  `project.toml`.
- `nix develop`: enter the project development shell when working on systems
  that use the provided `flake.nix`.

## Coding Style & Naming Conventions

Use existing Saga style: two-space indentation inside records, handlers, and
case arms; short `snake_case` function names; `PascalCase` types and variants.
Saga does not use significant whitespace for function calls, so do not break
arguments onto multiple lines unless the whole call is wrapped in parentheses.
Keep public APIs small and documented with `#@` comments when they are intended
for generated reference docs. Use `saga fmt` for mechanical formatting. Prefer
composing ordinary functions and effects over adding framework-specific
abstractions. Keep library code in `lib/` and examples in `src/Demo/`.

## Testing Guidelines

Run `saga test` after library changes and `saga build` before handing work back.
Prefer in-process tests in `tests/` for pure routing and request/response
helpers. Add demo routes in `src/Demo/` when behavior needs browser/manual
exercise, for example file uploads, sessions, or static assets.

## Agent-Specific Instructions

Read `planning/design.md` before making architectural changes. For Saga
language help, use `~/projects/saga-website/public/llms.txt` as the guide index, or
`~/projects/saga-website/public/syntax-reference.md` to double-check surface syntax.
Avoid reverting unrelated work. Treat `Edda.Spec` as experimental unless the
task explicitly asks to stabilize it. If Saga hits an unexpected compiler panic
or runtime panic, pause work and report it instead of working around it; those
should be fixed in the compiler.
