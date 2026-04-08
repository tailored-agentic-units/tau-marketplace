---
name: go-patterns
description: >
  Go design patterns for the tau ecosystem. Use when designing interfaces,
  organizing packages, handling errors, or structuring configuration.
  Triggers: interface design, error handling, package structure, dependency
  hierarchy, configuration lifecycle, parameter encapsulation, Go idioms.
---

# Go Design Patterns

## When This Skill Applies

- Designing package structures and interfaces
- Implementing error handling patterns
- Organizing code within files
- Managing configuration lifecycle
- Creating clean dependency hierarchies

## Principles

### 1. Encapsulation and Data Access

Provide methods for accessing meaningful values from complex nested structures.
Do not expose or require direct field access to inner state.

**Example**: Instead of `chunk.Choices[0].Delta.Content`, provide
`chunk.ExtractContent()` that handles nested access and bounds checking.

### 2. Interface-Based Layer Interconnection

Layers communicate through interfaces, not concrete types.

- Define interfaces at package boundaries
- Higher layers depend on interfaces from lower layers
- Create concrete implementations at system edges
- Use dependency injection

### 3. Package Dependency Hierarchy

Maintain unidirectional dependencies: high-level → low-level.

- Lower layers must not import higher layers
- Shared types belong in the lowest layer that needs them
- Use interfaces to invert dependencies when needed

### 4. Configuration Lifecycle

Configuration only exists during initialization.

- Load config → Transform to domain types → Discard config
- Domain objects should not hold config references
- Runtime behavior depends on initialized state, not config values

### 5. Parameter Encapsulation

More than two parameters? Use a struct.

```go
// Instead of: Execute(ctx, capability, input, timeout, retries)
// Use: Execute(request ExecuteRequest)
```

### 6. Modern Go Idioms

- `sync.WaitGroup.Go(func())` - Combined Add/Done/goroutine
- `for range n` - Integer range iteration
- `min()`/`max()` - Built-in comparison
- `errors.Join(...)` - Combine multiple errors
- `defer close(channel)` - Guaranteed channel closure

### 7. Error Handling

- Wrap errors with context: `fmt.Errorf("operation failed: %w", err)`
- Define package-level errors in `errors.go`
- Use `Err` prefix for exported error variables
- Error messages: lowercase, no punctuation
