# Go Developer Full Reference

Complete reference for Go 1.25+ development. This document covers all major topics in depth.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Modules and Dependencies](#modules-and-dependencies)
3. [Types and Interfaces](#types-and-interfaces)
4. [Error Handling](#error-handling)
5. [Generics](#generics)
6. [Concurrency](#concurrency)
7. [Configuration](#configuration)
8. [Logging](#logging)
9. [Embedding (go:embed)](#embedding-goembed)
10. [Build Constraints](#build-constraints)
11. [Tooling](#tooling)
12. [Common Patterns](#common-patterns)

---

## Project Structure

### Minimal main.go

```go
// cmd/api/main.go
package main

import (
    "context"
    "log/slog"
    "os"
    "os/signal"
    "syscall"

    "github.com/yourorg/yourproject/internal/app"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)

    if err := app.Run(ctx, logger); err != nil {
        slog.Error("application error", "err", err)
        os.Exit(1)
    }
}
```

---

## Modules and Dependencies

```bash
# Initialize a new module
go mod init github.com/yourorg/yourproject

# Add a dependency
go get github.com/some/package@v1.2.3

# Update all dependencies
go get -u ./...

# Tidy (remove unused, add missing)
go mod tidy

# Verify module integrity
go mod verify

# Vendor dependencies (optional)
go mod vendor
```

### go.mod Best Practices

```
module github.com/yourorg/yourproject

go 1.25

require (
    golang.org/x/sync v0.10.0
    github.com/jackc/pgx/v5 v5.7.0
    github.com/stretchr/testify v1.10.0
)

// Exclude a specific version with a known bug
exclude github.com/some/package v1.0.0

// Replace for local development
replace github.com/yourorg/shared => ../shared

// Ignore directories from package matching (1.25+)
ignore (
    ./vendor/legacy
)
```

### Multi-Module Workspaces (go work)

For monorepos or developing multiple modules together, use `go work` (Go 1.18+):

```bash
# Initialize a workspace
go work init ./api ./shared ./worker

# Add a module to the workspace
go work use ./newmodule

# Sync workspace dependencies
go work sync
```

```
// go.work
go 1.25

use (
    ./api
    ./shared
    ./worker
)
```

**Rules:**

- Use `go.work` for local multi-module development — never commit it for libraries (add to `.gitignore`)
- For deployable applications (monorepos), committing `go.work` is acceptable
- Each module still has its own `go.mod` with explicit dependency versions
- `go work sync` keeps `go.mod` files consistent with the workspace

---

## Types and Interfaces

### Struct Design

**Rule of thumb:** Use value receivers for small, immutable structs (up to ~3 fields of basic types). Use pointer receivers for larger structs, structs with reference fields (slices, maps, pointers), or when methods need to mutate the receiver.

```go
// ✅ Use value receivers for small structs, pointer receivers for large or mutable
type Point struct {
    X, Y float64
}

func (p Point) Distance(other Point) float64 {
    dx := p.X - other.X
    dy := p.Y - other.Y
    return math.Sqrt(dx*dx + dy*dy)
}

// ✅ Embed for composition
type TimestampedEntity struct {
    CreatedAt time.Time
    UpdatedAt time.Time
}

type User struct {
    TimestampedEntity
    ID    int64
    Name  string
    Email string
}

// ✅ Use struct tags for JSON/DB mapping
type APIUser struct {
    ID    int64  `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
}
```

### Interface Design

```go
// ✅ Small, focused interfaces
type Reader interface {
    Read(ctx context.Context, id int64) (*Entity, error)
}

type Writer interface {
    Write(ctx context.Context, entity *Entity) error
}

// ✅ Compose interfaces
type ReadWriter interface {
    Reader
    Writer
}

// ✅ Compile-time interface check
var _ UserRepository = (*PostgresUserRepo)(nil)

// ✅ Accept interfaces, return concrete types
func NewService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}
```

### Type Aliases and New Types

```go
// New type - distinct from underlying type
type UserID int64
type Email string

func (e Email) Validate() error {
    if !strings.Contains(string(e), "@") {
        return errors.New("invalid email format")
    }
    return nil
}

// Type alias - same type, different name
type Milliseconds = int64
```

---

## Error Handling

### Error Wrapping Chain

```go
// Layer 1: repository
func (r *PostgresRepo) FindByID(ctx context.Context, id int64) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx, "SELECT id, name FROM users WHERE id = $1", id).
        Scan(&user.ID, &user.Name)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("find user %d: %w", id, ErrNotFound)
    }
    if err != nil {
        return nil, fmt.Errorf("find user %d: %w", id, err)
    }
    return &user, nil
}

// Layer 2: service
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }
    return user, nil
}

// Layer 3: handler
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
    if err != nil {
        http.Error(w, "invalid user ID", http.StatusBadRequest)
        return
    }

    user, err := h.svc.GetUser(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "user not found", http.StatusNotFound)
            return
        }
        slog.Error("get user failed", "err", err, "id", id)
        http.Error(w, "internal server error", http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

### Multiple Error Returns

```go
// ✅ Join multiple errors (Go 1.20+)
func validateUser(u *User) error {
    var errs []error

    if u.Name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if u.Email == "" {
        errs = append(errs, errors.New("email is required"))
    }

    return errors.Join(errs...)
}
```

### Generic Error Inspection (1.26+)

Go 1.26 adds `errors.AsType[T]` for type-safe, generic error inspection without pre-declaring a variable. See [go-1.26-features.md](go-1.26-features.md#errorsastype---generic-error-inspection) for examples comparing `errors.AsType` with the pre-1.26 `errors.As` pattern.

---

## Generics

See [generics-guide.md](generics-guide.md) for generic functions, constraints, data structures, self-referential types (1.26), when NOT to use generics, composable iterator pipelines, and standard library iterator support (`slices.All`, `maps.Keys`, etc.).

---

## Concurrency

See [concurrency-guide.md](concurrency-guide.md) for goroutines, channels, sync primitives, worker pools, errgroup, context propagation (`AfterFunc`, `WithoutCancel`), and race detection.

---

## Configuration

```go
// ✅ Config struct loaded from environment
type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Log      LogConfig
}

type ServerConfig struct {
    Host         string        `env:"SERVER_HOST" envDefault:"0.0.0.0"`
    Port         int           `env:"SERVER_PORT" envDefault:"8080"`
    ReadTimeout  time.Duration `env:"SERVER_READ_TIMEOUT" envDefault:"10s"`
    WriteTimeout time.Duration `env:"SERVER_WRITE_TIMEOUT" envDefault:"30s"`
}

type DatabaseConfig struct {
    URL             string        `env:"DATABASE_URL,required"`
    MaxOpenConns    int           `env:"DATABASE_MAX_OPEN_CONNS" envDefault:"25"`
    ConnMaxLifetime time.Duration `env:"DATABASE_CONN_MAX_LIFETIME" envDefault:"5m"`
}

type LogConfig struct {
    Level string `env:"LOG_LEVEL" envDefault:"info"`
    JSON  bool   `env:"LOG_JSON" envDefault:"true"`
}

// Use github.com/caarlos0/env or similar for parsing
func loadConfig() (*Config, error) {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return &cfg, nil
}
```

---

## Logging

Use `log/slog` (standard library, Go 1.21+) for all logging. See [logging-guide.md](logging-guide.md) for handler setup (JSON/text), dependency injection, context-aware logging with middleware, error logging patterns, and custom handlers.

---

## Embedding (go:embed)

Embed static files (templates, configs, migrations, web assets) directly into the binary at compile time using `//go:embed`:

```go
import "embed"

// Embed a single file
//go:embed schema.sql
var schema string

// Embed a single file as bytes
//go:embed logo.png
var logo []byte

// Embed a directory tree
//go:embed templates/*
var templates embed.FS

// Embed multiple patterns
//go:embed static/* templates/* migrations/*.sql
var assets embed.FS
```

### Serving Embedded Files

```go
// ✅ Serve embedded static files over HTTP
//go:embed static/*
var staticFiles embed.FS

func newRouter() http.Handler {
    mux := http.NewServeMux()
    // Strip "static/" prefix so /css/style.css maps to static/css/style.css
    mux.Handle("GET /static/", http.StripPrefix("/static/",
        http.FileServerFS(staticFiles)))
    return mux
}
```

### Reading Embedded Files

```go
// ✅ Read files from an embedded filesystem
//go:embed migrations/*.sql
var migrations embed.FS

func runMigrations(db *sql.DB) error {
    entries, err := migrations.ReadDir("migrations")
    if err != nil {
        return fmt.Errorf("read migrations: %w", err)
    }

    sort.Slice(entries, func(i, j int) bool {
        return entries[i].Name() < entries[j].Name()
    })

    for _, entry := range entries {
        data, err := migrations.ReadFile("migrations/" + entry.Name())
        if err != nil {
            return fmt.Errorf("read %s: %w", entry.Name(), err)
        }
        if _, err := db.Exec(string(data)); err != nil {
            return fmt.Errorf("exec %s: %w", entry.Name(), err)
        }
    }
    return nil
}
```

**Rules:**

- `//go:embed` directives must be on the line immediately before the variable declaration
- Only works with package-level variables (not inside functions)
- Use `string` for text files, `[]byte` for binary files, `embed.FS` for directories
- Patterns use `path.Match` syntax — `*` matches within a directory, `**` is not supported
- Hidden files (starting with `.` or `_`) are excluded by default; use `all:` prefix to include them: `//go:embed all:templates`

---

## Build Constraints

Use `//go:build` lines (Go 1.17+) to control which files are included in a build:

```go
// Platform-specific code
//go:build linux
package mypackage

// Multiple platforms
//go:build linux || darwin
package mypackage

// Exclude a platform
//go:build !windows
package mypackage

// Custom build tags (e.g., integration tests)
//go:build integration
package store_test

// Combine tags with AND
//go:build integration && linux
package store_test
```

```bash
# Build with custom tags
go build -tags=integration ./...
go test -tags=integration ./...

# Multiple tags
go test -tags='integration e2e' ./...
```

**Rules:**

- Use `//go:build` (not the legacy `// +build` syntax)
- Place the `//go:build` line before the `package` declaration, separated by a blank line
- Use for: platform-specific code, integration test separation, feature flags, optional dependencies
- Prefer `t.Skip()` with env vars for simple test gating; use build tags for larger separations

---

## Tooling

### Makefile

```makefile
.PHONY: build test test-ci lint vet tidy vuln coverage

build:
	go build ./cmd/...

test:
	go test -race -cover ./...

test-ci:
	go test -race -cover -json ./... > test-results.json

lint:
	golangci-lint run ./...

vet:
	go vet ./...

tidy:
	go mod tidy

vuln:
	govulncheck ./...

coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html
	go tool cover -func=coverage.out

check: vet lint test vuln

generate:
	go generate ./...
```

### golangci-lint

```bash
# Install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run
golangci-lint run ./...

# Run with specific linters
golangci-lint run --enable=gosec,errcheck ./...

# Auto-fix where possible
golangci-lint run --fix ./...
```

### govulncheck

```bash
# Install
go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan source
govulncheck ./...

# Scan binary
govulncheck -mode=binary ./bin/myapp
```

### go generate

```go
//go:generate stringer -type=Status
type Status int

const (
    StatusPending Status = iota
    StatusActive
    StatusInactive
)

//go:generate mockgen -source=user_repo.go -destination=mock_user_repo.go
```

### Profiling with pprof

Go has built-in profiling support via `runtime/pprof` and `net/http/pprof`:

```go
// ✅ Enable pprof HTTP endpoints (development/staging)
import _ "net/http/pprof"

func main() {
    // Registers /debug/pprof/* endpoints on the default mux
    go http.ListenAndServe(":6060", nil)
    // ... your application ...
}
```

```bash
# CPU profile: capture 30 seconds of CPU usage
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap (memory) profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile (find goroutine leaks)
go tool pprof http://localhost:6060/debug/pprof/goroutine

# View as flame graph in browser (1.26+ default view)
go tool pprof -http=:8080 cpu.prof

# Common pprof interactive commands
#   top       - show top functions by CPU/memory
#   list fn   - show annotated source for function
#   web       - open call graph in browser
#   svg       - export call graph as SVG
```

```go
// ✅ Programmatic CPU profiling (for benchmarks or CLI tools)
import "runtime/pprof"

func profileCPU(filename string) (stop func()) {
    f, err := os.Create(filename)
    if err != nil {
        log.Fatal(err)
    }
    pprof.StartCPUProfile(f)
    return func() {
        pprof.StopCPUProfile()
        f.Close()
    }
}

// Usage:
stop := profileCPU("cpu.prof")
defer stop()
// ... code to profile ...
```

**Rules:**

- NEVER expose pprof endpoints in production without authentication
- Always benchmark first (`go test -bench`) before resorting to pprof
- Use `-http` flag with `go tool pprof` for interactive flame graph exploration
- Profile in conditions that resemble production (realistic data, load)
- Use `runtime/trace` for latency analysis; use `pprof` for CPU/memory hotspots

---

## Common Patterns

### Options Pattern

```go
type ServerOption func(*Server)

func WithTimeout(d time.Duration) ServerOption {
    return func(s *Server) {
        s.timeout = d
    }
}

func WithLogger(logger *slog.Logger) ServerOption {
    return func(s *Server) {
        s.logger = logger
    }
}

func NewServer(addr string, opts ...ServerOption) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second,
        logger:  slog.Default(),
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### Retry with Backoff

```go
func retry(ctx context.Context, maxAttempts int, fn func() error) error {
    var err error
    for attempt := range maxAttempts {
        if err = fn(); err == nil {
            return nil
        }

        if attempt == maxAttempts-1 {
            break
        }

        backoff := time.Duration(1<<attempt) * 100 * time.Millisecond
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff):
        }
    }
    return fmt.Errorf("after %d attempts: %w", maxAttempts, err)
}
```

### Graceful Shutdown

```go
func run(ctx context.Context, logger *slog.Logger) error {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: newRouter(),
    }

    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        logger.Info("starting server", "addr", srv.Addr)
        if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            return fmt.Errorf("listen: %w", err)
        }
        return nil
    })

    g.Go(func() error {
        <-ctx.Done()
        logger.Info("shutting down server")
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        return srv.Shutdown(shutdownCtx)
    })

    return g.Wait()
}
```
