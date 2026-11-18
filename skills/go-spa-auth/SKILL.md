---
name: go-spa-auth
description: The user's auth design for single-page apps with a Go backend - short-lived JWT access tokens in frontend memory, rotating refresh tokens in an HttpOnly cookie stored hashed, where CSRF protection applies, and "Login with Google" via OAuth with state and PKCE issuing your own tokens. Use this whenever implementing or reviewing login, logout, token refresh, sessions, cookies, CSRF, CORS for an API, or OAuth/OIDC sign-in in Go.
---

# Auth for SPAs

Security-relevant: write the failing test first (expired token rejected, reused refresh token rejected, cross-site POST rejected), watch it fail, then implement. Reference code is in `references/`.

## Access and refresh tokens

**Access token:** a JWT living 5–15 minutes, kept in **frontend memory** (not localStorage), sent as `Authorization: Bearer ...`. Holds user id, role and other non-sensitive claims.

**Refresh token:** a long-lived (days–weeks) **random string, not a JWT**.
- Set by the backend as a cookie: `HttpOnly; Secure; SameSite=Lax; Path=/auth`.
  - `HttpOnly`: JavaScript can't read it. `Secure`: HTTPS only. `SameSite=Lax`: not sent on cross-site POSTs. `Path=/auth`: only sent to `/auth/refresh` and `/auth/logout`.
- Stored in the database **hashed (SHA-256)**, never plain.
- `POST /auth/refresh` returns a new access token and **rotates** the refresh token: the old one is revoked/marked used. A used token showing up again means theft, so treat it as such (revoke the family).

`https://app.example.com` and `https://api.example.com` are different origins; CORS decides whether the app's JavaScript can read API responses.

Example: `references/e1-access-and-refresh-tokens/main.go`

## CSRF

CSRF only matters for endpoints authenticated by **cookies**. Endpoints reading `Authorization` headers are not exposed. Here that means the `/auth` endpoints.

- `SameSite=Lax` is enough in modern browsers when the API is same-site.
- If the API is on another site and the cookie must be `SameSite=None`, add both:
  - `Origin` header must be in an allow list.
  - Double-submit token: the same random value in a cookie and in an `X-CSRF-Token` header. A foreign site can't read the cookie, so it can't set the header.
- Always check `Origin` on `/auth/login`, whatever the `SameSite` setting: login isn't authenticated by a cookie, so `SameSite` doesn't protect it, and another site could log the user into the attacker's account.

Example: `references/e2-csrf/main.go`

## Login with Google (OAuth + PKCE)

Google only **authenticates** the user. You still issue your own access and refresh tokens. Keep Google's tokens only if you'll call Google APIs later.

1. **`GET /auth/google/login`** (a browser navigation, not `fetch`):
   set a short-lived cookie with `state` (random, against CSRF) and the PKCE `verifier` (against a stolen `code`), then redirect to Google with `state` and the verifier's S256 challenge.
2. **`GET /auth/google/callback?code=...&state=...`**:
   read `state` and `verifier` from the cookie, compare `state` with the query, exchange `code` + `verifier` for an ID token, verify it, find or create the user, issue your own tokens.

The state/verifier cookie must be `SameSite=Lax`, not `Strict`: the callback is a cross-site redirect from Google and the browser would drop a `Strict` cookie.

Example: `references/e3-oauth-login/main.go`

## Review checklist

- access tokens in localStorage/sessionStorage, or living longer than 15 minutes
- refresh token that is a JWT, readable by JavaScript, missing `Secure`/`SameSite`/`Path=/auth`, or stored unhashed
- refresh without rotation, or a reused refresh token not treated as theft
- cookie-authenticated endpoints with `SameSite=None` and no Origin check or CSRF token; `/auth/login` without an `Origin` check
- OAuth without `state` or PKCE, a `Strict` state cookie, or Google's tokens used as your own session
