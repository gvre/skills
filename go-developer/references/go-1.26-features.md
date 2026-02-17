# Go 1.26 Features Reference

Released February 10, 2026. Significant language changes, GC improvements, and new packages.

## Language Changes

### `new(expr)` - Initialize Pointer with Value

The built-in `new` function now accepts an expression, allowing direct initialization:

```go
// ✅ Go 1.26: initialize a pointer with a value in one step
ptr := new(int64(300))
// *ptr == 300

name := new(string("Alice"))
// *name == "Alice"

// Particularly useful for optional struct fields (JSON, protobuf)
type Config struct {
    Timeout *int64
    Name    *string
}

cfg := Config{
    Timeout: new(int64(30)),
    Name:    new(string("default")),
}

// Before 1.26, you needed a helper or two lines:
timeout := int64(30)
cfg.Timeout = &timeout
```

### Self-Referential Generic Types

Generic types can now refer to themselves in their own type parameter list, enabling recursive type constraints:

```go
// ✅ Go 1.26: self-referential generic types
type Ordered[T Ordered[T]] interface {
    Less(other T) bool
}

type TreeNode[T Ordered[T]] struct {
    Value T
    Left  *TreeNode[T]
    Right *TreeNode[T]
}
```

See [generics-guide.md](generics-guide.md#self-referential-generic-types-126) for the full implementation with `Insert`, concrete type satisfaction, and usage.

---

## Performance

### Green Tea GC (Now Default)

The experimental Green Tea garbage collector from 1.25 is now enabled by default. Expected improvements:

- **10–40% reduction** in GC overhead for real-world programs
- Better locality when marking and scanning small objects
- Improved CPU scalability

```bash
# No action needed - enabled by default in 1.26
# To opt out (not recommended):
GOEXPERIMENT=nogreenteagc go run ./...
```

### Reduced cgo Overhead

Baseline `cgo` call overhead reduced by approximately 30%:

```go
// cgo calls are now significantly cheaper
// Beneficial for systems using SQLite, system APIs, or native libraries
import "C"

//export MyGoFunction
func MyGoFunction() C.int {
    return 42
}
```

### Faster Memory Allocation

- Specialized allocation functions for small objects (1–512 bytes)
- `io.ReadAll` is significantly faster without code changes
- Compiler allocates backing store for slices on the stack in more situations

---

## Standard Library

### `errors.AsType` - Generic Error Inspection

Type-safe, generic error inspection without type assertions:

```go
import "errors"

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// ✅ Go 1.26: errors.AsType for generic, type-safe inspection
if valErr, ok := errors.AsType[*ValidationError](err); ok {
    fmt.Printf("Field %q failed: %s\n", valErr.Field, valErr.Message)
}

// Before 1.26:
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("Field %q failed: %s\n", valErr.Field, valErr.Message)
}
```

### Experimental: `simd/archsimd`

Access to SIMD operations for `amd64` (experimental):

```go
//go:build amd64

import "simd/archsimd"

// Low-level SIMD operations for performance-critical code
// Use only when you've benchmarked and confirmed the need
```

### Experimental: `runtime/secret`

Securely erase sensitive data after use:

```go
import "runtime/secret"

// Ensure sensitive data is zeroed when no longer needed
key := make([]byte, 32)
// ... use key for crypto ...
secret.Erase(key) // Securely zero the memory
```

### Experimental: Goroutine Leak Profiler

Detect leaked goroutines blocked on unreachable concurrency primitives:

```go
import "runtime/pprof"

// The goroutineleak profile reports leaked goroutines
// Will be enabled by default in Go 1.27

// Access via pprof HTTP endpoint:
// GET /debug/pprof/goroutineleak
```

---

## Tooling

### Rewritten `go fix` with Modernizers

`go fix` now uses the Go analysis framework and includes "modernizers" that suggest safe, automated upgrades:

```bash
# Apply all available modernizations
go fix ./...

# Examples of what go fix now handles:
# - Replace deprecated API calls with modern equivalents
# - Update to use new 1.26 language features where safe
# - Suggest errors.AsType over errors.As where applicable
```

### `pprof` Flame Graphs by Default

The `pprof` web UI now defaults to flame graphs for easier performance analysis:

```bash
go tool pprof -http=:8080 cpu.prof
# Opens flame graph view by default
```

### `go mod init` Lower Default Version

`go mod init` now defaults to a lower Go version in new `go.mod` files to promote broader module compatibility. Explicitly set your minimum version:

```bash
go mod init github.com/yourorg/yourproject
# go.mod will have a conservative go version

# Update to your actual minimum:
go mod edit -go=1.26
```

### Unified `go doc`

The `go doc` command has been simplified and unified for easier documentation discovery:

```bash
go doc fmt.Println
go doc -http=:6060  # Local doc server (from 1.25)
```

---

## Migration Notes

### Upgrading from 1.25 to 1.26

1. **Green Tea GC is now default** - test your application's memory behavior; most apps will see improvement
2. **`go fix` is more powerful** - run it to get automated modernization suggestions
3. **`new(expr)` syntax** - optional to adopt, but reduces boilerplate for pointer initialization
4. **Self-referential generics** - enables new patterns for recursive data structures

```bash
# Upgrade your go.mod
go mod edit -go=1.26

# Run modernizer
go fix ./...

# Verify nothing broke
go test -race ./...
govulncheck ./...
```
