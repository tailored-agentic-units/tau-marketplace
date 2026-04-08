# Orchestrate Reference

Multi-agent coordination and workflow patterns for the TAU kernel.

## Hub Coordination

### Basic Setup

```go
import (
    "github.com/tailored-agentic-units/kernel/orchestrate/config"
    "github.com/tailored-agentic-units/kernel/orchestrate/hub"
    "github.com/tailored-agentic-units/kernel/orchestrate/messaging"
    "github.com/tailored-agentic-units/kernel/agent"
)

ctx := context.Background()
hubConfig := config.DefaultHubConfig()
hubConfig.Name = "main-hub"
h := hub.New(ctx, hubConfig)
defer h.Shutdown(5 * time.Second)

a, _ := agent.New(agentConfig)
handler := func(ctx context.Context, msg *messaging.Message, msgCtx *hub.MessageContext) (*messaging.Message, error) {
    response, _ := a.Chat(ctx, msg.Data.(string))
    return messaging.NewResponse(a.ID(), msg.From, msg.ID, response.Content()).Build(), nil
}
h.RegisterAgent(a, handler)
```

### Communication Patterns

```go
h.Send(ctx, fromID, toID, data)                    // Fire-and-forget
response, err := h.Request(ctx, fromID, toID, data) // Request/Response
h.Broadcast(ctx, fromID, data)                      // All except sender
h.Subscribe(agentID, "topic")                       // Pub/Sub
h.Publish(ctx, fromID, "topic", data)
```

### Hub Lifecycle

```go
h.Shutdown(5 * time.Second)  // Graceful with timeout
h.Shutdown(0)                // Immediate
```

## Messaging

### Fluent Builders

```go
msg := messaging.NewRequest("agent-a", "agent-b", taskData).
    Priority(messaging.PriorityHigh).
    Headers(map[string]string{"source": "workflow"}).
    Build()

resp := messaging.NewResponse("agent-b", "agent-a", msg.ID, resultData).Build()
```

## State Management

### State Type

```go
s := state.New(observer)
s = s.Set("key", "value")
val, exists := s.Get("key")
s = s.SetSecret("token", "abc123")  // Excluded from serialization
```

### Edge Predicates

```go
state.AlwaysTransition()
state.KeyExists("result")
state.KeyEquals("status", "approved")
state.Not(predicate)
state.And(pred1, pred2)
state.Or(pred1, pred2)
```

### State Graph

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

## Workflow Patterns

### Sequential Chain

```go
processor := func(ctx context.Context, item string, state MyState) (MyState, error) {
    response, err := a.Chat(ctx, item)
    if err != nil {
        return state, err
    }
    state.Results = append(state.Results, response.Content())
    return state, nil
}

cfg := config.DefaultChainConfig()
result, err := workflows.ProcessChain(ctx, cfg, items, MyState{}, processor, nil)
```

### Parallel Execution

```go
processor := func(ctx context.Context, item string) (string, error) {
    response, err := a.Chat(ctx, item)
    if err != nil {
        return "", err
    }
    return response.Content(), nil
}

cfg := config.DefaultParallelConfig()
result, err := workflows.ProcessParallel(ctx, cfg, items, processor, nil)
// result.Results preserves original order
```

### Conditional Routing

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

### Integration Helpers

```go
analysisNode := workflows.ChainNode(chainCfg, sections, analyzeSection, nil)
reviewNode := workflows.ParallelNode(parallelCfg, reviewers, reviewDoc, nil, aggregate)
routingNode := workflows.ConditionalNode(conditionalCfg, checkConsensus, routes)
```

## Observability

```go
// Built-in: NoOpObserver, SlogObserver, MultiObserver
observability.RegisterObserver("custom", myObserver)

cfg := config.DefaultGraphConfig()
cfg.Observer = "slog"  // or "noop"
```

## Checkpointing

```go
store := state.NewMemoryCheckpointStore()
state.RegisterCheckpointStore("memory", store)

cfg := config.GraphConfig{
    Checkpoint: config.CheckpointConfig{
        Store:    "memory",
        Interval: 1,
        Preserve: false,
    },
}

// Resume from checkpoint
finalState, err := graph.Resume(ctx, runID)
```

## Configuration Defaults

```go
config.DefaultHubConfig()          // Buffer: 100, Timeout: 30s
config.DefaultGraphConfig()        // MaxIterations: 100, Observer: "slog"
config.DefaultChainConfig()        // CaptureIntermediateStates: false
config.DefaultParallelConfig()     // Workers: auto, FailFast: true
config.DefaultConditionalConfig()  // Observer: "slog"
```
