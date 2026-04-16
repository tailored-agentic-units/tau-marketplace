# Modern Go Idioms

Language and standard-library features added since Go 1.20 that the tau
ecosystem uses regularly. Baseline assumed by this skill is Go 1.26.

## Contents

1. [Version baseline](#version-baseline)
2. [`maps.Copy`](#mapscopy)
3. [`slices` package](#slices-package)
4. [`min` / `max` builtins](#min--max-builtins)
5. [`for range n`](#for-range-n)
6. [`sync.WaitGroup.Go`](#syncwaitgroupgo)
7. [Range-over-function iterators](#range-over-function-iterators)
8. [`errors.Join` (scope reminder)](#errorsjoin-scope-reminder)

## Version baseline

The tau workspace pins Go 1.26 (see `/home/jaime/tau/go.work` line 1: `go
1.26.1`). All examples in this skill assume that baseline. When maintaining
older code, prefer migrating to the modern idiom over preserving the old
shape — these features are net wins in clarity, not stylistic preferences.

## `maps.Copy`

Added in Go 1.21. Shallow-copies one map into another. Heavily used in
tau/agent for option merging.

```go
func (a *agent) mergeOptions(proto protocol.Protocol, opts ...map[string]any) map[string]any {
    options := make(map[string]any)
    if modelOpts := a.model.Options[proto]; modelOpts != nil {
        maps.Copy(options, modelOpts)
    }
    if len(opts) > 0 && opts[0] != nil {
        maps.Copy(options, opts[0])
    }
    return options
}
```

Source: `/home/jaime/tau/agent/agent.go:241-251`

Pattern: build a result map, apply defaults via `maps.Copy`, then layer
overrides via another `maps.Copy`. Later writes win.

When **not** to use: deep copies. `maps.Copy` is shallow — nested maps
remain shared by reference. If you need a deep copy, write it explicitly.

## `slices` package

Added in Go 1.21. The `slices` package replaces most hand-written slice
utilities.

Common operations:

- `slices.Contains(s, v)` — linear search (use `bytes.Contains` /
  `strings.Contains` for those types).
- `slices.Sort(s)` — in-place sort for ordered types.
- `slices.SortFunc(s, cmp)` — in-place sort with comparator.
- `slices.Delete(s, i, j)` — remove indices `[i, j)`.
- `slices.Insert(s, i, v...)` — insert at index.
- `slices.Clone(s)` — shallow copy for defensive returns.
- `slices.Equal(a, b)` — element-by-element equality.

Rule: reach for `slices.X` before writing a loop. The standard library is
usually faster, always more readable, and never has off-by-one bugs.

## `min` / `max` builtins

Added in Go 1.21. Work on any ordered type (numbers, strings, durations)
without generics ceremony.

```go
maxAttempt := min(attempt, 10)
// ...
return min(delay, cfg.MaxBackoffDuration())
```

Source: `/home/jaime/tau/agent/client/retry.go:85, 95`

Rule: never hand-write a two-line `if a < b { x = a } else { x = b }`. The
exception is when you also need to mutate other state based on the
comparison — then a full `if/else` reads better than calling `min` and then
inspecting which branch won.

## `for range n`

Added in Go 1.22. Range over an integer instead of using a counter loop.

```go
for range 5 {
    // do something 5 times; no index needed
}

for i := range 5 {
    // index 0..4
}
```

Replaces:

```go
for i := 0; i < n; i++ {
    // ...
}
```

Use when:

- You want a fixed-count loop without three-clause ceremony.
- You don't need increment hooks (`i += 2`, etc.).

Skip when you need irregular increment, decrement, or to start from a
non-zero value.

## `sync.WaitGroup.Go`

Added in Go 1.25. Combines `Add(1)`, `go`, and `Done()` into one call.

```go
wg.Go(func() {
    work()
})
```

Source: `/home/jaime/code/agent-lab/pkg/lifecycle/lifecycle.go:43-51`

See [concurrency.md#syncwaitgroupgo](concurrency.md#syncwaitgroupgo) for
details. Use in all new code; the Add/goroutine/Done triplet is for
maintaining pre-1.25 codebases.

## Range-over-function iterators

Added in Go 1.23. Functions with signature `func(yield func(V) bool)` can
appear in `range` expressions.

```go
import "iter"

func Protocols(f Format) iter.Seq[protocol.Protocol] {
    return func(yield func(protocol.Protocol) bool) {
        for _, p := range f.SupportedProtocols() {
            if !yield(p) {
                return
            }
        }
    }
}

// Consumer:
for p := range Protocols(f) {
    // use p
}
```

When to use:

- Generator-style sequences where building a slice up-front is wasteful.
- Composable iterator chains (`iter.Seq` and `iter.Seq2` interoperate
  through helpers).
- Lazy enumeration where the consumer might exit early.

When **not** to use: simple slice iteration. A slice is cheaper to build,
easier to inspect, and easier to debug than an iterator function.

The tau ecosystem has limited adoption of this pattern so far — it's a tool
to reach for when a slice doesn't fit, not a wholesale replacement.

## `errors.Join` (scope reminder)

Added in Go 1.20. Combines multiple errors into a single error implementing
`Unwrap() []error`.

**Scope**: only at aggregation boundaries — batch processors, validators
that collect all failures, multi-resource shutdown that wants to log every
cleanup error.

Do **not** use `errors.Join` for sequential operations where causation
matters — it hides which step actually broke. See
[error-handling.md#errorsjoin-at-aggregation-boundaries](error-handling.md#errorsjoin-at-aggregation-boundaries)
for the full rule and contrasting examples.
