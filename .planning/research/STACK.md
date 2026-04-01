# Technology Stack: Debugging Async Memory Persistence

**Project:** opencode-mem0  
**Researched:** 2025-04-01  
**Domain:** Async TypeScript API debugging, observability, and reliability patterns

---

## Executive Summary

The core issue in opencode-mem0 is **silent async failures**: memory operations fail but errors aren't visible to users. The plugin uses Bun's native `fetch()` with a 30s timeout and basic logging, but lacks structured observability, retry logic, and user-facing error feedback.

**Key Insight:** The current implementation catches errors and returns `{ success: false, error }`, but these results are consumed by the mem0 tool which returns JSON to the LLM. The user never sees failures unless they explicitly check tool results—which LLMs rarely do.

---

## Recommended Stack

### 1. Structured Logging (HIGH Priority)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **Pino** | ^9.0.0 | Structured JSON logging | Bun-compatible, zero overhead in production, child loggers for request context |
| **pino-pretty** | ^13.0.0 | Development log formatting | Human-readable output during debugging |

**Why Pino over console.log:**
- Structured JSON logs enable filtering/searching by fields (e.g., `operation:addMemory`, `success:false`)
- Child loggers automatically inject context (memoryId, scope, duration)
- Bun-native compatibility (no polyfills needed)
- Log levels (debug/info/warn/error) allow runtime verbosity control

**Implementation Pattern:**
```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: { service: 'opencode-mem0' }
});

// Create child logger per memory operation
const opLogger = logger.child({ 
  operation: 'addMemory', 
  scope: scope.projectId ? 'project' : 'user',
  contentLength: content.length 
});

opLogger.info('Starting memory add');
// ... operation ...
opLogger.info({ durationMs: Date.now() - start, memoryId: result.id }, 'Memory add complete');
```

**Confidence:** HIGH — Pino is the industry standard, verified via Context7 docs

---

### 2. Async Debugging & Stack Traces (HIGH Priority)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **Bun Inspector** | Built-in | Interactive debugging | Native WebKit protocol, no external deps |
| **Async Stack Traces** | Bun 1.1.30+ | Full async call chains | PR #22517 merged, enables async frame tracking |

**Bun-Specific Debugging Tools:**

1. **`--inspect` flag** for interactive debugging:
   ```bash
   bun --inspect ./dist/index.js
   # Opens debug.bun.sh web interface
   ```

2. **`BUN_CONFIG_VERBOSE_FETCH`** for HTTP debugging:
   ```bash
   # Print all fetch requests as curl commands
   BUN_CONFIG_VERBOSE_FETCH=curl bun run ./dist/index.js
   
   # Or print request/response details
   BUN_CONFIG_VERBOSE_FETCH=true bun run ./dist/index.js
   ```

3. **Async Stack Traces** (enabled by default in recent Bun):
   Bun now includes async prefix in stack traces, making it easier to trace failures across async boundaries.

**When to Use:**
- Development: Use `--inspect-brk` to pause on first line and trace memory operations
- Production debugging: Enable `BUN_CONFIG_VERBOSE_FETCH=true` to capture raw HTTP traffic
- CI/CD: Capture logs with structured JSON for analysis

**Confidence:** HIGH — Bun official documentation and merged PR #22517

---

### 3. HTTP Resilience Layer (MEDIUM Priority)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **fetch-retry-ts** | ^1.3.1 | Retry wrapper for fetch | Zero deps, TypeScript-native, configurable backoff |
| **Custom Circuit Breaker** | N/A | Fail-fast after repeated failures | Prevent cascading failures when Mem0 API is down |

**Why Add Retry Logic:**
Current implementation has no retry on transient failures (network blips, 503s, 429 rate limits). A single failed request = lost memory.

**Recommended Configuration:**
```typescript
import fetchRetry from 'fetch-retry-ts';

const fetchWithRetry = fetchRetry(fetch, {
  retries: 3,
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 10000), // Exponential backoff, max 10s
  retryOn: [429, 500, 502, 503, 504, 'ECONNRESET', 'ETIMEDOUT'],
});
```

**Circuit Breaker Pattern:**
After 5 consecutive failures, enter "open" state for 30s and fail fast. This prevents:
- Hanging requests during Mem0 outages
- Wasted API calls that will likely fail
- User confusion from long delays

**Confidence:** MEDIUM — fetch-retry-ts is mature but adding new deps requires validation

---

### 4. Observability & Tracing (MEDIUM Priority)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **OpenTelemetry JS** | ^2.0.0 | Distributed tracing | Track memory operations across async boundaries |
| **@opentelemetry/resources** | ^2.0.0 | Resource attributes | Identify plugin vs core system |

**Why OpenTelemetry:**
- Traces show the full lifecycle: trigger detection → API call → response → result handling
- Spans can track timing, success/failure, and metadata
- Compatible with Jaeger, Zipkin, or cloud APM tools

**Implementation Pattern:**
```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('opencode-mem0');

async function addMemory(content, scope) {
  return tracer.startActiveSpan('addMemory', async (span) => {
    span.setAttribute('memory.scope', scope.projectId ? 'project' : 'user');
    span.setAttribute('memory.content_length', content.length);
    
    try {
      const result = await fetch(...);
      span.setAttribute('memory.success', result.success);
      if (result.id) span.setAttribute('memory.id', result.id);
      return result;
    } catch (error) {
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

**Confidence:** MEDIUM — OpenTelemetry is standard but adds complexity; verify need before implementing

---

### 5. Error Visibility Layer (HIGH Priority)

| Pattern | Implementation | Purpose |
|---------|---------------|---------|
| **In-Memory Operation Log** | Ring buffer (last 50 ops) | Debug why recent memories failed |
| **Synchronous Validation** | Pre-flight checks | Fail fast on config/auth errors |
| **Health Check Endpoint** | Periodic Mem0 API ping | Detect systemic issues early |

**Critical Gap Identified:**
The current `log()` function writes to OpenCode's internal log stream, which users rarely see. Failed memory operations need **explicit user feedback**.

**Recommended: In-Memory Debug Log**
```typescript
// services/debug-log.ts
const MAX_LOG_ENTRIES = 50;
const operationLog: OperationLogEntry[] = [];

export function logOperation(entry: OperationLogEntry) {
  operationLog.unshift(entry);
  if (operationLog.length > MAX_LOG_ENTRIES) operationLog.pop();
}

// Add to tool: mem0 mode=debug shows recent operations
```

**Confidence:** HIGH — Pattern derived from codebase analysis, no external deps needed

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| HTTP Client | Native `fetch()` | Undici | Bun's fetch is already optimized; Undici adds bundle size |
| Logging | Pino | Winston | Winston has heavier deps; Pino is Bun-optimized |
| Tracing | OpenTelemetry | DIY spans | OTel is industry standard; DIY creates maintenance burden |
| Retries | fetch-retry-ts | Custom implementation | Custom retry logic is error-prone; library is well-tested |

---

## Installation

```bash
# Core dependencies
bun add pino

# Development dependencies
bun add -D pino-pretty

# Optional: retry and observability
bun add fetch-retry-ts @opentelemetry/api @opentelemetry/sdk-trace-base
```

---

## Anti-Patterns to Avoid

### 1. Silent Failures (CRITICAL)
**What:** Catching errors and only logging to internal logger
**Why it's bad:** Users never know memories failed to save
**Current code:**
```typescript
// BAD: Error swallowed
} catch (error) {
  log("addMemory: error", { error: errorMessage });
  return { success: false, error: errorMessage }; // User never sees this
}
```
**Instead:** Return error in tool result AND add to operation log

### 2. No Request Timeouts (CRITICAL)
**What:** Relying solely on Bun's default fetch timeout
**Why it's bad:** Default timeout may be too long; hanging requests block UI
**Current code:**
```typescript
// Has timeout but no retry
timeout: TIMEOUT_MS // 30000ms
```
**Instead:** Implement circuit breaker for repeated failures

### 3. Console.log in Production (MODERATE)
**What:** Using console.log for operational logging
**Why it's bad:** Unstructured, can't filter by level, no child context
**Instead:** Use Pino with JSON output in production

### 4. No Idempotency (MODERATE)
**What:** Retrying POST requests without idempotency keys
**Why it's bad:** Duplicate memories on retry
**Mitigation:** Mem0's API may handle duplicates; verify behavior before implementing client-side deduplication

### 5. Missing Pre-Flight Validation (MODERATE)
**What:** Not validating API key/config before operations
**Why it's bad:** First failure happens at runtime, not startup
**Instead:** Validate configuration on plugin init and surface errors early

---

## Implementation Priority

1. **P0 (This Week):** 
   - Add structured logging with Pino
   - Implement in-memory operation log for debugging
   - Add `BUN_CONFIG_VERBOSE_FETCH` documentation for users

2. **P1 (Next Sprint):**
   - Add retry logic with exponential backoff
   - Implement circuit breaker for Mem0 API
   - Add health check command to mem0 tool

3. **P2 (Future):**
   - OpenTelemetry tracing for detailed operation tracking
   - Metrics collection (success rate, latency percentiles)

---

## Sources

| Source | Confidence | Relevance |
|--------|------------|-----------|
| Bun Debugging Docs | HIGH | Native debugging capabilities |
| Bun PR #22517 (async stack traces) | HIGH | Async debugging improvements |
| Pino Context7 Documentation | HIGH | Structured logging patterns |
| Undici RetryAgent Context7 | MEDIUM | Retry pattern implementation |
| Exa Search: TypeScript async debugging 2025 | MEDIUM | Current best practices |
| Mem0 GitHub Issues (#3901, #4055) | HIGH | API-specific error patterns |
| fetch-retry-ts GitHub | MEDIUM | Retry library evaluation |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Bun Debugging | HIGH | Official docs, verified features |
| Pino Logging | HIGH | Context7-verified, industry standard |
| Retry Patterns | MEDIUM | Standard patterns, library choice needs validation |
| OpenTelemetry | MEDIUM | Standard but complex; verify value before implementing |
| Mem0 API Behavior | MEDIUM | Limited docs; derived from GitHub issues and code analysis |
