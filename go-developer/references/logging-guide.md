# Logging Guide Reference

Comprehensive guide to structured logging in Go with `log/slog` (standard library, Go 1.21+): setup, handlers, context-aware logging, middleware, and best practices.

## Basics

```go
import "log/slog"

// ✅ Structured logging with key-value pairs
slog.Info("user created", "id", user.ID, "name", user.Name)
slog.Error("database error", "err", err, "query", query)
slog.Debug("cache hit", "key", key, "ttl", ttl)
slog.Warn("rate limit approaching", "current", count, "limit", max)

// ✅ Logger with pre-set attributes
logger := slog.With("service", "user", "env", os.Getenv("ENV"))
logger.Info("starting up")  // includes service and env in every log line

// ✅ Attribute groups for structured nesting
slog.Info("request handled",
    slog.Group("request",
        slog.String("method", r.Method),
        slog.String("path", r.URL.Path),
    ),
    slog.Group("response",
        slog.Int("status", status),
        slog.Duration("latency", elapsed),
    ),
)
```

---

## Handler Setup

### JSON Handler (production)

```go
func setupLogger(cfg LogConfig) *slog.Logger {
    level := slog.LevelInfo
    switch strings.ToLower(cfg.Level) {
    case "debug":
        level = slog.LevelDebug
    case "warn":
        level = slog.LevelWarn
    case "error":
        level = slog.LevelError
    }

    opts := &slog.HandlerOptions{Level: level}

    var handler slog.Handler
    if cfg.JSON {
        handler = slog.NewJSONHandler(os.Stdout, opts)
    } else {
        handler = slog.NewTextHandler(os.Stdout, opts)
    }

    return slog.New(handler)
}

// ✅ Configure at startup, set as default
func main() {
    logger := setupLogger(cfg.Log)
    slog.SetDefault(logger)
}
```

### Text Handler (development)

```go
// Human-readable output for local development
handler := slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
    Level:     slog.LevelDebug,
    AddSource: true, // include file:line in log output
})
slog.SetDefault(slog.New(handler))
```

### Dynamic Log Level

```go
// ✅ Change log level at runtime (e.g., via HTTP endpoint)
var levelVar slog.LevelVar
levelVar.Set(slog.LevelInfo)

handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: &levelVar,
})
slog.SetDefault(slog.New(handler))

// Later, change level without restart:
levelVar.Set(slog.LevelDebug)
```

---

## Dependency Injection

```go
// ✅ Pass logger via struct fields — not as a global
type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger.With("component", "user_service"),
    }
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*app.User, error) {
    s.logger.InfoContext(ctx, "fetching user", "id", id)

    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        s.logger.ErrorContext(ctx, "failed to fetch user", "id", id, "err", err)
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return user, nil
}
```

---

## Context-Aware Logging

### Logger in Context

```go
// ✅ Use typed keys to avoid collisions
type contextKey string

const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}
```

### Using `LogContext` methods

```go
// ✅ Use *Context variants to propagate trace/request info
func (s *Service) Process(ctx context.Context, req Request) error {
    slog.InfoContext(ctx, "processing request", "type", req.Type)
    // ...
    return nil
}
```

---

## HTTP Middleware

```go
// ✅ Request logging middleware with request ID and timing
func requestLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID()
        }

        logger := slog.With(
            "request_id", requestID,
            "method", r.Method,
            "path", r.URL.Path,
            "remote_addr", r.RemoteAddr,
        )

        // Store enriched logger in context
        ctx := WithLogger(r.Context(), logger)

        // Wrap response writer to capture status code
        wrapped := &statusResponseWriter{ResponseWriter: w, status: http.StatusOK}

        logger.Info("request started")
        next.ServeHTTP(wrapped, r.WithContext(ctx))

        logger.Info("request completed",
            "status", wrapped.status,
            "duration", time.Since(start),
        )
    })
}

type statusResponseWriter struct {
    http.ResponseWriter
    status int
}

func (w *statusResponseWriter) WriteHeader(code int) {
    w.status = code
    w.ResponseWriter.WriteHeader(code)
}
```

---

## Error Logging Patterns

```go
// ✅ Log at the boundary, not at every layer
// Only the top-level handler/caller should log the error.
// Lower layers wrap and return errors — they don't log.

// Repository: wrap and return (no logging)
func (r *Repo) FindByID(ctx context.Context, id int64) (*User, error) {
    // ...
    if err != nil {
        return nil, fmt.Errorf("find user %d: %w", id, err)
    }
    return &user, nil
}

// Service: wrap and return (no logging)
func (s *Service) GetUser(ctx context.Context, id int64) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }
    return user, nil
}

// Handler: log at the boundary
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.svc.GetUser(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            // Client error: no need to log at error level
            slog.DebugContext(r.Context(), "user not found", "id", id)
            http.Error(w, "not found", http.StatusNotFound)
            return
        }
        // Unexpected error: log at error level
        slog.ErrorContext(r.Context(), "get user failed", "id", id, "err", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    // ...
}
```

---

## Logging Best Practices

- **Use `log/slog`** — not `fmt.Println`, not `log.Printf`, not third-party loggers unless you have a specific need
- **Use structured key-value pairs** — not `fmt.Sprintf` formatted strings
- **Pass logger via dependency injection** — not as a global; use `slog.SetDefault` only at startup
- **Log at boundaries** — handlers/entrypoints log; lower layers wrap and return errors
- **Use `*Context` methods** (`InfoContext`, `ErrorContext`) to propagate request-scoped data
- **Include request IDs** in every log line via middleware
- **NEVER log sensitive data** — passwords, tokens, API keys, PII
- **Use appropriate log levels:**
  - `Debug` — verbose detail useful during development
  - `Info` — normal operations (request handled, job completed)
  - `Warn` — something unexpected but recoverable (approaching rate limit, fallback used)
  - `Error` — something failed and needs attention (unhandled error, external service down)
- **Use `slog.LevelVar`** for dynamic level changes without restart
- **Use `AddSource: true`** in development to include file and line numbers
- **Use JSON handler in production** for machine-parseable output
- **Use text handler in development** for human-readable output
