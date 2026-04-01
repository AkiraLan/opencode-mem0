# Architecture Patterns for Async Memory Operation Observability

**Project:** opencode-mem0
**Research Date:** 2026-04-01
**Domain:** OpenCode Plugin System with Async Memory Backend (Mem0 Platform API)

---

## Executive Summary

The opencode-mem0 plugin faces a critical observability challenge: **async memory operations may fail silently**, leading to the user-reported issue where "remember xxx" commands appear to succeed but memories don't persist. This document outlines architectural patterns to add debugging capabilities, observability, and reliability guarantees to the existing async plugin architecture without major refactoring.

**Core Problem:** The current implementation uses fire-and-forget async patterns where:
1. API failures are caught, logged to file, and swallowed
2. No correlation IDs link user actions to backend operations
3. No health checks verify backend connectivity
4. No user-facing feedback confirms operation success/failure
5. Compaction logic is model-specific (Claude only), leaving non-Claude users without context preservation

**Recommended Approach:** Implement a layered observability strategy using correlation IDs, structured logging, health checks, circuit breakers, and user-facing operation receipts—all compatible with the existing singleton client pattern and hook-based architecture.

---

## Current Architecture Analysis

### Component Boundaries

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenCode Plugin Host                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ chat.message │  │    tool      │  │    event     │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
└─────────┼─────────────────┼─────────────────┼────────────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenMemoryPlugin (src/index.ts)                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Hook Handlers                                               │    │
│  │  • Memory keyword detection (regex) → Nudges agent to use    │    │
│  │    mem0 tool (FIRE-AND-FORGET: no confirmation)              │    │
│  │  • First-message context injection → Parallel fetch profile  │    │
│  │    + memories + project memories                             │    │
│  │  • Tool registration (mem0) → Async CRUD operations          │    │
│  │  • Event handling → Compaction trigger (Claude-only)         │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 Mem0RESTClient Singleton (src/services/client.ts)    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ searchMemories│  │  addMemory   │  │ listMemories │                │
│  └────────┬─────┘  └──────┬───────┘  └──────┬───────┘                │
│           │               │                 │                        │
│           └───────────────┼─────────────────┘                        │
│                           ▼                                          │
│              ┌─────────────────────┐                                 │
│              │  fetch() wrapper    │                                 │
│              │  • 30s timeout      │                                 │
│              │  • redirect handling│                                 │
│              │  • Error → log file │ ← SILENT FAILURE PATTERN        │
│              └─────────────────────┘                                 │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ HTTPS
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Mem0 Platform API (api.mem0.ai)                     │
│  • Embeddings & vector search                                       │
│  • Hierarchical Semantic Graph                                      │
│  • Scoped memory namespaces (user_id)                               │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow with Observability Gaps

**Current "Remember" Flow (Problematic):**
```
User: "Remember to use Prettier with single quotes"
    │
    ▼
┌────────────────────────┐
│ detectMemoryKeyword()  │ ──Yes──▶ Inject nudge to agent
└────────────────────────┘          ("You MUST use mem0 tool...")
    │
    ▼
Agent calls mem0 tool with mode="add"
    │
    ▼
┌────────────────────────┐
│ addMemory() in client  │ ──Async──▶ POST /v1/memories
└────────────────────────┘              │
    │                                   │
    │ Success?                          │ Failure?
    │                                   │
    │ Return {success: true, id}        │ Return {success: false, error}
    │                                   │
    ▼                                   ▼
┌────────────────────────┐      ┌────────────────────────┐
│ User sees success JSON │      │ Error logged to file   │
│ in tool output         │      │ User may not notice    │
└────────────────────────┘      └────────────────────────┘
    │
    ▼
User assumes memory saved
But has NO WAY TO VERIFY
```

**Key Observability Gaps:**

| Gap | Impact | Detection Difficulty |
|-----|--------|---------------------|
| No operation correlation ID | Cannot trace "remember" → API call → confirmation | Hard |
| Silent API failures | User thinks memory saved, but it's not | Very Hard |
| No health check | Don't know if Mem0 is reachable until operation fails | Medium |
| No retry mechanism | Transient failures become permanent data loss | Medium |
| No operation audit log | Cannot diagnose "where did my memory go?" | Hard |
| Model-specific compaction | Non-Claude users lose context silently | Hard |

---

## Recommended Architecture Patterns

### Pattern 1: Operation Correlation Context

**What:** Generate unique operation IDs that propagate through the entire async chain, enabling end-to-end tracing of memory operations.

**Why:** Currently, there's no way to correlate a user's "remember" command with the resulting API call and its outcome. A correlation ID enables debugging and user feedback.

**Implementation Strategy:**

```typescript
// New: Operation context that flows through async operations
interface OperationContext {
  operationId: string;      // UUID for this specific operation
  parentOperationId?: string; // For nested operations
  sessionId: string;        // OpenCode session ID
  timestamp: number;        // Operation start time
  operation: string;        // "addMemory", "searchMemories", etc.
  scope: MemoryScopeContext;
}

// Enhanced result type with correlation
interface TracedResult<T> extends OperationResult<T> {
  operationId: string;
  duration: number;
  attempts: number;
  trace: OperationTrace[];
}

// Add to existing methods
async addMemory(
  content: string,
  scope: MemoryScopeContext,
  options?: AddMemoryOptions,
  context?: Partial<OperationContext>  // NEW: Optional correlation context
): Promise<TracedResult<AddMemoryResult>>
```

**Build Order Implication:** Add correlation infrastructure first—it's a cross-cutting concern that enables all other observability features.

---

### Pattern 2: Structured Operation Logging

**What:** Replace simple file logging with structured, queryable operation logs that include correlation IDs, timing, and outcomes.

**Why:** Current logging (`log()` writes to `~/.opencode-supermemory.log`) is unstructured and hard to query. Structured logs enable debugging "why didn't my memory save?"

**Implementation Strategy:**

```typescript
// Enhanced logger with operation context
interface LogEntry {
  timestamp: string;
  level: 'debug' | 'info' | 'warn' | 'error';
  operationId: string;
  component: string;        // "client", "compaction", "tool"
  action: string;           // "addMemory.start", "addMemory.success", "addMemory.error"
  duration?: number;
  metadata: Record<string, unknown>;
  error?: {
    message: string;
    code?: string;
    stack?: string;
  };
}

// Usage in client operations
async addMemory(...) {
  const operationId = generateOperationId();
  const startTime = Date.now();
  
  logger.info({
    operationId,
    component: 'client',
    action: 'addMemory.start',
    metadata: { contentLength: content.length, scope }
  });
  
  try {
    const result = await this.executeAdd(...);
    
    logger.info({
      operationId,
      component: 'client',
      action: 'addMemory.success',
      duration: Date.now() - startTime,
      metadata: { memoryId: result.id }
    });
    
    return { ...result, operationId, duration: Date.now() - startTime };
  } catch (error) {
    logger.error({
      operationId,
      component: 'client',
      action: 'addMemory.error',
      duration: Date.now() - startTime,
      error: { message: error.message, code: error.code }
    });
    throw error;
  }
}
```

**Build Order Implication:** Implement after correlation IDs, as logs need operation context to be useful.

---

### Pattern 3: Health Check & Circuit Breaker

**What:** Proactive health checks verify Mem0 API availability before operations. Circuit breaker pattern prevents cascading failures when the backend is down.

**Why:** Currently, every operation attempts a 30s timeout fetch. If Mem0 is down, all operations hang and fail individually. A circuit breaker fails fast when the service is unhealthy.

**Implementation Strategy:**

```typescript
// Circuit breaker states
enum CircuitState {
  CLOSED = 'CLOSED',       // Normal operation
  OPEN = 'OPEN',          // Failing fast
  HALF_OPEN = 'HALF_OPEN' // Testing if recovered
}

interface CircuitBreakerConfig {
  failureThreshold: number;      // Failures before opening
  resetTimeout: number;          // Time before half-open
  halfOpenMaxCalls: number;      // Test calls in half-open
}

class Mem0CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private lastFailureTime?: number;
  private halfOpenCalls = 0;
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - (this.lastFailureTime || 0) > this.config.resetTimeout) {
        this.state = CircuitState.HALF_OPEN;
        this.halfOpenCalls = 0;
      } else {
        throw new CircuitOpenError('Mem0 API temporarily unavailable');
      }
    }
    
    if (this.state === CircuitState.HALF_OPEN && 
        this.halfOpenCalls >= this.config.halfOpenMaxCalls) {
      throw new CircuitOpenError('Too many test calls while half-open');
    }
    
    try {
      if (this.state === CircuitState.HALF_OPEN) {
        this.halfOpenCalls++;
      }
      
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess() {
    this.failureCount = 0;
    if (this.state === CircuitState.HALF_OPEN) {
      this.state = CircuitState.CLOSED;
    }
  }
  
  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.config.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
  }
  
  // Health check endpoint
  async healthCheck(): Promise<HealthStatus> {
    try {
      // Lightweight ping to Mem0
      const response = await fetch(`${this.baseUrl}/health`, { 
        method: 'GET',
        timeout: 5000 
      });
      return { healthy: response.ok, latency: ..., timestamp: ... };
    } catch {
      return { healthy: false, error: 'Unreachable' };
    }
  }
}
```

**Build Order Implication:** Implement after structured logging, as the circuit breaker should log state transitions.

---

### Pattern 4: Operation Receipts (User-Facing Observability)

**What:** Provide users with confirmation receipts for memory operations, including operation ID and verification steps.

**Why:** Users currently have no feedback that a "remember" command actually persisted. A receipt pattern gives them confidence and debugging information.

**Implementation Strategy:**

```typescript
// Enhanced tool response with receipt
interface MemoryReceipt {
  operationId: string;
  status: 'confirmed' | 'pending' | 'failed';
  memoryId?: string;
  timestamp: number;
  content: string;
  scope: string;
  verificationSteps: string[];  // How to verify this worked
}

// In tool handler (src/index.ts)
case "add": {
  const result = await openMemoryClient.addMemory(...);
  
  if (!result.success) {
    return JSON.stringify({
      success: false,
      error: result.error,
      receipt: {
        operationId: result.operationId,
        status: 'failed',
        timestamp: Date.now(),
        troubleshooting: [
          "Check your Mem0 API key configuration",
          "Verify network connectivity to api.mem0.ai",
          `Search logs: grep "${result.operationId}" ~/.opencode-supermemory.log`
        ]
      }
    });
  }
  
  // Verify the memory was actually saved (read-after-write)
  const verifyResult = await openMemoryClient.searchMemories(
    content.slice(0, 50), // Search for beginning of content
    scope,
    { limit: 5 }
  );
  
  const found = verifyResult.results?.some(m => m.id === result.id);
  
  return JSON.stringify({
    success: true,
    receipt: {
      operationId: result.operationId,
      status: found ? 'confirmed' : 'pending',
      memoryId: result.id,
      timestamp: Date.now(),
      scope: args.scope || "project",
      verificationSteps: found 
        ? ["Memory confirmed in search results"]
        : [
            "Memory ID assigned but not yet searchable",
            "This is normal - Mem0 indexing may take a few seconds",
            `To verify: mem0 search "${content.slice(0, 30)}..."`
          ]
    }
  });
}
```

**Build Order Implication:** Implement after correlation IDs and structured logging, as receipts rely on operation tracing.

---

### Pattern 5: Async Operation Verification Queue

**What:** Background verification that async operations actually completed, with retry for transient failures.

**Why:** Mem0 API may accept a write but fail to index it. A verification queue confirms persistence and retries if needed.

**Implementation Strategy:**

```typescript
// Pending operation tracking
interface PendingOperation {
  operationId: string;
  type: 'add' | 'delete';
  content: string;
  scope: MemoryScopeContext;
  attempts: number;
  maxAttempts: number;
  createdAt: number;
  verifiedAt?: number;
}

class OperationVerificationQueue {
  private pending = new Map<string, PendingOperation>();
  private verifyInterval: NodeJS.Timeout;
  
  constructor(private client: Mem0RESTClient) {
    // Verify pending operations every 5 seconds
    this.verifyInterval = setInterval(() => this.processQueue(), 5000);
  }
  
  add(operation: PendingOperation) {
    this.pending.set(operation.operationId, operation);
  }
  
  private async processQueue() {
    for (const [id, op] of this.pending) {
      if (op.attempts >= op.maxAttempts) {
        logger.error({ operationId: id, action: 'verification.maxAttempts' });
        this.pending.delete(id);
        continue;
      }
      
      try {
        const verified = await this.verifyOperation(op);
        
        if (verified) {
          logger.info({ operationId: id, action: 'verification.success' });
          this.pending.delete(id);
        } else {
          op.attempts++;
          
          if (op.type === 'add' && op.attempts < op.maxAttempts) {
            // Retry the add
            await this.client.addMemory(op.content, op.scope);
          }
        }
      } catch (error) {
        op.attempts++;
        logger.warn({ operationId: id, action: 'verification.error', error });
      }
    }
  }
  
  private async verifyOperation(op: PendingOperation): Promise<boolean> {
    if (op.type === 'add') {
      // Search for the content
      const results = await this.client.searchMemories(
        op.content.slice(0, 100),
        op.scope,
        { limit: 10 }
      );
      return results.results?.some(r => 
        r.content.includes(op.content.slice(0, 50))
      ) ?? false;
    }
    return true;
  }
}
```

**Build Order Implication:** Implement after basic operation tracing, as it's an enhancement to existing operations.

---

### Pattern 6: Telemetry Dashboard (Optional Enhancement)

**What:** In-memory or file-based telemetry that tracks operation metrics over time.

**Why:** Enables identifying patterns like "add operations fail 20% of the time between 2-4 AM" or "user scope has higher failure rate than project scope."

**Implementation Strategy:**

```typescript
interface TelemetryMetrics {
  operations: {
    total: number;
    success: number;
    failure: number;
    byType: Record<string, { success: number; failure: number }>;
    byScope: Record<string, { success: number; failure: number }>;
    latencyBuckets: number[];  // Histogram of operation durations
  };
  errors: {
    byCode: Record<string, number>;
    recent: Array<{ timestamp: number; message: string; operationId: string }>;
  };
  health: {
    lastCheck: number;
    status: 'healthy' | 'degraded' | 'unhealthy';
    uptime: number;
  };
}

class TelemetryCollector {
  private metrics: TelemetryMetrics;
  
  recordOperation(result: TracedResult<unknown>) {
    this.metrics.operations.total++;
    
    if (result.success) {
      this.metrics.operations.success++;
    } else {
      this.metrics.operations.failure++;
      this.metrics.errors.recent.push({
        timestamp: Date.now(),
        message: result.error,
        operationId: result.operationId
      });
    }
    
    // Update latency buckets
    const bucket = Math.floor(result.duration / 100); // 100ms buckets
    this.metrics.operations.latencyBuckets[bucket] = 
      (this.metrics.operations.latencyBuckets[bucket] || 0) + 1;
  }
  
  getSummary(): string {
    const successRate = this.metrics.operations.total > 0
      ? (this.metrics.operations.success / this.metrics.operations.total * 100).toFixed(1)
      : 'N/A';
      
    return `Mem0 Plugin Telemetry:
- Total Operations: ${this.metrics.operations.total}
- Success Rate: ${successRate}%
- Average Latency: ${this.calculateAverageLatency()}ms
- Health Status: ${this.metrics.health.status}
- Recent Errors: ${this.metrics.errors.recent.length}`;
  }
}

// Expose via tool command
case "status": {
  return JSON.stringify({
    success: true,
    telemetry: telemetryCollector.getSummary(),
    health: await healthChecker.getStatus(),
    circuitBreaker: circuitBreaker.getState(),
    pendingVerifications: verificationQueue.getPendingCount()
  });
}
```

**Build Order Implication:** Last to implement—builds on all other observability features.

---

## Suggested Build Order

Based on dependencies between patterns:

### Phase 1: Foundation (Correlation & Logging)
**Goal:** Enable tracing of operations

1. **Operation Correlation Context** (Pattern 1)
   - Add `OperationContext` interface
   - Generate operation IDs in all client methods
   - Return operation IDs in results
   - **Unblocks:** All subsequent patterns

2. **Structured Operation Logging** (Pattern 2)
   - Replace `log()` with structured logger
   - Include operationId in all log entries
   - Add log levels and component tagging
   - **Dependencies:** Pattern 1

### Phase 2: Reliability (Health & Circuit Breaking)
**Goal:** Prevent silent failures and cascade errors

3. **Health Check & Circuit Breaker** (Pattern 3)
   - Implement health check endpoint
   - Add circuit breaker wrapper
   - Log state transitions
   - **Dependencies:** Pattern 2

### Phase 3: User Confidence (Receipts & Verification)
**Goal:** Give users visibility into operation outcomes

4. **Operation Receipts** (Pattern 4)
   - Enhance tool responses with receipts
   - Add verification steps
   - Include troubleshooting guidance
   - **Dependencies:** Patterns 1, 2

5. **Async Operation Verification Queue** (Pattern 5)
   - Implement pending operation tracking
   - Add background verification loop
   - Implement retry logic
   - **Dependencies:** Patterns 1, 3

### Phase 4: Insight (Telemetry)
**Goal:** Enable pattern analysis and debugging

6. **Telemetry Dashboard** (Pattern 6)
   - Implement metrics collection
   - Add `mem0 status` tool command
   - Export telemetry summaries
   - **Dependencies:** All previous patterns

---

## Component Boundaries with Observability

### Updated Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenCode Plugin Host                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ chat.message │  │    tool      │  │    event     │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
└─────────┼─────────────────┼─────────────────┼────────────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenMemoryPlugin (src/index.ts)                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Hook Handlers                                               │    │
│  │  • Memory keyword detection → Inject nudge (with opId)       │    │
│  │  • Tool handlers → Return receipts with verification         │    │
│  │  • Event handling → Log with correlation                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Telemetry       │ │ Verification    │ │ Circuit Breaker │
│ Collector       │ │ Queue           │ │ & Health Check  │
│                 │ │                 │ │                 │
│ • Metrics       │ │ • Pending ops   │ │ • Health probe  │
│ • Summaries     │ │ • Retry logic   │ │ • Fail-fast     │
│ • Error tracking│ │ • Confirm save  │ │ • State machine │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 Mem0RESTClient (with observability)                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Operation Context                                           │    │
│  │  • operationId generation                                    │    │
│  │  • timing measurement                                        │    │
│  │  • structured logging                                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Structured Logger → ~/.opencode-supermemory.log (JSON)      │    │
│  │  { operationId, timestamp, level, component, action, ... }   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ HTTPS
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Mem0 Platform API (api.mem0.ai)                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Communication Patterns

| From | To | Data | Purpose |
|------|-----|------|---------|
| `index.ts` (tool) | `client.ts` | OperationContext + args | Initiate traced operation |
| `client.ts` | `logger.ts` | Structured log entry | Record operation lifecycle |
| `client.ts` | `circuitBreaker.ts` | Operation function | Wrapped execution |
| `client.ts` | `verificationQueue.ts` | PendingOperation | Schedule verification |
| `index.ts` (tool) | User | Receipt JSON | Operation confirmation |
| `telemetry.ts` | `index.ts` (tool) | Metrics summary | Status command response |

---

## Data Flow: Enhanced "Remember" with Observability

```
User: "Remember to use Prettier with single quotes"
    │
    ▼
┌────────────────────────┐
│ detectMemoryKeyword()  │
│ Generate operationId-1 │
└────────────────────────┘
    │
    ▼
┌────────────────────────┐
│ Inject nudge + opId    │
└────────────────────────┘
    │
    ▼
Agent calls mem0 tool
    │
    ▼
┌────────────────────────┐     ┌────────────────────────┐
│ START: operationId-2   │────▶│ LOG: addMemory.start   │
│ component: tool        │     │ opId: operationId-2    │
└────────────────────────┘     └────────────────────────┘
    │
    ▼
┌────────────────────────┐
│ Circuit Breaker Check  │
│ State: CLOSED ✓        │
└────────────────────────┘
    │
    ▼
┌────────────────────────┐     ┌────────────────────────┐
│ POST /v1/memories      │────▶│ LOG: api.request       │
│ with operationId-2     │     │ opId: operationId-2    │
└────────────────────────┘     └────────────────────────┘
    │
    ├── Success ──────────────────────────┐
    │                                     ▼
    │                       ┌────────────────────────┐
    │                       │ LOG: addMemory.success │
    │                       │ duration: 245ms        │
    │                       │ memoryId: mem-abc123   │
    │                       └────────────────────────┘
    │                                     │
    │                                     ▼
    │                       ┌────────────────────────┐
    │                       │ Verification Queue     │
    │                       │ Schedule verify in 2s  │
    │                       └────────────────────────┘
    │                                     │
    │                                     ▼
    │                       ┌────────────────────────┐
    │                       │ RETURN RECEIPT:        │
    │                       │ {                      │
    │                       │   operationId: op-2,   │
    │                       │   status: "confirmed", │
    │                       │   memoryId: mem-abc,   │
    │                       │   verificationSteps: [ │
    │                       │     "Memory confirmed" │
    │                       │   ]                    │
    │                       │ }                      │
    │                       └────────────────────────┘
    │
    └── Failure ──────────────────────────┐
                                          ▼
                            ┌────────────────────────┐
                            │ LOG: addMemory.error   │
                            │ error: "HTTP 503"      │
                            │ duration: 30000ms      │
                            └────────────────────────┘
                                          │
                                          ▼
                            ┌────────────────────────┐
                            │ Circuit Breaker        │
                            │ Increment failure      │
                            │ count → Open if > 5    │
                            └────────────────────────┘
                                          │
                                          ▼
                            ┌────────────────────────┐
                            │ RETURN RECEIPT:        │
                            │ {                      │
                            │   operationId: op-2,   │
                            │   status: "failed",    │
                            │   error: "HTTP 503",   │
                            │   troubleshooting: [   │
                            │     "Check API key",   │
                            │     "Search logs:      │
                            │      grep 'op-2' ..."  │
                            │   ]                    │
                            │ }                      │
                            └────────────────────────┘
```

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|------------|--------|
| Pattern Selection | HIGH | Standard observability patterns (correlation IDs, structured logging, circuit breakers) are well-established in distributed systems |
| Implementation Feasibility | HIGH | Patterns can be layered onto existing singleton client without breaking changes |
| Plugin SDK Compatibility | HIGH | All patterns work within OpenCode plugin hook constraints |
| Build Order | MEDIUM | Dependencies between patterns are clear, but actual integration testing needed |
| Performance Impact | MEDIUM | Low overhead for correlation IDs and logging; circuit breaker adds negligible latency |

---

## Sources

- Correlation IDs in distributed systems: [DEV Community - Correlation IDs in ASP.NET Core (2026)](https://dev.to/cristiansifuentes/correlation-ids-in-aspnet-core-designing-observability-like-a-senior-engineer-2026-edition-43c5)
- Context propagation patterns: [KruN - Distributed Tracing Internals](https://krun.pro/distributed-tracing-observability/)
- Circuit breaker implementation: [DEV Community - Circuit Breaker in TypeScript (2026)](https://dev.to/young_gao/implementing-the-circuit-breaker-pattern-in-typescript-182m)
- Background job observability: [Medium - BullMQ Queue Observability (2026)](https://medium.com/@horkaydcuo/stop-flying-blind-bullmq-queue-observability-with-bullstudio-418d6c855849)
- OpenTelemetry context propagation: [OneUptime - OpenTelemetry Context Propagation](https://oneuptime.com/blog/post/2026-02-02-opentelemetry-context-propagation/view)

---

*Architecture research for opencode-mem0 observability improvements*
