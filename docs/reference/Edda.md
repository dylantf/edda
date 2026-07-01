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

### AcceptRange

```saga
record AcceptRange {
  media_type: String,
  quality: Int
}
```

A parsed `Accept` media range. `quality` is `0..1000`, where `1000` means
`q=1.0`.

### EncodingRange

```saga
record EncodingRange {
  encoding: String,
  quality: Int
}
```

A parsed `Accept-Encoding` coding range. `quality` is `0..1000`, where `1000`
means `q=1.0`.

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
  expose_headers: List String,
  allow_credentials: Bool,
  max_age: Maybe Int
}
```

CORS policy for `with_cors`. Use `"*"` in `allow_origins` for wildcard origin.
When credentials are enabled with wildcard origins, Edda echoes the request
origin instead of emitting `*`.

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

### UrlEncodedOptions

```saga
record UrlEncodedOptions {
  max_total_bytes: Maybe Int,
  max_field_count: Maybe Int,
  max_field_name_bytes: Maybe Int,
  max_field_value_bytes: Maybe Int
}
```

Size policy for URL-encoded query and form parsing. `Nothing` means no
Edda-level limit for that dimension.

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

### RequestParserOptions

```saga
record RequestParserOptions {
  urlencoded: UrlEncodedOptions,
  multipart: MultipartOptions
}
```

App-level parser policy used by the ambient request parser effect.

### UrlEncodedError

```saga
type UrlEncodedError =
  | InvalidPercentEscape String
  | InvalidUrlEncodedUtf8
  | UrlEncodedTotalTooLarge Int Int
  | UrlEncodedFieldCountTooLarge Int Int
  | UrlEncodedFieldNameTooLarge String Int Int
  | UrlEncodedFieldValueTooLarge String Int Int
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

### App

```saga
opaque type App
```

Root Edda application configuration.

### RunningApp

```saga
opaque type RunningApp
```

A running Edda server.

## Effects

### Skip

```saga
effect Skip {
  fun skip : Unit -> a
  fun method_not_allowed : List Method -> a
}
```

### RequestParserConfig

```saga
effect RequestParserConfig {
  fun request_parser_options : Unit -> RequestParserOptions
}
```

Ambient request parser configuration.

## Values

### create_app

```saga
fun create_app : App
```

Create an app with default HTTP config, default parser options, and a 404
fallback route.

### default_config

```saga
fun default_config : Config
```

Default HTTP server config used by `create_app`.

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

### default_urlencoded_options

```saga
fun default_urlencoded_options : UrlEncodedOptions
```

Default URL-encoded parsing policy: 64 KiB total input, 1024 fields, 1024-byte
field names, and 16 KiB field values.

### empty_multipart_form_values

```saga
fun empty_multipart_form_values : MultipartFormValues
```

Empty multipart form values.

### default_multipart_options

```saga
fun default_multipart_options : MultipartOptions
```

Default multipart parsing policy: 10 MiB total input/part/file, 1000 parts,
and 64 KiB text parts.

### default_request_parser_options

```saga
fun default_request_parser_options : RequestParserOptions
```

Default app-level request parser policy.

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

### accept_ranges

```saga
fun accept_ranges : Request -> List AcceptRange
```

Parse `Accept` headers into ordered media ranges. Missing `Accept` means
`*/*` at full quality.

### accepts

```saga
fun accepts : String -> Request -> Bool
```

Return whether a response content type is acceptable for the request.

### preferred_content_type

```saga
fun preferred_content_type : List String -> Request -> Maybe String
```

Pick the best server-offered content type. Higher q-values win, then more
specific media ranges, then the order of the offered list.

### accept_encoding_ranges

```saga
fun accept_encoding_ranges : Request -> List EncodingRange
```

Parse `Accept-Encoding` headers into ordered coding ranges. Missing
`Accept-Encoding` means any encoding is acceptable.

### accepts_encoding

```saga
fun accepts_encoding : String -> Request -> Bool
```

Return whether a response content encoding is acceptable for the request.

### preferred_content_encoding

```saga
fun preferred_content_encoding : List String -> Request -> Maybe String
```

Pick the best server-offered content encoding. Higher q-values win, then
explicit encodings beat wildcard matches, then the order of the offered list.
`identity` is acceptable unless explicitly refused.

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
fun query_values : Request -> Result FormValues UrlEncodedError needs {RequestParserConfig}
```

Decode the request query string as URL-encoded form values. `+` decodes to a
space and `%XX` escapes decode as UTF-8 bytes. Uses the ambient parser policy.

### query_values_with

```saga
fun query_values_with : UrlEncodedOptions -> Request -> Result FormValues UrlEncodedError
```

Decode the request query string with explicit URL-encoded parser limits.

### query_value

```saga
fun query_value : String -> Request -> Result (Maybe String) UrlEncodedError needs {RequestParserConfig}
```

Decode and return the first query value for `name`.

### query_value_with

```saga
fun query_value_with : UrlEncodedOptions -> String -> Request -> Result (Maybe String) UrlEncodedError
```

Decode and return the first query value for `name` with explicit limits.

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
fun form_values : Request -> Result FormValues FormError needs {RequestParserConfig}
```

Decode an `application/x-www-form-urlencoded` request body using the ambient
parser policy.

### form_values_with

```saga
fun form_values_with : UrlEncodedOptions -> Request -> Result FormValues FormError
```

Decode an `application/x-www-form-urlencoded` request body with explicit
URL-encoded parser limits.

### form_value

```saga
fun form_value : String -> Request -> Result (Maybe String) FormError needs {RequestParserConfig}
```

Decode and return the first form body value for `name`.

### form_value_with

```saga
fun form_value_with : UrlEncodedOptions -> String -> Request -> Result (Maybe String) FormError
```

Decode and return the first form body value for `name` with explicit limits.

### multipart_values

```saga
fun multipart_values : Request -> Result MultipartFormValues MultipartError needs {RequestParserConfig}
```

Decode a buffered `multipart/form-data` request body using the ambient parser
policy.

### multipart_values_with

```saga
fun multipart_values_with : MultipartOptions -> Request -> Result MultipartFormValues MultipartError
```

Decode a buffered `multipart/form-data` request body with explicit size limits.

### multipart_value

```saga
fun multipart_value : String -> Request -> Result (Maybe MultipartValue) MultipartError needs {RequestParserConfig}
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

### empty_response

```saga
fun empty_response : Int -> Response
```

Build an empty response with the given status.

### redirect

```saga
fun redirect : Int -> String -> Response
```

Build a redirect response with the given status and `Location`.

### bytes

```saga
fun bytes : Int -> BitString -> Response
```

Build a binary response without setting a default Content-Type.

### status

```saga
fun status : Int -> Response -> Response
```

Replace the response status while preserving headers and body.

### with_header

```saga
fun with_header : String -> String -> Response -> Response
```

Append a response header without removing existing values.

### with_headers

```saga
fun with_headers : List (String, String) -> Response -> Response
```

Append several response headers without removing existing values.

### replace_header

```saga
fun replace_header : String -> String -> Response -> Response
```

Set or replace a response header by name, case-insensitively.

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
(capturing any `:name` segments into params), the handler runs. If the path
does not match, the route skips. If the path matches but the method does not,
`choose` can use that information to produce `405 Method Not Allowed`.

### choose

```saga
fun choose : List (Request -> Response needs {Skip, ..e}) -> Request -> Response needs {..e}
```

Try each route in order. If no path matches, return `not_found`. If a path
matches but no route accepts the request method, return `405 Method Not
Allowed` with an `Allow` header. Matched paths get automatic `OPTIONS`, and
`HEAD` can use `GET` routes with the response body stripped.

### group

```saga
fun group : String -> List (Request -> Response needs {Skip, ..e}) -> Request -> Response needs {Skip, ..e}
```

Match a path prefix and run the inner routes against the remainder.
Captured `:name` params accumulate and are visible to sub-routes.

If no inner route matches, the group skips so the outer `choose` can continue.

### mount

```saga
fun mount : String -> (Request -> Response needs {..e}) -> Request -> Response needs {Skip, ..e}
```

Mount a sub-app at a prefix. Captured `:name` params accumulate and are visible
to the sub-app.

### use_routes

```saga
fun use_routes : (Request -> Response needs {RequestParserConfig}) -> App -> App
```

Set the app router.

### use_options

```saga
fun use_options : RequestParserOptions -> App -> App
```

Set app-wide request parser options.

### use_config

```saga
fun use_config : Config -> App -> App
```

Set the full underlying HTTP server config.

### use_port

```saga
fun use_port : Int -> App -> App
```

Set the HTTP port.

### use_server_events

```saga
fun use_server_events : Handler Server -> App -> App
```

Set the server event handler. If unset, `start` and `serve` discard server
events.

### start

```saga
fun start : App -> Result RunningApp String
```

Start an Edda app. Returns after the listener is bound.

### wait

```saga
fun wait : RunningApp -> Unit
```

Wait until a running app shuts down.

### serve

```saga
fun serve : App -> Result Unit String
```

Start an Edda app and block until it shuts down. If no server-event handler was
configured, discards server events.

### with_request_parser_options

```saga
fun with_request_parser_options : RequestParserOptions -> (Request -> Response needs {RequestParserConfig, ..e}) -> Request -> Response needs {..e}
```

Install request parser options for an app or sub-app.

### from_http

```saga
fun from_http : Http.Request -> Request
```

Lift a SagaHttp.Http.Request into an Edda.Request.

### to_handler

```saga
fun to_handler : (Request -> Response needs {RequestParserConfig, ..e}) -> Http.Request -> Response needs {..e}
```

Adapt an Edda app to the `Request -> Response` shape `SagaHttp.serve`
expects, installing Edda's default request parser options. Use as:
`SagaHttp.Http.serve config (to_handler app)`.

### to_handler_with

```saga
fun to_handler_with : RequestParserOptions -> (Request -> Response needs {RequestParserConfig, ..e}) -> Http.Request -> Response needs {..e}
```

Adapt an Edda app with explicit request parser options.
