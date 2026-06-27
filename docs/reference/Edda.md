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

### CookieError

```saga
type CookieError =
  | InvalidCookieName String
```

Cookie validation errors from checked cookie helpers.

### CookieOptions

```saga
record CookieOptions {
  path: Maybe String,
  domain: Maybe String,
  expires: Maybe NaiveDateTime,
  max_age: Maybe Int,
  http_only: Bool,
  secure: Bool,
  same_site: Maybe SameSite
}
```

Options for outgoing `Set-Cookie` headers. Names and values are percent-encoded
when emitted. `expires` is formatted as an HTTP cookie date with a `GMT` suffix;
pass a GMT/UTC `NaiveDateTime`. `SameSite=None` automatically emits `Secure`.

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

### FormValues

```saga
record FormValues {
  entries: List (String, String)
}
```

Decoded text form/query values. Preserves insertion order and duplicates.

### MultipartFile

```saga
record MultipartFile {
  filename: String,
  content_type: Maybe String,
  headers: List (String, String),
  body: BitString
}
```

A buffered file part from a multipart form.

### MultipartValue

```saga
type MultipartValue =
  | MultipartText String
  | MultipartFileValue MultipartFile
```

A text or file value from a multipart form.

### MultipartFormValues

```saga
record MultipartFormValues {
  entries: List (String, MultipartValue)
}
```

Decoded multipart form values. Preserves insertion order and duplicates.

### MultipartOptions

```saga
record MultipartOptions {
  max_total_bytes: Maybe Int,
  max_part_count: Maybe Int,
  max_part_bytes: Maybe Int,
  max_text_bytes: Maybe Int,
  max_file_bytes: Maybe Int
}
```

Size policy for buffered multipart parsing. `Nothing` means no Edda-level limit
for that dimension.

### UrlEncodedError

```saga
type UrlEncodedError =
  | InvalidPercentEscape String
  | InvalidUrlEncodedUtf8
```

Errors from URL-encoded query/form decoding.

### FormError

```saga
type FormError =
  | FormMissingBody
  | FormBodyNotUtf8
  | FormMissingContentType
  | FormUnexpectedContentType String
  | FormUrlEncodedError UrlEncodedError
```

Errors from `application/x-www-form-urlencoded` request parsing.

### MultipartError

```saga
type MultipartError =
  | MultipartMissingBody
  | MultipartMissingContentType
  | MultipartUnexpectedContentType String
  | MultipartMissingBoundary
  | MultipartInvalidBoundary String
  | MultipartMalformed String
  | MultipartTotalTooLarge Int Int
  | MultipartPartCountTooLarge Int Int
  | MultipartPartTooLarge Int Int
  | MultipartTextTooLarge String Int Int
  | MultipartFileTooLarge String Int Int
  | MultipartPartHeadersNotUtf8
  | MultipartMissingName
  | MultipartTextNotUtf8 String
```

Errors from `multipart/form-data` request parsing.

### StaticPathError

```saga
type StaticPathError =
  | InvalidStaticPercentEscape String
  | InvalidStaticPathUtf8
  | UnsafeStaticPathSegment String
```

Errors from decoding and validating a static request path.

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

### empty_form_values

```saga
fun empty_form_values : FormValues
```

Empty form values.

### empty_multipart_form_values

```saga
fun empty_multipart_form_values : MultipartFormValues
```

Empty multipart form values.

### default_multipart_options

```saga
fun default_multipart_options : MultipartOptions
```

Default multipart parsing policy: no Edda-level size limits.

## Functions

### method_str

```saga
fun method_str : Method -> String
```

Return the wire string for a method. Prefer `show method` in new code; this
helper remains for compatibility.

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

### query_values

```saga
fun query_values : Request -> Result FormValues UrlEncodedError
```

Decode the request query string as URL-encoded form values. `+` decodes to a
space and `%XX` escapes decode as UTF-8 bytes.

### query_value

```saga
fun query_value : String -> Request -> Result (Maybe String) UrlEncodedError
```

Decode and return the first query value for `name`.

### cookies

```saga
fun cookies : Request -> List (String, String)
```

Parse Cookie headers into ordered `(name, value)` pairs.
Duplicate names are preserved. Valid percent escapes in names and values are
decoded.

### cookie

```saga
fun cookie : String -> Request -> Maybe String
```

Look up the first cookie value by name.

### form_values_from_entries

```saga
fun form_values_from_entries : List (String, String) -> FormValues
```

Build form values from ordered entries.

### multipart_form_values_from_entries

```saga
fun multipart_form_values_from_entries : List (String, MultipartValue) -> MultipartFormValues
```

Build multipart form values from ordered entries.

### form_entries

```saga
fun form_entries : FormValues -> List (String, String)
```

Return ordered form entries.

### form_get

```saga
fun form_get : String -> FormValues -> Maybe String
```

Return the first value for `name`.

### form_get_all

```saga
fun form_get_all : String -> FormValues -> List String
```

Return all values for `name`, preserving insertion order.

### form_append

```saga
fun form_append : String -> String -> FormValues -> FormValues
```

Append a value for `name`.

### form_set

```saga
fun form_set : String -> String -> FormValues -> FormValues
```

Replace all existing values for `name` with one value appended at the end.

### form_delete

```saga
fun form_delete : String -> FormValues -> FormValues
```

Delete all values for `name`.

### form_values

```saga
fun form_values : Request -> Result FormValues FormError
```

Decode an `application/x-www-form-urlencoded` request body.

### form_value

```saga
fun form_value : String -> Request -> Result (Maybe String) FormError
```

Decode and return the first form body value for `name`.

### multipart_values

```saga
fun multipart_values : Request -> Result MultipartFormValues MultipartError
```

Decode a buffered `multipart/form-data` request body.

### multipart_values_with

```saga
fun multipart_values_with : MultipartOptions -> Request -> Result MultipartFormValues MultipartError
```

Decode a buffered `multipart/form-data` request body with explicit size limits.

### multipart_value

```saga
fun multipart_value : String -> Request -> Result (Maybe MultipartValue) MultipartError
```

Decode and return the first multipart value for `name`.

### multipart_entries

```saga
fun multipart_entries : MultipartFormValues -> List (String, MultipartValue)
```

Return ordered multipart form entries.

### multipart_get

```saga
fun multipart_get : String -> MultipartFormValues -> Maybe MultipartValue
```

Return the first multipart value for `name`.

### multipart_get_all

```saga
fun multipart_get_all : String -> MultipartFormValues -> List MultipartValue
```

Return all multipart values for `name`, preserving insertion order.

### multipart_text

```saga
fun multipart_text : String -> MultipartFormValues -> Maybe String
```

Return the first text multipart value for `name`.

### multipart_file

```saga
fun multipart_file : String -> MultipartFormValues -> Maybe MultipartFile
```

Return the first file multipart value for `name`.

### sanitize_filename

```saga
fun sanitize_filename : String -> Maybe String
```

Return a client filename when it is safe to treat as a basename. Preserves
ordinary filename characters, including whitespace, but rejects empty names,
`.`/`..`, path separators, and ASCII control bytes.

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

### file_response

```saga
fun file_response : String -> Response needs {File}
```

Serve a specific filesystem path as a buffered response. The content type is
inferred from the file extension.

### file_response_as

```saga
fun file_response_as : String -> String -> Response needs {File}
```

Serve a specific filesystem path as a buffered response with an explicit
content type.

### static_dir

```saga
fun static_dir : String -> String -> Request -> Response needs {Skip, File}
```

Serve files from `root` under `url_prefix`. The matched prefix is stripped, the
remaining path is percent-decoded, and unsafe segments (`.`, `..`, decoded
slashes, and backslashes) are rejected.

### set_cookie

```saga
fun set_cookie : String -> String -> Response -> Response
```

Add a `Set-Cookie` response header with default options. Cookie names and
values are percent-encoded.

### set_cookie_with

```saga
fun set_cookie_with : String -> String -> CookieOptions -> Response -> Response
```

Add a `Set-Cookie` response header with explicit options. Cookie names and
values are percent-encoded. `SameSite=None` automatically emits `Secure`.

### valid_cookie_name

```saga
fun valid_cookie_name : String -> Bool
```

Return whether a cookie name is a non-empty RFC token.

### set_cookie_checked

```saga
fun set_cookie_checked : String -> String -> Response -> Result Response CookieError
```

Add a cookie with default options, returning `InvalidCookieName` for invalid
names.

### set_cookie_with_checked

```saga
fun set_cookie_with_checked : String -> String -> CookieOptions -> Response -> Result Response CookieError
```

Add a cookie with explicit options, returning `InvalidCookieName` for invalid
names.

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
when deleting scoped cookies. Emits both `Max-Age=0` and an old `Expires`
date.

### delete_cookie_checked

```saga
fun delete_cookie_checked : String -> Response -> Result Response CookieError
```

Expire a cookie with default options, returning `InvalidCookieName` for invalid
names.

### delete_cookie_with_checked

```saga
fun delete_cookie_with_checked : String -> CookieOptions -> Response -> Result Response CookieError
```

Expire a cookie with explicit options, returning `InvalidCookieName` for invalid
names. Emits both `Max-Age=0` and an old `Expires` date.

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
