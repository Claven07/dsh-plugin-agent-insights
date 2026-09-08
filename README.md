# dsh-plugin-agent-insights

[![npm version](https://img.shields.io/npm/v/dsh-plugin-agent-insights.svg)](https://www.npmjs.com/package/dsh-plugin-agent-insights)
[![license](https://img.shields.io/github/license/Claven07/dsh-plugin-agent-insights.svg)](./LICENSE)

Session-scoped observability metrics and performance tracking plugin for the DeepSeek Harness ecosystem.

---

## Overview

`dsh-plugin-agent-insights` provides lightweight, non-invasive, session-scoped observability and performance tracking for DeepSeek Harness agents. It registers as a Cordis service (`ctx.agentInsights`) and automatically hooks into runtime dispatch pipelines to collect granular metrics without modifying any core packages or interfering with agent execution.

### Key Capabilities

- **Tool Execution Metrics**: Captures total tool calls, successful calls, failed calls, precise execution duration via `performance.now()`, and identifies the slowest tool invocation.
- **LLM & Agent Metrics**: Tracks total agent steps closed, total model requests dispatched, and failed model requests.
- **Session Duration Metrics**: Records session start timestamps and total elapsed session duration.
- **Global Slowest Tool Tracking**: Tracks the single slowest tool invocation across all active sessions without retaining references to the sessions.
- **Zero-Leak Memory Architecture**: Stores session statistics in a `WeakMap<Session, SessionAgentInsights>`, preventing memory retention of completed sessions and performing deterministic cleanup on `session/disposed`.
- **Configurable Disposal Logging**: Automatically outputs a concise, formatted summary to the Cordis logger when a session terminates (`logSummaryOnDisposed`).
- **Fail-Safe Isolation**: All internal telemetry collection is defensively isolated with internal error boundaries; telemetry failures will never crash the agent loop or fail a tool dispatch.

---

## Features

| Category | Metric / Feature | Description |
| :--- | :--- | :--- |
| **Tools** | `totalToolCalls` | Total tool dispatches initiated by the agent |
| | `successfulToolCalls` | Tool dispatches that completed without errors |
| | `failedToolCalls` | Tool dispatches that threw or returned an error result |
| | `durationMs` | High-resolution execution time measured per tool call |
| | `slowestTool` | Slowest tool invocation recorded for the session |
| **Agent** | `totalSteps` | Total agent steps closed (`step/end` events) |
| | `totalLlmRequests` | Model requests initiated (`agent/request` waterfall) |
| | `failedLlmRequests` | Model requests that failed (`agent/request-error` waterfall) |
| **Session** | `sessionStartTime` | Epoch timestamp (ms) when the session was created |
| | `sessionDurationMs` | Total elapsed duration of the active session |
| **Global** | `getSlowestTool()` | Slowest tool call across all sessions without session retention |
| **Lifecycle** | Memory management | `WeakMap` storage + explicit cleanup on `session/disposed` |

---

## Installation

### From npm

Install the package into your project or DeepSeek Harness workspace:

```bash
npm install dsh-plugin-agent-insights
# or with pnpm
pnpm add dsh-plugin-agent-insights
```

### From GitHub

You can also install directly from GitHub:

```bash
npm install github:Claven07/dsh-plugin-agent-insights
# or with pnpm
pnpm add github:Claven07/dsh-plugin-agent-insights
```

> [!IMPORTANT]
> **pnpm Git Build Requirement (`onlyBuiltDependencies`):**
> When installing directly from GitHub via `pnpm`, pnpm restricts lifecycle build scripts (`prepare`) by default for security. To allow pnpm to compile the package on install, ensure `dsh-plugin-agent-insights` is included in your root `package.json`:
>
> ```json
> {
>   "pnpm": {
>     "onlyBuiltDependencies": [
>       "dsh-plugin-agent-insights"
>     ]
>   }
> }
> ```
> Alternatively, enable builds with `pnpm config set allow-builds true` or build the package after cloning.

---

## DeepSeek Harness Profile Integration

### Adding to a Profile

To add `dsh-plugin-agent-insights` to a DeepSeek Harness profile via the CLI:

```bash
dsh plugin --profile <profile-name> add dsh-plugin-agent-insights
```

The plugin exports a bundle patch (`cordis.patch.yml`) that is automatically recognized and applied to your profile.

### Manual Configuration (`cordis.yml` / `cordis.patch.yml`)

You can also declare the plugin directly in your Cordis configuration file:

```yaml
- insert:
    - id: agent-insights
      name: dsh-plugin-agent-insights
      config:
        maxToolHistoryPerSession: 1000
        logSummaryOnDisposed: true
```

---

## Programmatic Usage

Mount the plugin onto your Cordis `Context`:

```typescript
import { Context } from '@deepseek-ai/cordis'
import AgentInsights from 'dsh-plugin-agent-insights'

const ctx = new Context()

// Mount plugin with optional configuration
await ctx.plugin(AgentInsights, {
  maxToolHistoryPerSession: 500,
  logSummaryOnDisposed: true,
})

// Query metrics for an active session
const metrics = ctx.agentInsights.getMetrics(session)

console.log(`Session ID: ${metrics.sessionId}`)
console.log(`Steps: ${metrics.totalSteps}`)
console.log(`LLM Requests: ${metrics.totalLlmRequests} (${metrics.failedLlmRequests} failed)`)
console.log(`Tool Calls: ${metrics.totalToolCalls} (${metrics.successfulToolCalls} ok, ${metrics.failedToolCalls} failed)`)

if (metrics.slowestTool) {
  console.log(`Slowest Tool: ${metrics.slowestTool.name} (${metrics.slowestTool.durationMs.toFixed(2)}ms)`)
}

console.log(`Session Duration: ${(metrics.sessionDurationMs / 1000).toFixed(2)}s`)
```

### Finding the Global Slowest Tool

```typescript
// Query the slowest tool call across all sessions
const globalSlowest = ctx.agentInsights.getSlowestTool()
if (globalSlowest) {
  console.log(`Global slowest tool: ${globalSlowest.name} took ${globalSlowest.durationMs.toFixed(2)}ms`)
}
```

### Resetting Metrics

```typescript
// Reset metrics for a specific session
ctx.agentInsights.reset(session)

// Or reset all tracked metrics globally
ctx.agentInsights.reset()
```

---

## Configuration Options

The plugin accepts an `AgentInsightsConfig` object (validated with `@deepseek-ai/schemastery` schema):

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `maxToolHistoryPerSession` | `number` | `1000` | Maximum number of tool execution records retained in memory per session using a sliding FIFO window. |
| `logSummaryOnDisposed` | `boolean` | `true` | When `true`, automatically logs a formatted summary line to `ctx.logger.info` when a session terminates. |

---

## Lifecycle & Memory Architecture

```
                       ┌────────────────────────────┐
                       │   DeepSeek Harness Loop    │
                       └──────────────┬─────────────┘
                                      │
     ┌──────────────────┬─────────────┼─────────────┬──────────────────┐
     ▼                  ▼             ▼             ▼                  ▼
tools/execute      agent/request  agent/error  session/event    session/disposed
(around hook)       (waterfall)   (waterfall)  (step/end event) (disposal edge)
     │                  │             │             │                  │
     ▼                  ▼             ▼             ▼                  ▼
[Measure duration  [Increment    [Increment    [Increment step   [Log summary &
 & tool outcome]    total LLM]    failed LLM]   & duration]       evict WeakMap]
     │                                                                 │
     └──────────────────────────────┬──────────────────────────────────┘
                                    ▼
                     WeakMap<Session, SessionAgentInsights>
                        (Zero-leak memory storage)
```

1. **Weak Reference Storage**: Session metrics are keyed by `Session` in a `WeakMap`. No strong references are held, allowing the V8 engine to garbage-collect sessions if they fall out of scope.
2. **Deterministic Disposal**: Upon the `session/disposed` lifecycle event, entries are explicitly evicted from the `WeakMap`.
3. **Disposal Summary**: If `logSummaryOnDisposed` is enabled, a formatted summary is emitted to the logger:
   ```text
   [agent-insights] Session "session-123" completed: 12 steps, 5 LLM requests (0 failed), 8 tool calls (0 failed), duration 4.32s (slowest tool: web_search [1234.5ms])
   ```
4. **Global Slowest Tool Safety**: The global slowest tool record captures only primitive values (`callId`, `name`, `durationMs`, `isError`, `errorMessage`) and does not retain references to the `Session` or `Agent` objects.

---

## Public API Reference

The plugin attaches to Cordis context under `ctx.agentInsights`:

### `getMetrics(session: Session): SessionAgentInsights`
Returns a deep clone of the metric snapshot collected for the specified session, including calculated live session duration.

### `getSlowestTool(session?: Session): ToolExecutionMetric | undefined`
- If `session` is provided, returns the slowest tool execution record within that session.
- If `session` is omitted, returns the slowest tool execution record recorded globally across all sessions.

### `reset(session?: Session): void`
- If `session` is provided, removes the session's recorded metrics.
- If `session` is omitted, resets all recorded sessions and clears the global slowest tool metric.

---

## Standalone Development

To develop, test, or build `dsh-plugin-agent-insights` independently:

```bash
# Install dependencies
npm install

# Run TypeScript type verification without emitting files
npm run typecheck

# Execute Vitest test suite
npm test

# Compile TypeScript to lib/
npm run build

# Clean compiled artifacts
npm run clean

# Validate package before publishing
npm run prepublishOnly
```

---

## License

[MIT](./LICENSE) &copy; 2026 Omansh Bhatnagar
