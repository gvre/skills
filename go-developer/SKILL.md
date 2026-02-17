---
name: go-developer
description:
  Comprehensive rules and best practices for writing modern Go 1.25+ projects.
  Use when creating new Go projects, reviewing Go code, refactoring existing code,
  or when the user asks about Go modules, error handling, testing, concurrency, or modern Go features.
  Ensures code follows current best practices with proper tooling, interfaces,
  generics, and goroutines. Includes version-specific features for 1.25 and 1.26.
disable-model-invocation: true
license: MIT
metadata:
  author: Giannis Vrentzos
  version: "1.0.0"
---

# Go Developer

Guide for writing production-quality Go 1.25+ projects following current best practices.

> **Structure:** This file contains rules, guidelines, and compact examples. For full code examples and deep dives, see the files in `references/`. Consult them when you need implementation details beyond what's shown here.

## Version Support

This guide covers:

- **Go 1.25+**: Base requirements and common features (all sections below)
- **Go 1.25**: Runtime, tooling, and stdlib improvements → See [references/go-1.25-features.md](references/go-1.25-features.md)
- **Go 1.26**: Language changes, GC improvements, new packages → See [references/go-1.26-features.md](references/go-1.26-features.md)

### Feature Version Matrix

| Feature | 1.25 | 1.26 |
|---------|------|------|
| Container-aware GOMAXPROCS | ✅ | ✅ |
| `sync.WaitGroup.Go()` | ✅ | ✅ |
| `testing/synctest` (stable) | ✅ | ✅ |
| `go doc -http=:6060` | ✅ | ✅ |
| `go.mod ignore` directive | ✅ | ✅ |
| Green Tea GC (default) | ❌ (experimental) | ✅ |
| `new(expr)` initialization | ❌ | ✅ |
| Self-referential generic types | ❌ | ✅ |
| `errors.AsType` | ❌ | ✅ |
| Rewritten `go fix` with modernizers | ❌ | ✅ |
| `pprof` flame graphs by default | ❌ | ✅ |

## Quick Reference

**Core Requirements (1.25+):**

- Go 1.25+ minimum
- Use `go.mod` for all dependency management
- `golangci-lint` for linting
- `govulncheck` for vulnerability scanning
- Table-driven tests with `t.Run`
- Interfaces for dependency injection
- Explicit error handling (no panics for expected errors)

## Module Configuration (go.mod)

```
module github.com/yourorg/yourproject

go 1.25

require (
    github.com/some/dependency v1.2.3
)
```

**Rules:**

- Pin the minimum Go version you actually require
- Run `go mod tidy` regularly to keep go.sum clean
- Use `go mod verify` to check module integrity
- Use `go.mod ignore` directive (1.25+) to exclude directories from package matching
- Use `go work` for multi-module development (monorepos); don't commit `go.work` for libraries

## Interfaces and Dependency Injection

**Define interfaces where they are used (consumer side), not where they are implemented.**

**Rules:**

- Keep interfaces small (1–3 methods) - prefer composition over fat interfaces
- Accept interfaces, return concrete types
- Use `any` instead of `interface{}`
- Verify interface satisfaction at compile time: `var _ UserRepository = (*PostgresRepo)(nil)`

## Error Handling

**Return errors explicitly - never use panics for expected failures:**

- Wrap errors with context
- Sentinel errors for known conditions
- Custom error types for structured context

Inspect error chains with `errors.Is` (sentinels), `errors.As` (typed), or `errors.AsType[T]` (1.26, generic).
See [references/REFERENCE.md](references/REFERENCE.md) for a full 3-layer error wrapping example (repo → service → handler).

**Rules:**

- ALWAYS wrap errors with `fmt.Errorf("context: %w", err)`
- NEVER ignore errors (use `_` only when truly intentional, add a comment)
- Use `errors.Is` for sentinel errors, `errors.As` for typed errors
- Define sentinel errors at the package level with `var Err... = errors.New(...)`
- Panic only for programmer errors (nil pointer dereference, index out of bounds in init)

## Generics (1.18+) and Iterators (1.23+)

**Rules:**

- Use generics for reusable data structures and algorithms, not for everything
- Prefer interfaces when you need runtime polymorphism (method dispatch)
- Use `comparable` constraint for map keys and equality checks
- Use `cmp.Ordered` for types that support `<`, `>`, `<=`, `>=`
- Don't over-engineer: if a function works fine with `any` or a concrete type, don't add generics
- Use `iter.Seq[T]` for single-value iterators, `iter.Seq2[K, V]` for key-value pairs
- Always check `yield`'s return value and stop if it returns `false`
- Use `slices.Collect`, `maps.Collect` to materialize iterators into concrete types
- Prefer iterators over returning `[]T` when the caller may not need all elements (lazy evaluation)

See [references/generics-guide.md](references/generics-guide.md) for generic functions, constraints, data structures, self-referential types (1.26), when NOT to use generics, and composable iterator pipelines.

## Concurrency

### Goroutines and Channels

```go
// ✅ Always handle goroutine lifecycle
func (s *Server) Start(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return s.httpServer.ListenAndServe()
    })

    g.Go(func() error {
        <-ctx.Done()
        return s.httpServer.Shutdown(context.Background())
    })

    return g.Wait()
}

// ✅ Use sync.WaitGroup.Go() (1.25+) for fire-and-forget goroutines
var wg sync.WaitGroup
wg.Go(func() {
    processItem(item)
})
wg.Wait()
```

### Context Propagation

```go
// ✅ Always pass context as first parameter
func (s *UserService) GetUser(ctx context.Context, id int64) (*app.User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, fmt.Sprintf("/users/%d", id), nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }
    // ...
}
```

**Rules:**

- ALWAYS pass `context.Context` as the first parameter
- NEVER store context in a struct
- ALWAYS call `cancel()` when using `context.WithCancel` or `context.WithTimeout`
- Use `context.AfterFunc` (1.21+) to register cleanup callbacks on context cancellation instead of spawning goroutines that block on `<-ctx.Done()`
- Use `context.WithoutCancel` (1.21+) for background work that should outlive the parent context (carries values but not cancellation)
- Use `errgroup` (golang.org/x/sync/errgroup) for concurrent tasks with error propagation
- Use `-race` flag in tests to detect data races

**Advanced patterns:** See [references/concurrency-guide.md](references/concurrency-guide.md) for channels, mutexes, worker pools, context.WithoutCancel, and sync primitives.

## Testing

**Key patterns:**

- Use the standard `testing` package (no external test framework required)
- Table-driven tests with `t.Run` for multiple scenarios
- Use `testify` for assertions when it reduces boilerplate
- Mock via interfaces - no magic mocking frameworks needed
- Use `-race` flag to detect data races
- Use `testing/synctest` (stable in 1.25) for concurrent code testing

See [references/testing-guide.md](references/testing-guide.md) for comprehensive testing patterns, mocking, benchmarks, and fuzzing.

## Logging (slog)

**Use `log/slog` (standard library, Go 1.21+).**

**Rules:**

- Use `log/slog` — not `fmt.Println`, not `log.Printf`
- Use structured key-value pairs, not string formatting
- Pass logger via dependency injection, not as a global; use `slog.SetDefault` only at startup
- Log at boundaries (handlers/entrypoints) — lower layers wrap and return errors, they don't log
- Use `*Context` methods (`InfoContext`, `ErrorContext`) to propagate request-scoped data
- NEVER log sensitive data (passwords, tokens, PII)

See [references/logging-guide.md](references/logging-guide.md) for setup, handlers, middleware, context-aware logging, and custom handlers.

## Code Quality

### golangci-lint Configuration

**Note:** The `version: "2"` format below requires golangci-lint v2+. If your team uses golangci-lint v1, remove the `version` field and adjust the config accordingly.

```yaml
# .golangci.yml
version: "2"

linters:
  enable:
    - errcheck       # Check for unchecked errors
    - govet          # Reports suspicious constructs
    - staticcheck    # Comprehensive static analysis
    - gosec          # Security checks
    - revive         # Opinionated linter
    - goimports      # Import formatting
    - misspell       # Spell checker
    - unconvert      # Remove unnecessary type conversions
    - unparam        # Unused function parameters
    - noctx          # Detect HTTP requests without context

linters-settings:
  revive:
    rules:
      - name: exported
      - name: var-naming
  govet:
    enable-all: true

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
```

### Naming Conventions

- **Packages:** short, lowercase, no underscores: `user`, `httputil`, `store`
- **Functions/variables:** camelCase: `getUserByID`, `maxRetries`
- **Exported:** PascalCase: `UserService`, `ErrNotFound`
- **Interfaces:** noun or `-er` suffix: `Reader`, `Stringer`
- **Constants:** camelCase (unexported) or PascalCase (exported)
- **Acronyms:** keep consistent casing: `userID`, `parseURL`, `HTTPClient`

✅ GOOD: `getUserByID()`, `ErrNotFound`,
❌ BAD: `get_user_by_id()`, `err_not_found`

## Security Best Practices

**Critical rules:**

- ✅ ALWAYS validate and sanitize user input
- ✅ ALWAYS use parameterized queries for SQL
- ✅ ALWAYS load secrets from environment or secret manager
- ✅ ALWAYS use HTTPS for external APIs
- ✅ Run `govulncheck ./...` regularly
- ❌ NEVER hardcode secrets or API keys
- ❌ NEVER use `fmt.Sprintf` to build SQL queries
- ❌ NEVER log sensitive data

## Anti-Patterns (Never Do)

- ❌ Panic for expected errors - use error returns
- ❌ Ignore errors: `result, _ := doSomething()`
- ❌ Store context in a struct field
- ❌ Use `init()` for complex initialization
- ❌ Global mutable state (use dependency injection)
- ❌ Fat interfaces (>5 methods) - split them
- ❌ `interface{}` / `any` when a concrete type or generic works
- ❌ Goroutine leaks - always ensure goroutines can exit
- ❌ Unbuffered channels in hot paths without careful design
- ❌ `time.Sleep` in tests - use `testing/synctest` or channels

## Go 1.25 and 1.26 Features

- **Go 1.25**: See [references/go-1.25-features.md](references/go-1.25-features.md)
- **Go 1.26**: See [references/go-1.26-features.md](references/go-1.26-features.md)

## Checklist

### For Code Authors

When creating Go code:

- [ ] Go 1.25+ features used where appropriate
- [ ] All errors handled explicitly (no `_` without comment)
- [ ] Errors wrapped with `fmt.Errorf("context: %w", err)`
- [ ] `context.Context` passed as first parameter
- [ ] `cancel()` called for every `WithCancel`/`WithTimeout`
- [ ] Interfaces defined at consumer side, kept small
- [ ] Table-driven tests with `t.Run`
- [ ] `-race` flag used in CI
- [ ] `golangci-lint` passes
- [ ] `govulncheck` passes
- [ ] Structured logging with `slog`
- [ ] No hardcoded secrets
- [ ] `internal/` used for private code

### For Code Reviewers

When reviewing Go code, prioritize findings:

**Priority 1 - Critical Issues (Must Fix):**

1. **Security vulnerabilities** (SQL injection, hardcoded secrets, path traversal)
2. **Ignored errors** (silent failures, missing error checks)
3. **Goroutine leaks** (goroutines that can never exit)
4. **Data races** (shared state without synchronization)

**Priority 2 - Important Improvements (Should Fix):**

1. **Missing context propagation** (functions missing `ctx context.Context`)
2. **Fat interfaces** (interfaces with too many methods)
3. **Error wrapping** (errors without context)
4. **Testing gaps** (missing table-driven tests, no race detection)

**Priority 3 - Nice to Have (Consider Fixing):**

1. **Naming conventions** (non-idiomatic names)
2. **Package organization** (generic package names)
3. **Performance** (unnecessary allocations, missing benchmarks)

## Resources

- **Full reference:** See [references/REFERENCE.md](references/REFERENCE.md)
- **Generics & iterators guide:** See [references/generics-guide.md](references/generics-guide.md)
- **Concurrency guide:** See [references/concurrency-guide.md](references/concurrency-guide.md)
- **Testing guide:** See [references/testing-guide.md](references/testing-guide.md)
- **Logging guide:** See [references/logging-guide.md](references/logging-guide.md)
- **Go 1.25 features:** See [references/go-1.25-features.md](references/go-1.25-features.md)
- **Go 1.26 features:** See [references/go-1.26-features.md](references/go-1.26-features.md)

The full reference also covers: [embedding (go:embed)](references/REFERENCE.md), [build constraints](references/REFERENCE.md), and [profiling with pprof](references/REFERENCE.md).
