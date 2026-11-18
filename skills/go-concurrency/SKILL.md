---
name: go-concurrency
description: The user's Go concurrency rules and patterns - goroutine lifecycle, who closes channels, context cancellation, worker pool, fan-out/fan-in, pipeline, semaphore and errgroup. Use this whenever Go code starts goroutines, uses channels, sync.WaitGroup, errgroup, context cancellation or rate limiting, when choosing a concurrency pattern, or when debugging deadlocks, goroutine leaks or "send on closed channel" panics.
---

# Go concurrency

Jargon: **producer** sends to a channel, **consumer** receives, **caller** starts both. Every pattern below builds on the three golden rules, so get those right first. Reference implementations are in `references/<pattern>/main.go`.

## The three golden rules

1. **One side must already be running.** Producer and consumer never run on the same goroutine, or you get `all goroutines are asleep - deadlock!`. If the caller consumes, start producers first; if it produces, start consumers first. A big buffer "works" only until there are more items than the buffer holds, so don't rely on it.
2. **Every goroutine has a known exit.** Before typing `go`, decide what makes it return: a closed channel (`for v := range ch`, or `v, ok := <-ch; if !ok { return }`) or a cancelled context (`case <-ctx.Done(): return`). Without one it leaks and holds its memory until the process exits.
3. **Context comes first.** Anything that waits (I/O, channel ops) takes `ctx context.Context` as its first argument and returns when it's done.

Always run tests with `go test -race ./...`.

Example: `references/b1-three-golden-rules/main.go`

## Closing channels

**The producer closes the channel; the consumer cancels the context.**

- Only the producer knows when sending is finished, so only it closes.
- Several producers: none of them closes. One coordinator does `wg.Wait(); close(ch)`. This prevents `send on closed channel` and `close of closed channel` panics.
- To stop a producer from the consumer side, cancel its context. That works only if the producer sends inside a select:
  ```go
  select {
  case <-ctx.Done():
      return
  case out <- v:
  }
  ```
- Closing a channel does not discard values still buffered in it.

Every channel in a design should have **exactly one closer**. Name it when reviewing.

Example: `references/b2-closing-channels/main.go`

## Choosing a pattern

| Situation | Pattern | Closers |
|---|---|---|
| You start the goroutines and want a fixed cap on concurrent DB/API work | **Worker pool**: N workers read one `input`, write one shared `results` | job producer closes `input`; coordinator (`wg.Wait`) closes `results` |
| Same, but each worker needs its own output (merge later) | **Fan-out/fan-in**: workers share `input`, each has its own `out`, `fanIn` merges | producer closes `input`; each worker closes its `out`; `fanIn` closes `results` |
| One huge input through ordered stages (read CSV, parse, add tax, save) | **Pipeline**: each stage reads previous `out`, writes its own | each stage closes its own `out`; last stage closes `results` |
| You *don't* control goroutine creation (HTTP handler per request, N tasks handed to you) and need at most K at once | **Semaphore**: buffered channel of size K as tokens | nothing to close; release every token you acquire |
| Several tasks where the first error should cancel the rest | **errgroup** (`golang.org/x/sync/errgroup`) | `g.Wait()` handles it |

Notes per pattern:
- **Fan-out/fan-in:** results arrive in completion order, not send order.
- **Pipeline:** slow stages create backpressure (earlier stages wait), and latency chains (100ms + 10ms = 110ms per item). Scale a slow stage by running it as a worker pool.
- **Semaphore:** with thousands of tasks, `Acquire` *before* starting the goroutine, or you pile up blocked goroutines (cheap, not free).
- **errgroup:** `errgroup.WithContext` cancels the rest on first error. The derived `ctx` is **always cancelled after `g.Wait()`**, so create a new one for anything after it. `g.SetLimit(n)` caps concurrency.

These overlap on purpose. Pick the simplest one that fits; errgroup with `SetLimit` replaces a hand-written worker pool when you don't need a results channel.

References: `references/b3-worker-pool/`, `b4-fan-out-fan-in/`, `b5-pipeline/`, `b6-semaphore/`, `b7-errgroup/`.

## Review checklist

- Every `go` statement: what makes it return? If nobody can answer, it leaks.
- Every channel: exactly one closer, and it's the producer or a coordinator.
- Sends that can block have a `ctx.Done()` case.
- Waiting functions take `ctx` first.
- `go test -race` is in the test command.
