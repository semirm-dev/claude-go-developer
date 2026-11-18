---
name: go-memory
description: The user's approach to Go memory and allocation work - measure with benchmarks and pprof first, tune GOGC and GOMEMLIMIT second, rewrite code last; interface boxing, slice and map reuse, struct field layout, sync.Pool buffer pools. Use this whenever the user mentions Go performance, allocations, GC pressure, high memory, OOM kills, pprof, benchmarks, escape analysis, GOGC, GOMEMLIMIT or sync.Pool, or asks to optimize a Go hot path.
---

# Go memory and allocations

**Measure, then tune, then rewrite.** Don't change code to save allocations until a profile says where they come from, and try runtime settings before rewriting. Reference code is in `references/`.

## 1. Measure

```bash
go test -bench . -benchmem -memprofile mem.out       # allocs per op, saved to a profile
go tool pprof -alloc_space -top mem.out              # where they come from
go tool pprof -alloc_space http://localhost:6060/debug/pprof/allocs   # live service (import _ "net/http/pprof")
go build -gcflags="-m" ./...                         # what escapes to the heap
```

Report numbers before and after any change. An optimisation without a benchmark delta is a guess.

Example: `references/c1-measure-tune-rewrite/`

## 2. Tune

- **`GOGC`** (default `100`) sets when the GC runs, as a percentage of the live heap. With 100MB live: `GOGC=50` runs at 150MB, `GOGC=200` at 300MB. Higher means fewer collections and more memory. The target is recalculated after every GC.
- **`GOMEMLIMIT`** is a soft limit for the whole process. The GC runs more often as memory approaches it, which avoids container OOM kills. Set it to about 80–90% of the container limit: 8GiB limit → `GOMEMLIMIT=7GiB`.

## 3. Rewrite (only the hot path the profile showed)

### Interface boxing
Passing a concrete value as `any` usually copies it to the heap. It hides in `...any` parameters, for example loggers: `logger.Info("done", "user_id", big)` boxes `big` even when Info is disabled. Use typed attributes:
```go
logger.LogAttrs(ctx, slog.LevelInfo, "done", slog.Int("user_id", big))
```
When testing this, use a variable above 255: constants and 0–255 don't allocate when boxed, so they hide the effect.
Example: `references/c2-avoid-interface-boxing/main.go`

### Slices and maps: size up front, reuse in loops
- `buf := make([]byte, 0, 1024)`, then reset in the loop with `buf = buf[:0]` to keep capacity.
- Slice of pointers: `clear(s)` first so the GC can free the pointees, then `s = s[:0]`.
- `m := make(map[string]int, len(items))`. To reuse it in a loop, `clear(m)`: it keeps the buckets, so refilling to a similar size allocates nothing. Re-create it only after a one-off spike, because `clear` never shrinks a map and the memory stays held.
Example: `references/c3-slices-and-maps/main.go`

### Struct layout
- Order fields biggest-first to cut padding. Let the linter do it:
  `go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest` then `fieldalignment -fix -diff ./...`
- For big slices of structs, avoid pointers. A pointer-free struct is marked `noscan` and the GC skips it. One pointer field loses that, and `string`, `[]byte`, `map`, `chan`, `func`, interfaces and even `time.Time` all count.
Example: `references/c4-struct-layout/main.go`

### sync.Pool (buffer pools)
Ask: what type do you need, and does it grow?
- **`*bytes.Buffer` pool** for growing output (`json.NewEncoder`, `fmt.Fprintf`, `template.Execute`). `Reset()` before `Put`. If capacity grew past a `maxCap`, drop it instead of returning it, or the pool hoards memory.
- **`*[]byte` pool** for fixed-size chunks (`r.Read(buf)`, `io.CopyBuffer`). Restore full length before `Put` (`*b = (*b)[:size]`). If capacity no longer matches, drop it (smaller panics on `[:size]`, bigger wastes memory).
- Both: `defer pool.Put(x)` right after `Get` when the scope allows; never touch the buffer after `Put` (copy out first); it's not a cache, the GC may empty it at any time.
Example: `references/c5-sync-pool/main.go`

## Review checklist

- an optimisation with no benchmark or profile before and after
- code rewritten where `GOGC`/`GOMEMLIMIT` would do; a container without `GOMEMLIMIT`
- `...any` loggers (`Info(msg, k, v)`) in hot paths instead of `LogAttrs`
- slices and maps grown in loops without capacity; maps re-created every iteration instead of `clear`ed; a `clear`ed map still holding memory from a one-off spike
- pointer-heavy structs in large slices; fields not ordered by size
- pooled buffers not reset, oversized buffers returned to the pool, buffers used after `Put`
