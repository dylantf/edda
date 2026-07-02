---
title: Edda.Crypto
---

## Types

### CookieSecret

```saga
opaque type CookieSecret
```

Opaque signing secret used for signed cookie values.

### CookieSecrets

```saga
record CookieSecrets {
  current: CookieSecret,
  previous: List CookieSecret
}
```

Signing secrets for key rotation. New cookies are signed with `current`;
verification accepts `current` and every secret in `previous`.

### SignedCookieError

```saga
type SignedCookieError =
  | SignedCookieMissing String
  | SignedCookieBadFormat
  | SignedCookieBadSignature
  | SignedCookieValueNotUtf8
  deriving (Debug, Eq)
```

Errors returned while reading or verifying a signed cookie.

## Functions

### cookie_secret

```saga
fun cookie_secret : String -> CookieSecret
```

Build a cookie signing secret from a UTF-8 string.

Use a high-entropy random value from configuration in real applications.

### cookie_secret_bytes

```saga
fun cookie_secret_bytes : BitString -> CookieSecret
```

Build a cookie signing secret from raw bytes.

### cookie_secrets

```saga
fun cookie_secrets : CookieSecret -> CookieSecrets
```

Build a non-rotating secret set from the current signing secret.

### cookie_secrets_with_previous

```saga
fun cookie_secrets_with_previous : CookieSecret -> List CookieSecret -> CookieSecrets
```

Build a rotating secret set. Signing uses `current`; verification also accepts
`previous`.

### sign_cookie_value

```saga
fun sign_cookie_value : CookieSecret -> String -> String
```

Sign a cookie value. The result is cookie-safe but not encrypted.

### sign_cookie_value_with_secrets

```saga
fun sign_cookie_value_with_secrets : CookieSecrets -> String -> String
```

Sign a cookie value with the current secret from a rotating secret set.

### verify_cookie_value

```saga
fun verify_cookie_value : CookieSecret -> String -> Result String SignedCookieError
```

Verify a signed cookie value and return the original unsigned value.

### verify_cookie_value_with_secrets

```saga
fun verify_cookie_value_with_secrets : CookieSecrets -> String -> Result String SignedCookieError
```

Verify a signed cookie value against the current and previous secrets.

### signed_cookie

```saga
fun signed_cookie : CookieSecret -> String -> Request -> Result String SignedCookieError
```

Read and verify a named cookie from the request.

### signed_cookie_with_secrets

```saga
fun signed_cookie_with_secrets : CookieSecrets -> String -> Request -> Result String SignedCookieError
```

Read and verify a named cookie against current and previous secrets.

### set_signed_cookie

```saga
fun set_signed_cookie : CookieSecret -> String -> String -> Response -> Response
```

Add a signed cookie using default cookie options.

### set_signed_cookie_with

```saga
fun set_signed_cookie_with : CookieSecret -> String -> String -> CookieOptions -> Response -> Response
```

Add a signed cookie using explicit cookie options.

### set_signed_cookie_checked

```saga
fun set_signed_cookie_checked : CookieSecret -> String -> String -> Response -> Result Response CookieError
```

Add a signed cookie using default cookie options, validating the name.

### set_signed_cookie_with_checked

```saga
fun set_signed_cookie_with_checked : CookieSecret -> String -> String -> CookieOptions -> Response -> Result Response CookieError
```

Add a signed cookie using explicit cookie options, validating the name.

### set_signed_cookie_with_secrets

```saga
fun set_signed_cookie_with_secrets : CookieSecrets -> String -> String -> Response -> Response
```

Add a signed cookie with the current secret from a rotating secret set.

### set_signed_cookie_with_secrets_and_options

```saga
fun set_signed_cookie_with_secrets_and_options : CookieSecrets -> String -> String -> CookieOptions -> Response -> Response
```

Add a signed cookie with rotating secrets and explicit cookie options.
