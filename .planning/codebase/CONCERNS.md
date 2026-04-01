# Codebase Concerns

**Analysis Date:** 2026-04-01

## Overview

The opencode-mem0 plugin is a relatively small TypeScript codebase (~2,891 lines) that provides persistent memory capabilities for OpenCode agents via the Mem0 Platform API. While functional, several technical concerns exist across error handling, testing, architectural patterns, and operational reliability that could impact maintainability and user experience.

## Detailed Breakdown

### 1. No Test Coverage

**Issue:** Zero test files, test configuration, or testing infrastructure exists.

**Impact:** High - No automated verification of functionality, regression detection, or confidence in changes.

**Files Affected:** All (`src/**/*.ts`)

**Evidence:**
```bash
# No test files found
find . -name "*.test.*" -o -name "*.spec.*" -o -name "vitest.config.*" -o -name "jest.config.*"
# (empty result)
```

**Recommendations:**
- Add Vitest or Bun's built-in test runner
- Prioritize testing `src/services/client.ts` (API interactions) and `src/services/compaction.ts` (complex stateful logic)
- Add integration tests for the plugin lifecycle

---

### 2. Error Handling Inconsistencies

**Issue:** Error handling patterns vary significantly across the codebase - some errors are silently caught, others logged but swallowed, and some propagated incompletely.

**Files Affected:**
- `src/services/client.ts` - Returns error strings instead of throwing
- `src/services/compaction.ts` - Multiple empty catch blocks
- `src/index.ts` - Some errors logged but not surfaced to user

**Examples:**

```typescript
// src/services/client.ts:270-272 - Returns error string, caller may ignore
} catch (error) {
  const errorMessage = error instanceof Error ? error.message : String(error);
  return { success: false, results: [], total: 0, error: errorMessage };
}

// src/services/compaction.ts:547-548 - Empty catch, silent failure
} catch {}

// src/services/compaction.ts:236-238 - Logs but returns false, no retry
} catch (err) {
  log("[compaction] failed to inject hook message", { error: String(err) });
  return false;
}
```

**Recommendations:**
- Establish consistent error handling strategy (exceptions vs result types)
- Always propagate critical errors to users
- Add structured error codes for programmatic handling

---

### 3. Singleton Pattern with Mutable State

**Issue:** `getMemoryClient()` uses a module-level singleton with lazy initialization, making testing difficult and potentially causing issues with configuration changes.

**File:** `src/services/client.ts:479-486`

```typescript
let clientInstance: Mem0RESTClient | null = null;

export function getMemoryClient(): Mem0RESTClient {
  if (!clientInstance) {
    clientInstance = new Mem0RESTClient(); // Reads config at creation time
  }
  return clientInstance;
}
```

**Concerns:**
- Configuration loaded at first use, not at plugin initialization
- No way to reinitialize with different config without restart
- Makes unit testing difficult (requires module mocking or reset capability)

**Recommendations:**
- Inject configuration explicitly through constructor
- Provide reset mechanism for testing
- Consider factory pattern for client creation

---

### 4. Unbounded Memory Growth

**Issue:** The `injectedSessions` Set in `src/index.ts:137` grows indefinitely as new sessions are processed, never being cleaned up.

```typescript
const injectedSessions = new Set<string>(); // Line 137
// ...
if (isFirstMessage) {
  injectedSessions.add(input.sessionID); // Added but never removed
}
```

**Impact:** For long-running OpenCode processes, this could accumulate thousands of session IDs in memory.

**Recommendations:**
- Listen for `session.deleted` events to remove entries
- Use LRU cache with size limit
- Or use WeakSet if session objects are available

---

### 5. Race Conditions in Compaction State

**Issue:** Compaction state management uses synchronous Set operations but async operations without proper locking, risking race conditions during rapid events.

**File:** `src/services/compaction.ts:260-265`

```typescript
const state: CompactionState = {
  lastCompactionTime: new Map(),
  compactionInProgress: new Set(), // No async coordination
  summarizedSessions: new Set(),
};
```

**Scenario:** Multiple concurrent `message.updated` events could pass the `compactionInProgress` check simultaneously before any is added to the set.

**Recommendations:**
- Use async mutex or semaphore for coordination
- Or restructure to use single event queue

---

### 6. Outdated Logging Reference

**Issue:** Log file still references legacy project name "supermemory".

**File:** `src/services/logger.ts:6`

```typescript
const LOG_FILE = join(homedir(), ".opencode-supermemory.log"); // Should be opencode-mem0
```

**Impact:** Confusing for users debugging issues - logs go to unexpected file name.

---

### 7. Hardcoded Model Support

**Issue:** Context compaction only supports Claude models, hardcoded regex pattern.

**File:** `src/services/compaction.ts:16-17, 102-104`

```typescript
const CLAUDE_MODEL_PATTERN = /claude-(opus|sonnet|haiku)/i;

function isSupportedModel(modelID: string): boolean {
  return CLAUDE_MODEL_PATTERN.test(modelID);
}
```

**Impact:** Users with other models (GPT-4, Gemini, etc.) don't get compaction benefits.

**Recommendations:**
- Query model context limits from provider metadata
- Make pattern configurable
- Or support fallback to default limits for unknown models

---

### 8. Git Config Dependency for User Identification

**Issue:** User ID generation depends on git config being set up correctly.

**File:** `src/services/tags.ts:10-26`

```typescript
export function getUserId(): string {
  const email = getGitEmail(); // May return null
  if (email) {
    return sha256(email);
  }
  const fallback = process.env.USER || process.env.USERNAME || "anonymous";
  return sha256(fallback); // Not unique across machines
}
```

**Concerns:**
- Falls back to non-unique identifiers ("anonymous" could be many users)
- No error if git is not configured
- Same user on different machines may get different IDs

**Recommendations:**
- Generate and store persistent UUID in config directory
- Warn user if falling back to non-unique ID
- Document git config requirement

---

### 9. Missing Input Validation

**Issue:** User inputs from configuration and tool arguments lack comprehensive validation.

**Examples:**
- `keywordPatterns` regex validation only happens at use time, not config load
- `limit` parameters in tool calls not validated against reasonable bounds
- Memory content not validated for size limits before API submission

**File:** `src/index.ts:32-42` - Invalid patterns are silently filtered:

```typescript
const customPatterns = CONFIG.keywordPatterns
  .filter((pattern) => pattern.trim().length > 0)
  .map((pattern) => {
    try {
      return new RegExp(pattern, "i");
    } catch {
      log("Invalid keyword pattern ignored", { pattern }); // Silently ignored
      return null;
    }
  })
```

**Recommendations:**
- Validate all config at load time with clear error messages
- Add bounds checking for numeric parameters
- Validate memory content size before API calls

---

### 10. File System Operations Without Atomic Writes

**Issue:** CLI installer writes configuration files directly, risking corruption on interruption.

**File:** `src/cli.ts:206-267` (config modification functions)

```typescript
writeFileSync(configPath, newContent); // Direct write, no atomic operation
```

**Recommendations:**
- Write to temp file then rename for atomicity
- Create backup before modification
- Validate JSON after write

---

### 11. Hardcoded Timeouts Without Configurability

**Issue:** API timeout is hardcoded to 30 seconds with no configuration option.

**File:** `src/services/client.ts:22`

```typescript
const TIMEOUT_MS = 30000;
```

**Impact:** Users on slow networks or with large memory payloads cannot adjust.

**Recommendations:**
- Make configurable via `mem0.jsonc`
- Use different timeouts for different operation types

---

### 12. Duplicate Legacy Entry in Array

**Issue:** `LEGACY_PLUGIN_ENTRIES` contains the same entry twice.

**File:** `src/cli.ts:13-16`

```typescript
const LEGACY_PLUGIN_ENTRIES = [
  "opencode-mem0@latest",
  "opencode-mem0@latest", // Duplicate
];
```

**Impact:** Minor - unnecessary iteration, suggests incomplete cleanup from fork.

---

### 13. Privacy Filtering Bypass Risk

**Issue:** The privacy filtering uses simple regex that could be bypassed with creative formatting.

**File:** `src/services/privacy.ts:1-12`

```typescript
export function containsPrivateTag(content: string): boolean {
  return /<private>[\s\S]*?<\/private>/i.test(content); // Case insensitive
}
```

**Concerns:**
- `<Private>` with different casing might not be caught consistently
- Nested tags not handled
- No size limits on private content

**Recommendations:**
- Document the exact tag format required
- Add validation for private content size
- Consider stricter parsing (e.g., XML-like)

---

### 14. API Version Mismatch

**Issue:** Client uses both v1 and v2 API endpoints without clear rationale.

**File:** `src/services/client.ts`
- Search uses `/v2/memories/search` (line 242)
- Add uses `/v1/memories` (line 283)
- List uses `/v2/memories` (line 349)
- Delete uses `/v1/memories/{id}` (line 389)
- Feedback uses `/v1/feedback` (line 416)

**Concerns:**
- Inconsistent API usage may indicate migration in progress
- Different endpoints may have different behaviors
- Harder to maintain

**Recommendations:**
- Document why each endpoint version is used
- Migrate to consistent version when possible

---

### 15. No Retry Logic for Transient Failures

**Issue:** API calls fail immediately on network errors with no retry mechanism.

**File:** `src/services/client.ts` - All fetch calls lack retry logic.

**Impact:** Temporary network blips cause permanent operation failures.

**Recommendations:**
- Add exponential backoff retry for 5xx errors and network failures
- Make retry policy configurable

---

### 16. Type Safety Gaps

**Issue:** Several locations use type assertions or `any` types that reduce type safety.

**Examples:**

```typescript
// src/index.ts:84-87 - Complex type gymnastics with unknown
const client = ctx.client as unknown as {
  provider?: { list?: () => Promise<{ data?: unknown[] } | unknown[]> };
  providers?: { list?: () => Promise<{ data?: unknown[] } | unknown[]> };
};

// src/services/client.ts:312-316 - Multiple type assertions
const data = await response.json() as
  | Mem0AddResponse
  | Mem0AddResponseItem
  | Mem0AddResponseItem[];

// src/services/compaction.ts:455 - Type assertion for API response
const messages = (resp.data ?? resp) as Array<{ info: MessageInfo; parts?: Array<{ type: string; text?: string }> }>;
```

**Recommendations:**
- Define proper runtime validation with Zod (already a dependency)
- Use branded types for validated data
- Minimize type assertions

---

## Key Decisions and Rationale

### Current Architecture Choices

1. **Result Type Pattern (vs Exceptions):** The client uses `{ success: boolean, error?: string }` result types rather than throwing. This is a valid pattern for error visibility but requires discipline to check results.

2. **File-Based Message Injection:** Compaction injects messages by writing to OpenCode's internal storage. This is tightly coupled to OpenCode implementation details and may break with updates.

3. **Module-Level Configuration:** Config is loaded at module initialization time, making it effectively global. This simplifies access but makes testing and reconfiguration difficult.

### Security Considerations

1. **API Key Storage:** API keys are read from environment or config file, no encryption. This is standard practice but should be documented.

2. **Log File Exposure:** Log file at `~/.opencode-supermemory.log` may contain memory content and user queries. No rotation or cleanup.

3. **No Input Sanitization:** Memory content is sent to Mem0 API without additional sanitization beyond privacy tag stripping.

---

## Maintenance Notes

### Known Hotspots

1. **`src/services/compaction.ts:324-439`** - Complex async state machine for compaction, most likely area for bugs
2. **`src/services/client.ts:126-171`** - Redirect handling with security checks, sensitive code path
3. **`src/cli.ts:206-267`** - Config file modification, risk of corrupting user config

### Dependencies

- **@opencode-ai/plugin ^1.0.162** - Core dependency, breaking changes would be catastrophic
  - Currently resolves to 1.0.191 (lockfile)
- **No runtime dependencies beyond OpenCode plugin** - Minimal supply chain risk
- **Bun for runtime** - Fast but less ecosystem support than Node.js

---

## Prioritized Recommendations

### High Priority (Fix Soon)

1. **Add test suite** - Start with client.ts and config.ts
2. **Fix memory leak** - Clean up `injectedSessions` on session deletion
3. **Add retry logic** - For all Mem0 API calls
4. **Fix race conditions** - In compaction state management
5. **Rename log file** - From supermemory to mem0

### Medium Priority (Fix Eventually)

6. **Add input validation** - For all tool arguments and config values
7. **Remove singleton pattern** - Inject dependencies explicitly
8. **Add atomic file writes** - For CLI config modifications
9. **Make timeouts configurable**
10. **Improve error propagation** - Ensure users see critical errors

### Low Priority (Nice to Have)

11. **Support more models** - For context compaction
12. **Improve user ID generation** - More reliable cross-machine identity
13. **Use Zod for validation** - Leverage existing dependency
14. **Consolidate API versions** - Use v2 endpoints consistently
15. **Add rate limiting** - For memory operations

---

*Concerns audit: 2026-04-01*
