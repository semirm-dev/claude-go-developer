---
name: go-guidelines
description: The user's Go coding guidelines - naming, structs, interfaces, error handling, dependencies and project layout (group by feature, not by layer). Use this whenever writing, reviewing, refactoring or designing ANY Go code, creating a new Go package or project, naming Go types or packages, deciding where an interface goes, or laying out a Go repository - even if the user doesn't mention guidelines or style.
---

# Go coding guidelines

Go is not OOP. Structs are not classes and interfaces are not Java interfaces. Most bad Go code comes from importing habits from class-based languages, so when something feels "enterprise", stop and simplify.

Reference code lives in `references/` (one folder per topic). Open the matching file when you need a concrete pattern; the rules below are the summary.

## Naming: read it at the call site

A name is good if it reads well where it is used.

- **Length follows scope.** `i` is fine in a three-line loop, not 200 lines from its declaration.
- **Don't stutter.** The package name is part of the name: `transport.Config`, not `transport.TransportConfig`.
- **Drop suffixes and prefixes that add no meaning:** `Impl`, `Base`, `Default`, etc.
- **camelCase / PascalCase everywhere,** constants included. Never snake_case.
- **Package names:** lowercase, one word, no underscores. Name what the package *provides* (`discounts`, `inventory`), not a layer or data type (`controllers`, `services`, `models`, `utils`).

Example: `references/a1-naming/main.go`

## Structs: state containers, not classes

A struct groups related state (a connection pool, a cache, an order item). If there is no state to hold, write a plain function that takes its inputs as arguments.

- No getters/setters for fields without logic. Export the field.
- Pass the values a function needs, not the whole struct. If it needs `total` and `currency`, pass those two.
- No god objects (one struct holding the DB, an HTTP client, config and a logger).

Example: `references/a2-structs/main.go`

## Interfaces: the consumer owns them

**Accept interfaces, return structs.** Returning an interface hides every method not in it, so it keeps growing until it's a bloated copy of the struct, and every mock has to implement all of it.

- Define the interface **where it is used**, next to the code that calls it, with only the methods that caller needs.
- Add one only when there are **two or more real implementations**, or to decouple from something you don't control (payment API, message broker, cloud SDK) so tests can fake it.
- **Red flag:** an interface and its only implementation in the same file. That's the same type written twice.
- Never wrap your own service in an interface (`type OrderService interface` next to `type Service struct`).

Example: `references/a3-interfaces/main.go`

## Errors: handle it or return it, never both

Otherwise one failure gets logged three times in three layers and the logs lie about what happened.

- **Handle:** log it, retry, fall back, then stop. Do not also return it.
- **Return:** add context and pass it up. Do not log it. Wrap with `%w` so callers can use `errors.Is` / `errors.As`: `fmt.Errorf("user fetch failed: %w", err)`.
- Messages are lowercase with no trailing punctuation, because they get embedded in other messages: `user fetch failed: connection reset by peer`.
- Sentinel errors for conditions callers branch on: `var ErrNotFound = errors.New("order not found")`.

Example: `references/a4-error-handling/main.go`

## Formatting: blank lines between logical steps

gofmt doesn't decide where blank lines go, so write function bodies as short paragraphs. One blank line goes between logical steps. A wall of statements is hard to scan.

- **Keep together:** a call and its `if err != nil` check; the assignments that configure one value; a statement that only prepares the next block (e.g. `sort` right before the loop over the sorted slice).
- **Separate with a blank line:** each step that does a different job (defaults, setup, each call with its check, each loop or `if` block); also before the final `return` once the function has more than one step.
- **Leave short functions compact.** A single call, its check and the return need no blank lines.

```go
func LoadChart(ref, version string, settings *cli.EnvSettings) (*Chart, error) {
	opts := action.ChartPathOptions{Version: version}

	path, err := opts.LocateChart(ref, settings)
	if err != nil {
		return nil, fmt.Errorf("locate chart: %w", err)
	}

	c, err := loader.Load(path)
	if err != nil {
		return nil, fmt.Errorf("load chart: %w", err)
	}

	return &Chart{chart: c}, nil
}
```

## Declaration order: read a file top-down

A reader should meet the package's API before its plumbing. Within one file:

1. **`const`, then `var`** (sentinel errors included), each in one parenthesised block, exported before unexported.
2. **Types, each followed by its own code:**
   - the `type` declaration
   - the interfaces it consumes (consumer-owned, so they sit next to the struct that holds them)
   - its constructor `NewX`
   - its exported methods, then its unexported methods
3. **Plain exported functions** not tied to a type.
4. **Unexported helper functions** last.

- **Group methods by receiver.** Never interleave methods of two types; a reader should find all of `*Service` in one place.
- **Main type first.** In `order.go` put `Order` before the small types it uses (`Item`, `Status`).
- **Call order within a group.** A method used by the one above it comes after it, so the file reads like a story.
- **Struct fields:** exported before unexported, related fields together (e.g. `mu` directly above the fields it guards). Reorder for memory only when profiling says so (see `go-memory`).

```go
package orders

const defaultCurrency = "EUR"

var ErrNotFound = errors.New("order not found")

type Service struct {
	repo Repository
}

type Repository interface {
	Find(ctx context.Context, id string) (Order, error)
}

func NewService(repo Repository) *Service { ... }

func (s *Service) Get(ctx context.Context, id string) (Order, error) { ... }

func (s *Service) applyDiscount(o *Order) { ... }

func roundCents(v int64) int64 { ... }
```

Example: `references/a6-a7-project-layout/orders/service.go`

## Context comes first

Any function that does I/O or waits (database, HTTP, channels) takes `ctx context.Context` as its first parameter and returns when it is done. Pass it down; never store it in a struct. Concurrency details are in the `go-concurrency` skill.

## Dependencies: stdlib first

Every third-party package is something you maintain forever. Use the standard library unless it doesn't cover the case or the package is clearly worth it (e.g. `net/http` routing with `GET /orders/{id}` patterns before reaching for a router; `log/slog` before a logging library).

## Project layout: group by feature

In Go a package is the directory, the namespace and the visibility boundary at once, so layer folders (`controllers/`, `services/`, `repositories/`) hide what the project does and force cross-package coupling.

- Top level reads like the domain: `orders/`, `discount/`, `inventory/`.
- Each feature owns its whole stack (routing, validation, logic, persistence) in its own directory.
- No cross imports between features. One feature talks to another through a small typed client the other exposes (`transport/client.go`) or a shared contract in `internal/`.
- `cmd/` is the only place features meet: it builds repositories, passes them to services, mounts routes.
- Test for this: *if this feature had to become its own service tomorrow, how many directories would you touch?* The answer should be one, plus a few lines in `cmd/`.

### Inside a feature: start with one file

Add files only when the code actually needs them:

| File | Holds |
|---|---|
| `order.go` | Types, domain rules, validation. Start here. |
| `service.go` | Use cases + interfaces for external dependencies (storage, payment API). No interface wrapping the service. |
| `transport/http.go` | HTTP encode/decode and handlers that call the service. |
| `transport/client.go` | Typed client exposing only what other features need. |
| `repository/postgres.go` | Storage implementation of the interface `service.go` declared. |
| `cmd/main.go` | Wiring. |

File names are examples; the split is what matters. Full working example: `references/a6-a7-project-layout/` (start at `cmd/main.go`, then `orders/service.go`).

## Tests: external test package

Test files declare `package orders_test`, not `package orders`. The test imports the package and uses only its exported API, exactly as a caller would, so tests break on behaviour changes, not refactors.

- If something seems to need an unexported test, test it through the exported function that uses it. If that's impossible, the code probably belongs in its own package.
- **Internal tests only when you really need them:** an internal test (`package orders`, no `_test`) is allowed only in special cases where internals must be tested directly and the exported API can't reach them. Put those tests in a separate file (e.g. `internal_test.go`) and keep everything else external.
- `package main` can't be imported, so its tests stay in `package main`. Keep `main` thin so there is little to test there.

## Review checklist

When reviewing Go code, flag:
- stuttering names, `Impl`/`Base`/`Default` suffixes, layer-named packages
- function bodies with no blank lines between logical steps, or a blank line between a call and its error check
- declarations out of order: methods of one type scattered or interleaved, helpers above the API, constructor away from its type
- functions doing I/O or waiting without `ctx context.Context` as the first parameter
- getters/setters with no logic, whole structs passed for one field, god objects
- interfaces declared by the producer, single-implementation interfaces, interface + struct in one file
- errors both logged and returned, capitalised or punctuated error messages, `%v` where `%w` was meant
- a new dependency the stdlib already covers
- features importing each other directly
- test files not in an external `_test` package, unless they're `package main` or really need to test internals
