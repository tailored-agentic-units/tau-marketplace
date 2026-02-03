---
name: tau-orchestrate
description: >
  Use when building applications with the tau-orchestrate library (installation, hub coordination,
  messaging, state management, workflow patterns, observability, configuration).
  Triggers: import tau-orchestrate, hub.New, messaging.NewRequest, state.NewGraph,
  workflows.ProcessChain, workflows.ProcessParallel, workflows.ProcessConditional,
  Observer, CheckpointStore, config.Default, hub coordination, workflow patterns,
  state graphs, agent coordination, multi-agent.
---

# Building with tau-orchestrate

## When This Skill Applies

- Installing and importing tau-orchestrate
- Creating hubs and registering agents
- Sending messages between agents
- Building state graphs for workflow execution
- Running sequential, parallel, or conditional workflows
- Configuring observers for execution tracing
- Setting up checkpointing for workflow persistence

## Getting Started

### Installation

```bash
go get github.com/tailored-agentic-units/tau-orchestrate
```

### Basic Hub Setup

```go
import (
    "context"
    "time"

    "github.com/tailored-agentic-units/tau-orchestrate/pkg/config"
    "github.com/tailored-agentic-units/tau-orchestrate/pkg/hub"
    "github.com/tailored-agentic-units/tau-orchestrate/pkg/messaging"
    "github.com/tailored-agentic-units/tau-core/pkg/agent"
)

ctx := context.Background()

hubConfig := config.DefaultHubConfig()
hubConfig.Name = "main-hub"
h := hub.New(ctx, hubConfig)
defer h.Shutdown(5 * time.Second)

agent1, _ := agent.New(agentConfig1)

handler := func(ctx context.Context, msg *messaging.Message, msgCtx *hub.MessageContext) (*messaging.Message, error) {
    response, _ := agent1.Chat(ctx, msg.Data.(string))
    return messaging.NewResponse(agent1.ID(), msg.From, msg.ID, response.Content()).Build(), nil
}

h.RegisterAgent(agent1, handler)
```

## Hub Coordination

### Agent Registration

Agents implement `hub.Agent` interface (requires only `ID() string`). tau-core agents satisfy this directly.

```go
h.RegisterAgent(agent, handler)
h.UnregisterAgent(agentID)
```

### Communication Patterns

**Send (Fire-and-Forget)**:
```go
h.Send(ctx, fromAgentID, toAgentID, data)
```

**Request/Response** (synchronous with correlation):
```go
response, err := h.Request(ctx, fromAgentID, toAgentID, data)
```

**Broadcast** (all agents except sender):
```go
h.Broadcast(ctx, fromAgentID, data)
```

**Pub/Sub** (topic-based):
```go
h.Subscribe(agentID, "topic-name")
h.Publish(ctx, fromAgentID, "topic-name", data)
```

### Multi-Hub Coordination

Agents can register with multiple hubs, acting as bridges:

```go
globalHub.RegisterAgent(agent, globalHandler)
taskHub.RegisterAgent(agent, taskHandler)
```

### Hub Metrics

```go
metrics := h.GetMetrics()
// metrics.LocalAgents, metrics.MessagesSent, metrics.MessagesRecv
```

## Hub Lifecycle

### Creating a Hub

Hubs are created with a context and configuration. The context controls the hub's lifetime.

```go
ctx := context.Background()
hubConfig := config.DefaultHubConfig()
hubConfig.Name = "my-hub"
hubConfig.BufferSize = 200  // Default: 100
hubConfig.Timeout = 60 * time.Second  // Default: 30s

h := hub.New(ctx, hubConfig)
```

### Shutdown Semantics

The `Shutdown` method stops the hub and drains pending messages.

**Graceful shutdown** (with timeout): Waits for in-flight messages to complete, up to the specified duration.

```go
// Wait up to 5 seconds for pending messages to drain
h.Shutdown(5 * time.Second)
```

**Immediate shutdown** (zero timeout): Stops accepting new messages and cancels in-flight work immediately.

```go
// Stop immediately, discard pending messages
h.Shutdown(0)
```

**Best practice**: Always use `defer h.Shutdown(duration)` immediately after creating a hub to ensure cleanup on all exit paths. Agents registered with the hub remain valid after shutdown -- the hub does not own agent lifecycle.

### Context Cancellation

If the parent context is cancelled, the hub will begin shutdown automatically. Explicit `Shutdown` is still recommended for controlled draining.

```go
ctx, cancel := context.WithCancel(context.Background())
h := hub.New(ctx, hubConfig)

// Cancelling context triggers hub shutdown
cancel()
```

## Messaging

### Message Types

`MessageTypeRequest`, `MessageTypeResponse`, `MessageTypeNotification`, `MessageTypeBroadcast`

### Priority Levels

`PriorityLow`, `PriorityNormal`, `PriorityHigh`, `PriorityCritical`

### Fluent Builders

```go
msg := messaging.NewRequest("agent-a", "agent-b", taskData).
    Priority(messaging.PriorityHigh).
    Headers(map[string]string{"source": "workflow"}).
    Build()

resp := messaging.NewResponse("agent-b", "agent-a", msg.ID, resultData).Build()
notif := messaging.NewNotification("agent-a", "agent-b", statusUpdate).Build()
broadcast := messaging.NewBroadcast("agent-a", announcement).Build()
```

## State Management

### State Type

State holds workflow data as `map[string]any` with immutable operations:

```go
s := state.New(observer)
s = s.Set("key", "value")
val, exists := s.Get("key")
merged := s.Merge(otherState)
cloned := s.Clone()
```

### Secrets

Sensitive data excluded from JSON serialization and observer snapshots:

```go
s = s.SetSecret("token", "abc123")
val, exists := s.GetSecret("token")
s = s.DeleteSecret("token")
```

### StateNode

Computation steps that receive and return state:

```go
node := state.FunctionNode(func(ctx context.Context, s state.State) (state.State, error) {
    input, _ := s.Get("input")
    result := process(input)
    return s.Set("output", result), nil
})
```

### Edge Predicates

Control transitions between nodes:

```go
state.AlwaysTransition()
state.KeyExists("result")
state.KeyEquals("status", "approved")
state.Not(predicate)
state.And(pred1, pred2)
state.Or(pred1, pred2)
```

### State Graph Execution

```go
cfg := config.DefaultGraphConfig()
graph, _ := state.NewGraph(cfg)

graph.AddNode("process", processNode)
graph.AddNode("validate", validateNode)
graph.AddEdge("process", "validate", state.AlwaysTransition())
graph.SetEntryPoint("process")
graph.SetExitPoint("validate")

finalState, err := graph.Execute(ctx, initialState)
```

### Dependency Injection

```go
graph, _ := state.NewGraphWithDeps(cfg, observer, checkpointStore)
```

## Workflow Patterns

### Sequential Chain (ProcessChain)

Process items sequentially with state accumulation (fold/reduce pattern):

```go
type Context struct {
    Results []string
}

processor := func(ctx context.Context, item string, state Context) (Context, error) {
    response, err := agent.Chat(ctx, item)
    if err != nil {
        return state, err
    }
    state.Results = append(state.Results, response.Content())
    return state, nil
}

cfg := config.DefaultChainConfig()
result, err := workflows.ProcessChain(ctx, cfg, items, Context{}, processor, nil)
// result.FinalState contains accumulated context
// result.IntermediateStates available if cfg.CaptureIntermediateStates = true
```

#### Complete ProcessChain Working Example

End-to-end example that summarizes a list of documents sequentially:

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/tailored-agentic-units/tau-core/pkg/agent"
    "github.com/tailored-agentic-units/tau-core/pkg/config"
    "github.com/tailored-agentic-units/tau-orchestrate/pkg/workflows"
    orchConfig "github.com/tailored-agentic-units/tau-orchestrate/pkg/config"
)

type SummaryContext struct {
    Summaries []string
    Total     int
}

func main() {
    ctx := context.Background()

    // Create the agent
    agentCfg, err := config.LoadAgentConfig("config.json")
    if err != nil {
        log.Fatal(err)
    }
    a, err := agent.New(agentCfg)
    if err != nil {
        log.Fatal(err)
    }

    // Documents to summarize
    documents := []string{
        "The Go programming language was designed at Google...",
        "Concurrency in Go is based on CSP...",
        "Go modules provide dependency management...",
    }

    // Define the processor
    processor := func(ctx context.Context, doc string, state SummaryContext) (SummaryContext, error) {
        prompt := fmt.Sprintf("Summarize this in one sentence: %s", doc)
        response, err := a.Chat(ctx, prompt)
        if err != nil {
            return state, fmt.Errorf("failed to summarize document %d: %w", state.Total+1, err)
        }
        state.Summaries = append(state.Summaries, response.Choices[0].Message.Content)
        state.Total++
        return state, nil
    }

    // Run the chain
    chainCfg := orchConfig.DefaultChainConfig()
    chainCfg.CaptureIntermediateStates = true

    result, err := workflows.ProcessChain(ctx, chainCfg, documents, SummaryContext{}, processor, nil)
    if err != nil {
        log.Fatalf("chain failed: %v", err)
    }

    // Access results
    fmt.Printf("Summarized %d documents:\n", result.FinalState.Total)
    for i, summary := range result.FinalState.Summaries {
        fmt.Printf("  %d. %s\n", i+1, summary)
    }
}
```

### Parallel Execution

Process items concurrently with worker pool:

```go
processor := func(ctx context.Context, item string) (string, error) {
    response, err := agent.Chat(ctx, item)
    if err != nil {
        return "", err
    }
    return response.Content(), nil
}

cfg := config.DefaultParallelConfig() // Auto-detects worker count
result, err := workflows.ProcessParallel(ctx, cfg, items, processor, nil)
// result.Results preserves original order
// result.Errors contains TaskError items on partial failure
```

**Collect-All-Errors Mode**:
```go
cfg := config.ParallelConfig{
    FailFast: false,
    Observer: "slog",
}
result, err := workflows.ProcessParallel(ctx, cfg, items, processor, nil)
// err only if ALL items failed
// check result.Errors for partial failures
```

### Conditional Routing

Route state through handlers based on predicate evaluation:

```go
predicate := func(s state.State) (string, error) {
    status, _ := s.Get("status")
    if status == "approved" {
        return "approve", nil
    }
    return "reject", nil
}

routes := workflows.Routes[state.State]{
    Handlers: map[string]workflows.RouteHandler[state.State]{
        "approve": approveHandler,
        "reject":  rejectHandler,
    },
    Default: fallbackHandler,
}

cfg := config.DefaultConditionalConfig()
finalState, err := workflows.ProcessConditional(ctx, cfg, initialState, predicate, routes)
```

## Integration Helpers

Wrap workflow patterns as StateNodes for composition within state graphs:

```go
analysisNode := workflows.ChainNode(chainCfg, sections, analyzeSection, nil)

reviewNode := workflows.ParallelNode(
    parallelCfg, reviewers, reviewDocument, nil, aggregateReviews,
)

routingNode := workflows.ConditionalNode(conditionalCfg, checkConsensus, approvalRoutes)

graph.AddNode("analyze", analysisNode)
graph.AddNode("review", reviewNode)
graph.AddNode("route", routingNode)
```

## Error Types

### Hub Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Agent not found | Sending to an unregistered agent ID | Verify the agent is registered with `h.RegisterAgent()` before sending. Check for typos in agent IDs. |
| Message timeout | No response within hub timeout | Increase `hubConfig.Timeout` or check that the target agent's handler is not blocking indefinitely. |
| Buffer full | Hub message buffer at capacity | Increase `hubConfig.BufferSize` or add backpressure in the producer. Consider whether the consumer is processing fast enough. |
| Hub shutdown | Operation attempted after shutdown | Ensure operations complete before calling `h.Shutdown()`. Use context to coordinate. |

### Workflow Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `ProcessChain` error | A processor function returned an error | The chain stops at the failing item. Check `err` for details. Fix the processor or skip bad items. |
| `ProcessParallel` partial failure | Some items failed (FailFast: false) | Check `result.Errors` for `TaskError` items containing the index and error. `result.Results` still contains successful items. |
| `ProcessParallel` total failure | All items failed | The returned `err` is non-nil. Inspect individual errors in `result.Errors`. |
| Graph max iterations | Graph exceeded `MaxIterations` | Increase `cfg.MaxIterations` or check for cycles in edge predicates. |
| Checkpoint load failure | Run ID not found in store | Verify the run ID exists with `store.List()`. Checkpoints may have been cleaned up if `Preserve: false`. |

### Error Handling Pattern

```go
result, err := workflows.ProcessParallel(ctx, cfg, items, processor, nil)
if err != nil {
    // Total failure -- all items failed
    log.Fatalf("all items failed: %v", err)
}
if len(result.Errors) > 0 {
    // Partial failure -- some items succeeded
    for _, taskErr := range result.Errors {
        log.Printf("item %d failed: %v", taskErr.Index, taskErr.Err)
    }
}
// result.Results contains successful outputs (in original order)
```

## Observability

### Observer Interface

```go
type Observer interface {
    OnEvent(ctx context.Context, event Event)
}
```

### Built-in Observers

- **NoOpObserver**: Zero overhead, discards all events
- **SlogObserver**: Structured logging via Go's `slog` package
- **MultiObserver**: Broadcasts events to multiple wrapped observers

### Observer Registry

```go
observability.RegisterObserver("custom", myObserver)
observer, err := observability.GetObserver("custom")
```

### Configuration-Driven Selection

All config types accept an `Observer` string field resolved via registry:
```go
cfg := config.DefaultGraphConfig()  // Observer: "slog"
cfg.Observer = "noop"               // Override to zero overhead
```

## Checkpointing

### CheckpointStore Interface

```go
type CheckpointStore interface {
    Save(state State) error
    Load(runID string) (State, error)
    Delete(runID string) error
    List() ([]string, error)
}
```

### MemoryCheckpointStore

Thread-safe in-memory storage for development and testing:

```go
store := state.NewMemoryCheckpointStore()
state.RegisterCheckpointStore("memory", store)
```

### Graph Configuration

```go
cfg := config.GraphConfig{
    Name:          "my-workflow",
    Observer:      "slog",
    MaxIterations: 100,
    Checkpoint: config.CheckpointConfig{
        Store:    "memory",
        Interval: 1,       // Save after every node
        Preserve: false,   // Clean up on success
    },
}
```

### Resume from Checkpoint

```go
finalState, err := graph.Resume(ctx, runID)
```

## Configuration

### Default Factories

```go
config.DefaultHubConfig()          // Buffer: 100, Timeout: 30s
config.DefaultGraphConfig()        // MaxIterations: 100, Observer: "slog"
config.DefaultChainConfig()        // CaptureIntermediateStates: false
config.DefaultParallelConfig()     // Workers: auto, FailFast: true
config.DefaultConditionalConfig()  // Observer: "slog"
```

### Configuration Merge

All config types support merging where loaded values override defaults:

```go
base := config.DefaultParallelConfig()
override := config.ParallelConfig{MaxWorkers: 4}
merged := base.Merge(&override)
```

### Relationship to tau-core

tau-orchestrate extends tau-core with coordination primitives. tau-core provides LLM protocol execution (Chat, Vision, Tools, Embeddings). tau-orchestrate provides the orchestration layer:

- **tau-core agents** satisfy `hub.Agent` interface directly
- **Hub handlers** call tau-core agent methods for LLM processing
- **Workflow processors** use tau-core agents for each step
- **State graphs** compose tau-core calls into multi-step workflows

## Related Skills

- **tau:tau-core** - Core agent creation, protocol usage, provider setup, and mock testing
