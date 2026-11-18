---
name: go-grpc
description: The user's rules for gRPC and Protobuf in Go - buf for generation, linting and breaking-change checks, proto evolution rules (never reuse field numbers, reserved, _UNSPECIFIED enums), servers embedding Unimplemented*Server, status codes, client deadlines, and the three kinds of streaming. Use this whenever the user works with .proto files, buf, protoc, gRPC servers or clients, streaming RPCs, or asks how to change a proto without breaking clients.
---

# gRPC and Protobuf

Reference code is in `references/` (`f1-protobuf-and-buf` has `buf.yaml`, `buf.gen.yaml` and an example proto).

## Protobuf with buf

Use `buf` instead of hand-installed `protoc` and plugins: `go install github.com/bufbuild/buf/cmd/buf@latest`.
- `buf.yaml`: module location, lint and breaking rules.
- `buf.gen.yaml`: plugins, options, output paths.

CI commands:
```bash
buf lint
buf breaking --against '.git#branch=main'   # fails on changes that break existing clients
buf generate && git diff --exit-code        # fails if generated code wasn't committed
```

### Evolving a proto without breaking clients
The **field number is the contract, not the name.**
- Never reuse a field number. When deleting a field, add its number (and name) to `reserved`.
- Never change a field's type. Add a new field instead.
- Adding a field is always safe; old clients ignore it.
- Use 1–15 for the most frequent fields (1 byte on the wire; 16+ take 2).
- The first enum value (0) is `<ENUM>_UNSPECIFIED`, because 0 is what an unset field reads as.

## Server

A gRPC server is a struct implementing the generated interface.
- Embed `orderv1.UnimplementedOrderServiceServer` **by value** (not `*Unimplemented...`) even if you implement every RPC, so an RPC added to the proto later returns `codes.Unimplemented` instead of breaking the build or behaviour. Current generated code panics at registration when it's embedded as a nil pointer.
- Return **status errors** (`status.Error(codes.NotFound, "...")`). A plain error reaches the client as `codes.Unknown`.
- Clients **always** set a deadline: `ctx, cancel := context.WithTimeout(ctx, 2*time.Second)`. Servers respect `ctx.Done()`.

The `stream` keyword in the proto picks the RPC kind: unary (a normal function call), server streaming, client streaming, bidirectional.

Examples: `references/f2-grpc-server/main.go`, `references/f2-unary-rpc/main.go`

## Streaming

Same model as producer/consumer channels (see the `go-concurrency` skill):
- **Server streaming:** one request, many responses (watch new orders).
- **Client streaming:** many requests, one response at the end (upload items).
- **Bidirectional:** both send whenever they want (chat).

Rules:
- The sender closes its side when done: `CloseSend()` on the client; returning from the method on the server.
- Every receive loop checks `io.EOF` first: the other side finished, like a closed channel.
- The real result or error comes from the receiving call (`Recv()` / `CloseAndRecv()`), not from `Send()`.

Example: `references/f3-streaming/main.go`

## Review checklist

- a removed field without `reserved`, a reused field number, or a changed field type
- an enum whose zero value isn't `_UNSPECIFIED`
- generated code edited by hand or not committed; no `buf lint` / `buf breaking` in CI
- server not embedding `Unimplemented*Server` by value
- plain `error` returned from an RPC instead of a `status` error
- client calls without a deadline
- stream loops that don't check `io.EOF`, or a sender that never closes its side
