---
name: go-gin-server
description: The user's rules for production Go HTTP servers with Gin - explicit http.Server timeouts, trusted proxies, body size limits, graceful shutdown, middleware order, thin handlers that keep HTTP out of services, and testing handlers with httptest. Use this whenever building, reviewing or debugging a Go web server or REST API (Gin or net/http), adding middleware or handlers, or writing handler tests, even if Gin isn't named.
---

# Go web servers with Gin

Defaults are not production settings. `r.Run()` gives you no timeouts and no graceful shutdown, so build the `http.Server` yourself. Reference code is in `references/`.

**Gin or not:** use Gin when the project already does. In a plain `net/http` project, don't add Gin (stdlib first, see `go-guidelines`). The server configuration, shutdown, middleware and handler rules below apply to `http.Server` and `http.ServeMux` all the same; only the Gin-specific calls change.

## Server configuration

Configure at least:
- `gin.SetMode(gin.ReleaseMode)`: debug mode (the default) prints every route and a warning at startup.
- `r.SetTrustedProxies(...)`: by default Gin trusts every proxy, so anyone can fake `c.ClientIP()` with `X-Forwarded-For`. Pass your load balancer's range (e.g. `[]string{"10.0.0.0/8"}`), or `nil` when nothing sits in front.
- Body limit with `http.MaxBytesReader`: Gin has none, so a client can stream a huge body into `ShouldBindJSON`.
- `ReadHeaderTimeout` (headers), `ReadTimeout` (headers + body), `WriteTimeout` (handler + response), `IdleTimeout` (unused keep-alive).
- No proxy in front: `ListenAndServeTLS`, not `ListenAndServe`.

Streaming (SSE, long polling) needs a **separate server with `WriteTimeout: 0`**, or streams get cut at the timeout.

**Graceful shutdown:** run `ListenAndServe` in a goroutine, wait on `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`, then `srv.Shutdown(ctx)` with a timeout. It stops accepting connections and waits for in-flight requests. Treat `http.ErrServerClosed` as normal.

Example: `references/d1-configuration-and-graceful-shutdown/main.go`

## Middleware

A middleware is a handler that runs before and after the next one (`c.Next()`). Register globally with `router.Use`, per group with `group.Use`, or per route: `router.GET("/admin", adminOnly(), handler)`.

Usual set: request ID (find it in logs), tracing (OpenTelemetry), one log line per request (method, path, status, duration), recovery (panic → 500), CORS, compression, rate limiting, body size limit, auth.

Example: `references/d2-middleware/main.go`

## Handlers

Handlers are thin adapters. Services must not know about HTTP or Gin.

1. Bind and validate: `ShouldBindJSON` / `ShouldBindUri` / `ShouldBindQuery` with binding tags (`required`, `gt=0`). On failure wrap with `ErrInvalidInput` and send to the shared error responder (`respondError`).
2. Convert the request DTO into a domain type, so JSON tags never reach the service.
3. Pass `c.Request.Context()` to the service, never `c`.
4. Service errors go to the same `respondError`, which maps sentinel errors to status codes in one place.
5. No business logic in the handler.

Example: `references/d3-handlers/main.go`

## Testing handlers

No real server needed: a Gin engine is an `http.Handler`.
```go
gin.SetMode(gin.TestMode) // otherwise debug output floods test logs
req := httptest.NewRequest(http.MethodPost, "/orders", strings.NewReader(body))
w := httptest.NewRecorder()
router.ServeHTTP(w, req)
// assert w.Code, w.Body
```
Example: `references/d4-testing-handlers/`

## Review checklist

- `r.Run()` or an `http.Server` without `ReadHeaderTimeout`/`ReadTimeout`/`WriteTimeout`/`IdleTimeout`
- no graceful shutdown on SIGTERM, or `http.ErrServerClosed` treated as a failure
- Gin still in debug mode, or trusting every proxy
- no request body limit
- streaming endpoints on a server with a non-zero `WriteTimeout`
- handlers with business logic, JSON-tagged request types passed into services, or `c` passed instead of `c.Request.Context()`
- error-to-status mapping repeated per handler instead of one `respondError`

For project layout, naming and error handling, also follow the `go-guidelines` skill.
