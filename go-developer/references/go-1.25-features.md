# Go 1.25 Features Reference

Released August 12, 2025. Key improvements in runtime, tooling, and standard library.

## Runtime

### Container-Aware GOMAXPROCS

On Linux, the runtime now automatically adjusts `GOMAXPROCS` based on cgroup CPU bandwidth limits. This is critical for Kubernetes and containerized deployments.

```go
// Before 1.25: GOMAXPROCS defaulted to host CPU count, causing over-scheduling
// After 1.25: automatically respects container CPU limits

// You can still override manually if needed:
runtime.GOMAXPROCS(4)

// Or check the current value:
procs := runtime.GOMAXPROCS(0)
```

**Impact:** Prevents goroutine over-scheduling in containers, leading to more predictable latency and better CPU utilization.

### Experimental Green Tea Garbage Collector

A new GC designed for workloads with many small objects. Reduces GC overhead by 10–40% by scanning entire memory spans rather than individual objects.

```bash
# Enable experimentally in 1.25
GOEXPERIMENT=greenteagc go run ./...

# Note: Enabled by default in Go 1.26
```

### Safer Nil-Pointer Handling

A bug from Go 1.21 that sometimes prevented nil pointer panics from triggering has been fixed. Dereferencing a nil pointer now reliably causes a panic.

---

## Language

### Loop Variable Scoping (Finalized)

Each loop iteration now creates a new instance of the loop variable. This resolves a classic bug with goroutines and closures capturing loop variables.

```go
// ✅ Now safe in 1.25+ - each goroutine captures its own `i`
for i := range 10 {
    go func() {
        fmt.Println(i) // Correct: prints 0-9 in some order
    }()
}

// Before 1.22, you needed:
for i := range 10 {
    i := i // shadow to capture correctly
    go func() {
        fmt.Println(i)
    }()
}
```

### Removal of "Core Types" Concept

The abstract notion of "core types" has been removed from the language specification. Type rules are now defined directly through prose and type sets. This simplifies the spec without affecting existing behavior - no code changes needed.

---

## Standard Library

### `sync.WaitGroup.Go()` (New)

A cleaner way to spawn goroutines tracked by a `WaitGroup`:

```go
var wg sync.WaitGroup

// ✅ New in 1.25 - no need to call Add(1) separately
wg.Go(func() {
    processItem(item)
})

wg.Go(func() {
    processOtherItem(otherItem)
})

wg.Wait()

// Before 1.25:
wg.Add(1)
go func() {
    defer wg.Done()
    processItem(item)
}()
```

### `testing/synctest` (Stable)

Graduated from experimental to stable. Provides tools for testing concurrent code in an isolated "bubble" with virtualized time.

```go
func TestConcurrentOperation(t *testing.T) {
    synctest.Run(func() {
        // Code runs in an isolated bubble
        // time.Sleep and timers use virtual time
        ch := make(chan int)

        go func() {
            time.Sleep(1 * time.Second) // Virtual - doesn't actually wait
            ch <- 42
        }()

        synctest.Wait() // Advance virtual time until goroutines block

        result := <-ch
        if result != 42 {
            t.Errorf("got %d, want 42", result)
        }
    })
}
```

### New `slices` and `maps` Utilities

```go
import (
    "slices"
    "maps"
)

// slices additions
sorted := slices.SortedFunc(iter, cmp.Compare)
clipped := slices.Clip(s)  // Remove excess capacity

// maps additions
dst := make(map[string]int)
maps.Copy(dst, src)  // Copy all entries from src to dst
```

### Experimental `encoding/json/v2`

A rewritten JSON package with substantially faster decoding and near-zero heap allocation:

```bash
# Enable experimentally
GOEXPERIMENT=jsonv2 go run ./...
```

```go
import "encoding/json/v2"

// Faster decoding, streaming support, composable options
data, err := json.Marshal(v)
err = json.Unmarshal(data, &v)
```

**Note:** API is not stable yet. Use in new projects for experimentation only.

---

## Tooling

### `go doc -http`

Start a local documentation server and open it in a browser:

```bash
go doc -http=:6060
# Opens http://localhost:6060 in your browser
```

### `go.mod ignore` Directive

Exclude directories from package matching:

```
module github.com/yourorg/yourproject

go 1.25

ignore (
    ./vendor/legacy-tool
    ./scripts/generated
)
```

### DWARF 5 Debug Information

The compiler now generates DWARF 5 debug info by default, reducing binary size and linking time.

```bash
# Disable if needed (e.g., for older debuggers)
GOEXPERIMENT=nodwarf5 go build ./...
```

### Flight Recorder API

New tracing API for collecting execution data without stopping the program:

```go
import "runtime/trace"

// Start a flight recorder
rec := trace.NewFlightRecorder()
rec.Start()

// ... program runs ...

// Capture a snapshot
var buf bytes.Buffer
rec.WriteTo(&buf)
```

### Build with AddressSanitizer

`go build -asan` now detects memory leaks from C-allocated memory at program exit:

```bash
go build -asan ./...
go test -asan ./...
```

---

## Performance

- **Register-based calling convention** (AMD64): 5–10% improvement in CPU-bound workloads
- **SSA optimizations**: 3–7% reduction in p95 latency reported in some workloads
- **Stack-allocated slices**: Compiler allocates more slices on the stack, reducing GC pressure
- **Stricter module checksum enforcement**: Faster failure detection for corrupted modules
