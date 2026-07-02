---
title: Edda.Middleware
---

# Edda.Middleware

Tiny helpers for the ordinary function-wrapping style of middleware. These are
also exported from the main `Edda` barrel.

```saga
app
|> map_request rewrite_request
|> map_response add_common_headers
```

Use these when middleware only needs to transform the request before a handler
or the response after a handler. Use plain wrapper functions or effect handlers
for short-circuiting, timing, auth, request-scoped context, or other behavior
that needs custom control flow.
