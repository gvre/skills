# Generics and Iterators Guide Reference

Comprehensive guide to Go generics (1.18+) and range-over-function iterators (1.23+): type parameters, constraints, generic data structures, and composable lazy sequences.

## When NOT to Use Generics

```go
// ❌ Don't use generics just to avoid writing a concrete type
func PrintUser[T User](u T) { fmt.Println(u.Name) }  // just use User directly

// ❌ Don't use generics when `any` suffices
func Wrap[T any](v T) []any { return []any{v} }       // just accept any

// ❌ Don't add type parameters to methods that don't need them
func (s *Store[T]) Close() error { ... }               // T is unused in Close

// ❌ Don't use generics for one-off helper functions
func maxOfTwo[T cmp.Ordered](a, b T) T { ... }        // just use the concrete type

// ✅ DO use generics for reusable algorithms that work across types
func Map[T, U any](s []T, fn func(T) U) []U { ... }

// ✅ DO use generics for type-safe collections
type Set[T comparable] struct { items map[T]struct{} }

// ✅ DO use generics when constraints enforce meaningful behavior
func Min[T cmp.Ordered](a, b T) T { ... }
```

**Ask yourself before adding generics:**

1. Will this function/type be used with more than one concrete type?
2. Does a type parameter add safety that `any` or an interface doesn't provide?
3. Is the generic version simpler than writing 2-3 concrete versions?

If the answer to all three is "no", use a concrete type or interface.

---

## Generic Functions

```go
// ✅ Generic function for reusable algorithms
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// ✅ Constrained type parameters
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](values []T) T {
    var total T
    for _, v := range values {
        total += v
    }
    return total
}
```

---

## Constraints

```go
import "cmp"

// ✅ Use cmp.Ordered for ordered types
func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// ✅ Custom constraint with union
type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

func Abs[T Integer](x T) T {
    if x < 0 {
        return -x
    }
    return x
}

// ✅ Constraint with methods
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}
```

---

## Generic Data Structures

```go
// ✅ Generic Set
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{items: make(map[T]struct{})}
}

func (s *Set[T]) Add(item T)      { s.items[item] = struct{}{} }
func (s *Set[T]) Has(item T) bool { _, ok := s.items[item]; return ok }
func (s *Set[T]) Remove(item T)   { delete(s.items, item) }
func (s *Set[T]) Len() int        { return len(s.items) }

// ✅ Generic Result type
type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: value}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) Unwrap() (T, error) {
    return r.value, r.err
}

// ✅ Generic Stack
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) { s.items = append(s.items, item) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    n := len(s.items) - 1
    item := s.items[n]
    s.items = s.items[:n]
    return item, true
}
func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}
func (s *Stack[T]) Len() int { return len(s.items) }
```

---

## Self-Referential Generic Types (1.26+)

Generic types can refer to themselves in their own type parameter list, enabling recursive type constraints:

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

func (n *TreeNode[T]) Insert(val T) {
    if val.Less(n.Value) {
        if n.Left == nil {
            n.Left = &TreeNode[T]{Value: val}
        } else {
            n.Left.Insert(val)
        }
    } else {
        if n.Right == nil {
            n.Right = &TreeNode[T]{Value: val}
        } else {
            n.Right.Insert(val)
        }
    }
}

// Implement the Ordered constraint
type IntVal int

func (a IntVal) Less(b IntVal) bool { return a < b }

// Use it
tree := &TreeNode[IntVal]{Value: 42}
tree.Insert(10)
tree.Insert(55)
```

---

## Iterators (range-over-func, 1.23+)

Go 1.23 introduced range-over-function iterators via the `iter` package. Use them to create composable, lazy sequences.

### Basic Iterators

```go
import "iter"

// ✅ Single-value iterator (iter.Seq[T])
func (s *Set[T]) All() iter.Seq[T] {
    return func(yield func(T) bool) {
        for item := range s.items {
            if !yield(item) {
                return
            }
        }
    }
}

// ✅ Consume with range
for item := range mySet.All() {
    fmt.Println(item)
}

// ✅ Two-value iterator (iter.Seq2[K, V])
func Enumerate[T any](s iter.Seq[T]) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        i := 0
        for v := range s {
            if !yield(i, v) {
                return
            }
            i++
        }
    }
}

for i, item := range Enumerate(mySet.All()) {
    fmt.Printf("%d: %v\n", i, item)
}
```

### Composable Iterator Pipelines

```go
// ✅ Filter: keep elements that match a predicate
func Filter[T any](s iter.Seq[T], pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range s {
            if pred(v) {
                if !yield(v) {
                    return
                }
            }
        }
    }
}

// ✅ Map: transform elements
func MapIter[T, U any](s iter.Seq[T], fn func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range s {
            if !yield(fn(v)) {
                return
            }
        }
    }
}

// ✅ Take: limit to first n elements
func Take[T any](s iter.Seq[T], n int) iter.Seq[T] {
    return func(yield func(T) bool) {
        i := 0
        for v := range s {
            if i >= n {
                return
            }
            if !yield(v) {
                return
            }
            i++
        }
    }
}

// ✅ Compose into pipelines
result := slices.Collect(
    Take(
        Filter(mySet.All(), func(x int) bool { return x > 0 }),
        10,
    ),
)
```

### Standard Library Iterator Support

```go
import (
    "slices"
    "maps"
)

// Iterate over slices
for i, v := range slices.All(mySlice) { ... }
for v := range slices.Values(mySlice) { ... }
for v := range slices.Backward(mySlice) { ... }

// Iterate over maps
for k := range maps.Keys(myMap) { ... }
for v := range maps.Values(myMap) { ... }

// Collect iterators into concrete types
slice := slices.Collect(myIterator)
sorted := slices.Sorted(myIterator)
sortedBy := slices.SortedFunc(myIterator, cmp.Compare)
m := maps.Collect(myKeyValueIterator)

// Create iterators from slices/ranges
for v := range slices.Chunk(mySlice, 3) { ... }  // iterate in chunks of 3
```

---

## Generics Best Practices

- **Use generics for reusable data structures and algorithms**, not for everything
- **Prefer interfaces** when you need runtime polymorphism (method dispatch)
- **Use `comparable`** constraint for map keys and equality checks
- **Use `cmp.Ordered`** for types that support `<`, `>`, `<=`, `>=`
- **Don't over-engineer:** if a function works fine with `any` or a concrete type, don't add generics
- **Name type parameters clearly:** `T` for single, `K, V` for maps, `T, U` for transformations
- **Use `iter.Seq[T]`** for single-value iterators, **`iter.Seq2[K, V]`** for key-value pairs
- **Always check `yield`'s return value** and stop iteration if it returns `false`
- **Use `slices.Collect`**, `maps.Collect` to materialize iterators into concrete types
- **Prefer iterators over returning `[]T`** when the caller may not need all elements (lazy evaluation)