# Concurrency Guide Reference

Comprehensive guide to Go concurrency patterns: goroutines, channels, sync primitives, worker pools, and context propagation.

## Goroutines

### Basic Goroutine Lifecycle

Always ensure goroutines can exit. Goroutine leaks are a common source of memory and resource issues.

```go
// ✅ Goroutine with cancellation via context
func processInBackground(ctx context.Context, items <-chan Item) {
    for {
        select {
        case <-ctx.Done():
            return // Clean exit
        case item, ok := <-items:
            if !ok {
                return // Channel closed
            }
            process(item)
        }
    }
}

// ✅ Goroutine with done channel
func startWorker(done <-chan struct{}) {
    for {
        select {
        case <-done:
            return
        default:
            doWork()
        }
    }
}
```

### sync.WaitGroup (1.25+: `.Go()` method)

```go
var wg sync.WaitGroup

// ✅ Go 1.25+: use wg.Go() - no separate Add/Done needed
for _, item := range items {
    wg.Go(func() {
        process(item)
    })
}
wg.Wait()

// Pre-1.25 pattern:
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()
```

---

## errgroup - Concurrent Tasks with Error Propagation

`golang.org/x/sync/errgroup` is the standard way to run concurrent tasks and collect errors:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // At most 10 goroutines run concurrently
    results := make([]string, len(urls))

    for i, url := range urls {
        g.Go(func() error {
            body, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            results[i] = body
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

---

## Channels

### Buffered vs Unbuffered

```go
// Unbuffered: sender blocks until receiver is ready
ch := make(chan int)

// Buffered: sender blocks only when buffer is full
ch := make(chan int, 100)

// ✅ Use buffered channels for known batch sizes
results := make(chan Result, len(items))
for _, item := range items {
    go func(item Item) {
        results <- process(item)
    }(item)
}

for range items {
    result := <-results
    // handle result
}
```

### Channel Direction

```go
// ✅ Use directional channels in function signatures
func producer(out chan<- int) {  // send-only
    out <- 42
}

func consumer(in <-chan int) {   // receive-only
    val := <-in
    fmt.Println(val)
}
```

### Select with Timeout

```go
// ✅ Always add a timeout or context check
func waitForResult(ctx context.Context, ch <-chan Result) (Result, error) {
    select {
    case result := <-ch:
        return result, nil
    case <-ctx.Done():
        return Result{}, ctx.Err()
    }
}
```

### Fan-Out / Fan-In

```go
// Fan-out: distribute work to multiple workers
func fanOut(in <-chan Job, workers int) []<-chan Result {
    channels := make([]<-chan Result, workers)
    for i := range workers {
        ch := make(chan Result)
        channels[i] = ch
        go func() {
            defer close(ch)
            for job := range in {
                ch <- process(job)
            }
        }()
    }
    return channels
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Go(func() {
            for result := range ch {
                out <- result
            }
        })
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

---

## Worker Pool

```go
type WorkerPool struct {
    jobs    chan Job
    results chan Result
    wg      sync.WaitGroup
}

func NewWorkerPool(workers int) *WorkerPool {
    p := &WorkerPool{
        jobs:    make(chan Job, workers*2),
        results: make(chan Result, workers*2),
    }

    for range workers {
        p.wg.Go(func() {
            for job := range p.jobs {
                p.results <- process(job)
            }
        })
    }

    return p
}

func (p *WorkerPool) Submit(job Job) {
    p.jobs <- job
}

func (p *WorkerPool) Close() {
    close(p.jobs)
    p.wg.Wait()
    close(p.results)
}

func (p *WorkerPool) Results() <-chan Result {
    return p.results
}
```

---

## Mutexes and sync Primitives

### sync.Mutex

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### sync.RWMutex - Read-Heavy Workloads

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // Multiple readers allowed
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // Exclusive write lock
    defer c.mu.Unlock()
    c.items[key] = value
}
```

### sync.Once - One-Time Initialization

```go
type Singleton struct {
    once     sync.Once
    instance *Service
}

func (s *Singleton) Get() *Service {
    s.once.Do(func() {
        s.instance = &Service{}
    })
    return s.instance
}
```

### sync.Map - Concurrent Map

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
if val, ok := m.Load("key"); ok {
    fmt.Println(val.(string))
}

// LoadOrStore
actual, loaded := m.LoadOrStore("key", "default")

// Range
m.Range(func(key, value any) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true // continue iteration
})
```

### atomic - Lock-Free Primitives

```go
import "sync/atomic"

var counter atomic.Int64

// Increment atomically
counter.Add(1)

// Read atomically
val := counter.Load()

// Compare and swap
swapped := counter.CompareAndSwap(old, new)
```

---

## Context Propagation

### Rules

```go
// ✅ Context as first parameter, always
func DoWork(ctx context.Context, id int) error { ... }

// ✅ Propagate context to all downstream calls
func (s *Service) Process(ctx context.Context, req Request) error {
    user, err := s.repo.FindUser(ctx, req.UserID)    // pass ctx
    if err != nil {
        return err
    }
    return s.notifier.Send(ctx, user, req.Message)   // pass ctx
}

// ✅ Set timeouts at call boundaries
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    result, err := h.service.Process(ctx, parseRequest(r))
    // ...
}

// ❌ NEVER store context in a struct
type Bad struct {
    ctx context.Context // Don't do this
}
```

### context.AfterFunc (1.21+)

Register a callback that runs when a context is done (cancelled or timed out). Useful for cleanup without blocking goroutines on `<-ctx.Done()`:

```go
// ✅ Clean up resources when context is cancelled
func watchResource(ctx context.Context, conn *Connection) {
    stop := context.AfterFunc(ctx, func() {
        // Runs in its own goroutine when ctx is done
        conn.Close()
    })

    // If you no longer need the callback (e.g., finished normally):
    defer stop() // Returns true if the callback was stopped before running
}

// ✅ Combine multiple contexts: cancel when ANY parent is done
func mergeContexts(ctx1, ctx2 context.Context) (context.Context, context.CancelFunc) {
    ctx, cancel := context.WithCancel(ctx1)
    stop := context.AfterFunc(ctx2, cancel)
    return ctx, func() {
        stop()
        cancel()
    }
}
```

### context.WithoutCancel (1.21+)

Create a derived context that is never cancelled, even when the parent is. Useful for background work that should outlive the request:

```go
// ✅ Spawn background work that survives request cancellation
func (h *Handler) SubmitReport(w http.ResponseWriter, r *http.Request) {
    report := parseReport(r)

    // This context won't be cancelled when the HTTP request ends
    bgCtx := context.WithoutCancel(r.Context())
    go func() {
        // Still carries request-scoped values (request ID, etc.)
        // but won't be cancelled when the client disconnects
        if err := h.svc.GenerateReport(bgCtx, report); err != nil {
            slog.ErrorContext(bgCtx, "background report failed", "err", err)
        }
    }()

    w.WriteHeader(http.StatusAccepted)
}
```

**Rules:**

- Use `context.WithoutCancel` only when the background work genuinely should outlive the parent
- The returned context still carries all values from the parent (request IDs, loggers)
- The returned context has no deadline and is never cancelled
- Don't use this to "fix" cancellation bugs — if work should be cancelled with the request, pass the original context

### Context Values

```go
// ✅ Use typed keys to avoid collisions
type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userIDKey    contextKey = "userID"
)

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```

---

## Race Detection

Always run tests with `-race` in CI:

```bash
go test -race ./...
go build -race ./...  # For race detection in production debugging
```

```go
// ✅ Use testing/synctest for concurrent test isolation (1.25+)
import "testing/synctest"

func TestConcurrentAccess(t *testing.T) {
    synctest.Run(func() {
        cache := NewCache()

        var wg sync.WaitGroup
        for range 100 {
            wg.Go(func() {
                cache.Set("key", "value")
                cache.Get("key")
            })
        }
        wg.Wait()
    })
}
```

---

## Common Pitfalls

```go
// ❌ Goroutine leak: goroutine blocks forever if nobody reads
go func() {
    result := compute()
    ch <- result  // Blocks if nobody reads ch
}()

// ✅ Fix: use buffered channel or select with done
ch := make(chan Result, 1)
go func() {
    ch <- compute()
}()

// ❌ Closing a channel from the receiver side
close(ch) // Only the sender should close

// ✅ Fix: sender closes, receiver ranges
go func() {
    defer close(ch)
    for _, item := range items {
        ch <- process(item)
    }
}()
for result := range ch {
    handle(result)
}

// ❌ time.Sleep in tests - flaky
time.Sleep(100 * time.Millisecond)
assert(result != nil)

// ✅ Fix: use channels or synctest
select {
case result := <-resultCh:
    assert(result != nil)
case <-time.After(1 * time.Second):
    t.Fatal("timeout")
}
```
