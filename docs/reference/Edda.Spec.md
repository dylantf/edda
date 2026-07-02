---
title: Edda.Spec
---

Experimental sidecar for endpoint specifications. The core router API is
still `Edda.route`, `choose`, `group`, and `mount`; this module is a
prototype for type-driven route contracts that can also emit OpenAPI.

The builder produces a `RouteSchema`, which the router consumes for dispatch
and a spec generator consumes for OpenAPI emission. One definition, two
extractors, no runtime overhead on the dispatch path.

Status: experimental sidecar. It lives in-tree so it can evolve against real
Edda internals, but it is not the future primary route API yet. Schemas are
explicit `SchemaFor a` witnesses; the builder mechanics and type-check at
`performed_by` are the part worth pinning down first.

## Types

### Status

```saga
type Status =
  | Success
  | Created
  | NoContent
  | BadRequest
  | Unauthorized
  | Forbidden
  | NotFound
  | Conflict
  | Unprocessable
  | InternalError
  | Other Int
  deriving (Eq, Debug)
```

### Action

```saga
type alias Action = Request -> Response
```

### TypedAction

```saga
type alias TypedAction a = Request -> TypedResponse a
```

### TypedResponse

```saga
type TypedResponse a =
  | TypedResponse Response
```

### JsonSchema

```saga
type JsonSchema =
  | StringSchema
  | IntSchema
  | FloatSchema
  | BoolSchema
  | ArraySchema JsonSchema
  | ObjectSchema (List (String, JsonSchema)) (List String)
  | NullableSchema JsonSchema
```

### SchemaFor

```saga
type SchemaFor a =
  | SchemaFor JsonSchema
```

### Schema

```saga
type Schema =
  | Schema
```

### ContentSpec

```saga
record ContentSpec {
  content_type: String,
  schema: Maybe JsonSchema,
  description: Maybe String
}
```

### ParamMeta

```saga
record ParamMeta {
  name: String,
  description: String
}
```

### BodyMeta

```saga
record BodyMeta {
  content_type: String,
  description: String,
  schema: Maybe JsonSchema
}
```

### ResponseMeta

```saga
record ResponseMeta {
  content_type: String,
  description: Maybe String,
  schema: Maybe JsonSchema
}
```

### RouteSchema

```saga
record RouteSchema {
  method: Method,
  path: String,
  params: List ParamMeta,
  body: Maybe BodyMeta,
  responses: List (Status, ResponseMeta),
  action: Request -> Response
}
```

### RouteBuilder

```saga
record RouteBuilder a {
  method: Method,
  path: String,
  params: List ParamMeta,
  body: Maybe BodyMeta,
  responses: List (Status, ResponseMeta)
}
```

### OpenApiInfo

```saga
record OpenApiInfo {
  title: String,
  version: String
}
```

## Functions

### status_code

```saga
fun status_code : Status -> Int
```

### untyped

```saga
fun untyped : Response -> TypedResponse a
```

### ok

```saga
fun ok : a -> TypedResponse a where {a: ToJson}
```

### ok_status

```saga
fun ok_status : Status -> a -> TypedResponse a where {a: ToJson}
```

### text_response

```saga
fun text_response : Status -> String -> TypedResponse a
```

### no_content

```saga
fun no_content : TypedResponse a
```

### schema_for

```saga
fun schema_for : JsonSchema -> SchemaFor a
```

### schema_json

```saga
fun schema_json : SchemaFor a -> JsonSchema
```

### string_schema

```saga
fun string_schema : SchemaFor String
```

### int_schema

```saga
fun int_schema : SchemaFor Int
```

### float_schema

```saga
fun float_schema : SchemaFor Float
```

### bool_schema

```saga
fun bool_schema : SchemaFor Bool
```

### array_of

```saga
fun array_of : SchemaFor a -> SchemaFor (List a)
```

### nullable_of

```saga
fun nullable_of : SchemaFor a -> SchemaFor (Maybe a)
```

### object_of

```saga
fun object_of : List (String, JsonSchema) -> SchemaFor a
```

### optional

```saga
fun optional : String -> SchemaFor a -> SchemaFor a
```

### schema

```saga
fun schema : Schema
```

### schema_into

```saga
fun schema_into : a -> b -> SchemaFor (a -> b)
```

### schema_field

```saga
fun schema_field : String -> SchemaFor a -> SchemaFor (a -> b) -> SchemaFor b
```

### json_of

```saga
fun json_of : SchemaFor a -> ContentSpec
```

### plain_text

```saga
fun plain_text : ContentSpec
```

### no_body_spec

```saga
fun no_body_spec : ContentSpec
```

### describe

```saga
fun describe : String -> ContentSpec -> ContentSpec
```

### route

```saga
fun route : Method -> String -> RouteBuilder a
```

### path_param

```saga
fun path_param : String -> String -> RouteBuilder a -> RouteBuilder a
```

### json_body

```saga
fun json_body : SchemaFor body -> String -> RouteBuilder a -> RouteBuilder a
```

### responds

```saga
fun responds : Status -> ContentSpec -> RouteBuilder a -> RouteBuilder a
```

### responds_success_json

```saga
fun responds_success_json : Status -> SchemaFor a -> String -> RouteBuilder a -> RouteBuilder a
```

### performed_by

```saga
fun performed_by : Request -> TypedResponse a -> RouteBuilder a -> RouteSchema
```

### as_route

```saga
fun as_route : RouteSchema -> Request -> Response needs {Skip}
```

### to_openapi_json

```saga
fun to_openapi_json : OpenApiInfo -> List RouteSchema -> Json
```

### to_openapi

```saga
fun to_openapi : OpenApiInfo -> List RouteSchema -> String
```

### scalar_html

```saga
fun scalar_html : String -> String
```

### scalar_handler

```saga
fun scalar_handler : String -> Request -> Response
```

