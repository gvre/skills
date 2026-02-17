# Testing Guide Reference

Comprehensive guide to testing Go applications: table-driven tests, mocking, benchmarks, fuzzing, and integration tests.

## Basic Testing

### Table-Driven Tests (Idiomatic Go)

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 1, 2, 3},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 5, 5},
        {"overflow boundary", math.MaxInt64, 1, math.MinInt64},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Subtests with `t.Run`

```go
func TestUserService(t *testing.T) {
    t.Run("GetUser", func(t *testing.T) {
        t.Run("existing user", func(t *testing.T) {
            // ...
        })
        t.Run("not found", func(t *testing.T) {
            // ...
        })
    })

    t.Run("CreateUser", func(t *testing.T) {
        // ...
    })
}

// Run a specific subtest:
// go test -run TestUserService/GetUser/existing_user
```

### Testing Errors

```go
func TestGetUser_NotFound(t *testing.T) {
    svc := NewUserService(&mockRepo{err: ErrNotFound})

    _, err := svc.GetUser(context.Background(), 999)

    if !errors.Is(err, ErrNotFound) {
        t.Errorf("got %v, want ErrNotFound", err)
    }
}

func TestGetUser_ValidationError(t *testing.T) {
    svc := NewUserService(&mockRepo{})

    _, err := svc.GetUser(context.Background(), -1)

    var valErr *ValidationError
    if !errors.As(err, &valErr) {
        t.Fatalf("got %T, want *ValidationError: %v", err, err)
    }
    if valErr.Field != "id" {
        t.Errorf("got field %q, want 'id'", valErr.Field)
    }
}
```

---

## Mocking with Interfaces

Go's interface system makes mocking straightforward - no magic framework needed:

```go
// Production interface
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Mock implementation for tests
type mockUserRepo struct {
    user *User
    err  error
    saved []*User
}

func (m *mockUserRepo) FindByID(_ context.Context, _ int64) (*User, error) {
    return m.user, m.err
}

func (m *mockUserRepo) Save(_ context.Context, user *User) error {
    m.saved = append(m.saved, user)
    return m.err
}

// Use in tests
func TestUserService_CreateUser(t *testing.T) {
    repo := &mockUserRepo{}
    svc := NewUserService(repo, slog.Default())

    err := svc.CreateUser(context.Background(), "Alice", "alice@example.com")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if len(repo.saved) != 1 {
        t.Errorf("got %d saved users, want 1", len(repo.saved))
    }
    if repo.saved[0].Name != "Alice" {
        t.Errorf("got name %q, want 'Alice'", repo.saved[0].Name)
    }
}
```

### Testify for Assertions (Optional)

When testify reduces boilerplate significantly:

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUserService_GetUser(t *testing.T) {
    repo := &mockUserRepo{user: &User{ID: 1, Name: "Alice"}}
    svc := NewUserService(repo, slog.Default())

    user, err := svc.GetUser(context.Background(), 1)

    require.NoError(t, err)           // Stops test on failure
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, int64(1), user.ID)
}
```

---

## Test Helpers and Fixtures

### `t.Helper()` for Reusable Assertions

```go
func assertUserEqual(t *testing.T, got, want *User) {
    t.Helper() // Makes failure point to the caller, not this function
    if got.ID != want.ID {
        t.Errorf("user ID: got %d, want %d", got.ID, want.ID)
    }
    if got.Name != want.Name {
        t.Errorf("user name: got %q, want %q", got.Name, want.Name)
    }
}
```

### `TestMain` for Package-Level Setup

```go
func TestMain(m *testing.M) {
    // Setup: start test database, etc.
    db, cleanup := setupTestDB()
    testDB = db

    code := m.Run()

    // Teardown
    cleanup()
    os.Exit(code)
}
```

### `t.Cleanup` for Per-Test Teardown

```go
func TestWithTempDir(t *testing.T) {
    dir := t.TempDir() // Automatically cleaned up after test

    // Or register custom cleanup:
    conn := openConnection()
    t.Cleanup(func() {
        conn.Close()
    })
}
```

---

## Parallel Tests

```go
func TestParallelSafe(t *testing.T) {
    tests := []struct {
        name string
        id   int64
    }{
        {"user 1", 1},
        {"user 2", 2},
        {"user 3", 3},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Run subtests concurrently

            // Each subtest must be independent
            svc := NewUserService(&mockUserRepo{}, slog.Default())
            _, err := svc.GetUser(context.Background(), tt.id)
            _ = err
        })
    }
}
```

---

## Benchmarks

```go
func BenchmarkAdd(b *testing.B) {
    for b.Loop() { // Preferred since Go 1.24+ (range b.N still works but b.Loop() is idiomatic)
        Add(1, 2)
    }
}

func BenchmarkUserService_GetUser(b *testing.B) {
    repo := &mockUserRepo{user: &User{ID: 1, Name: "Alice"}}
    svc := NewUserService(repo, slog.Default())
    ctx := context.Background()

    b.ResetTimer()
    for b.Loop() {
        svc.GetUser(ctx, 1)
    }
}

// Run benchmarks:
// go test -bench=. -benchmem ./...
// go test -bench=BenchmarkAdd -count=5 ./...
```

### Benchmark with Allocation Tracking

```go
func BenchmarkJSONMarshal(b *testing.B) {
    user := &User{ID: 1, Name: "Alice", Email: "alice@example.com"}

    b.ReportAllocs()
    for b.Loop() {
        _, err := json.Marshal(user)
        if err != nil {
            b.Fatal(err)
        }
    }
}
// Output: BenchmarkJSONMarshal-8   500000   2400 ns/op   256 B/op   3 allocs/op
```

---

## Fuzzing

```go
func FuzzParseURL(f *testing.F) {
    // Seed corpus
    f.Add("https://example.com/path?query=1")
    f.Add("http://localhost:8080")
    f.Add("")

    f.Fuzz(func(t *testing.T, input string) {
        // Must not panic
        result, err := ParseURL(input)
        if err != nil {
            return // Errors are acceptable
        }
        // Invariant: round-trip must be stable
        if result.String() != input {
            // This might be expected; check your invariants carefully
        }
    })
}

// Run fuzzing:
// go test -fuzz=FuzzParseURL -fuzztime=30s ./...
// go test -fuzz=FuzzParseURL -fuzztime=30s -fuzzminimizetime=10s ./...
```

---

## Integration Tests

### Build Tags for Separation

```go
//go:build integration

package store_test

import (
    "testing"
    "database/sql"
)

func TestPostgresRepo_FindByID(t *testing.T) {
    db, err := sql.Open("postgres", os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        t.Fatalf("open db: %v", err)
    }
    defer db.Close()

    repo := NewPostgresRepo(db)
    // ...
}

// Run integration tests:
// go test -tags=integration ./...
```

### Skipping Without Build Tags

```go
func TestPostgresRepo_FindByID(t *testing.T) {
    if os.Getenv("TEST_DATABASE_URL") == "" {
        t.Skip("TEST_DATABASE_URL not set, skipping integration test")
    }
    // ...
}
```

---

## Testing Concurrent Code

### `testing/synctest` (Stable in 1.25)

```go
import "testing/synctest"

func TestCacheExpiry(t *testing.T) {
    synctest.Run(func() {
        cache := NewCache(1 * time.Minute)
        cache.Set("key", "value")

        // Advance virtual time by 2 minutes
        synctest.Wait()
        time.Sleep(2 * time.Minute) // Virtual - instant

        if _, ok := cache.Get("key"); ok {
            t.Error("expected cache entry to be expired")
        }
    })
}
```

### Race Detection

```go
// Always run with -race in CI
// go test -race ./...

func TestConcurrentCache(t *testing.T) {
    cache := NewCache()
    var wg sync.WaitGroup

    for range 100 {
        wg.Go(func() {
            cache.Set("key", "value")
            cache.Get("key")
        })
    }
    wg.Wait()
}
```

---

## Structured Output for CI

```bash
# JSON output for CI integration (machine-readable test results)
go test -json ./...

# Combine with race detection and coverage
go test -race -cover -json ./... > test-results.json

# Use gotestfmt or similar to render JSON results in CI
go test -json ./... 2>&1 | gotestfmt
```

JSON output produces one JSON object per line with fields like `Action` (`run`, `pass`, `fail`, `output`), `Package`, `Test`, and `Elapsed`. This integrates well with CI systems (GitHub Actions, GitLab CI, etc.) for structured test reporting.

---

## Coverage

```bash
# Run with coverage
go test -cover ./...

# Generate HTML report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Show per-function coverage
go tool cover -func=coverage.out

# Fail if coverage drops below threshold (in CI)
go test -cover ./... | grep -E "coverage: [0-9]+\.[0-9]+%" | \
    awk '{if ($2+0 < 80) exit 1}'
```

---

## Testing Best Practices

- **ALWAYS use table-driven tests** with `t.Run` for multiple scenarios
- **Use `t.Helper()`** in assertion helpers so failures point to the caller
- **Mock via interfaces** - define small interfaces, implement mocks manually
- **Use `-race` flag** in CI to detect data races
- **Use `t.Parallel()`** for independent tests to speed up the suite
- **Use `t.TempDir()`** for temporary files - auto-cleaned up
- **Use `t.Cleanup()`** for resource teardown instead of `defer` in subtests
- **Test behavior, not implementation** - tests should survive refactoring
- **Aim for >80% coverage** with meaningful tests
- **Fuzz critical parsers** - any function that parses untrusted input
- **Benchmark before optimizing** - measure first, then optimize
- **Keep tests fast** - slow tests discourage running them
- **Use `testify/require`** for fatal assertions (stops test immediately)
- **Use `testify/assert`** for non-fatal assertions (continues test)
