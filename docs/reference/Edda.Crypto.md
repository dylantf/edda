---
title: Edda.Crypto
---

## Types

### CookieSecret

```saga
opaque type CookieSecret
```

Opaque signing secret used for signed cookie values.

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

### sign_cookie_value

```saga
fun sign_cookie_value : CookieSecret -> String -> String
```

Sign a cookie value. The result is cookie-safe but not encrypted.

### verify_cookie_value

```saga
fun verify_cookie_value : CookieSecret -> String -> Result String SignedCookieError
```

Verify a signed cookie value and return the original unsigned value.

### signed_cookie

```saga
fun signed_cookie : CookieSecret -> String -> Request -> Result String SignedCookieError
```

Read and verify a named cookie from the request.

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

