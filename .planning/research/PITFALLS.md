# Domain Pitfalls: Memory Persistence Integrations

**Domain:** External memory API integrations (Mem0 Platform API via OpenCode plugin)
**Researched:** 2025-04-01
**Confidence:** HIGH (based on real Mem0 GitHub issues, production patterns, and codebase analysis)

---

## Executive Summary

Memory persistence integrations commonly fail in ways that create a dangerous illusion of success. Users issue "remember" commands, see no errors, and assume data is saved—when in reality, failures are silent and widespread. This document catalogs the specific failure modes affecting opencode-mem0 based on research of Mem0's own issues, similar integrations, and analysis of the current codebase.

**Critical insight:** The most dangerous failures are those that return HTTP 200 with no data stored. Users cannot distinguish between "saved successfully" and "failed silently."

---

## Critical Pitfalls

Mistakes that cause data loss and user trust erosion.

### Pitfall 1: Silent Fact Extraction Failures

**What goes wrong:**
Memory appears to save (API returns 200), but the underlying fact extraction returns empty results (`{"facts": []}`). No embeddings are generated, no memory is actually stored.

**Why it happens:**
1. LLM-based fact extraction is too conservative—valid user input is classified as "not worth remembering"
2. Non-OpenAI LLMs (Qwen3, Ollama, etc.) produce malformed JSON that fails parsing
3. No retry logic on transient LLM failures
4. Invalid UTF-16 surrogate pairs in LLM output cause `JSONDecodeError` even with `strict=False`

**Real-world evidence:**
- Mem0 Issue #3009: "3 out of 5 memory creations lost - Fact extraction inconsistently returns empty results"
- Mem0 Issue #4540: "Fact extraction silently fails on malformed JSON from non-OpenAI LLMs"
- User reports: "The same input succeeds on retry, but there's no retry mechanism"

**Consequences:**
- ~80% failure rate with null responses (from #3009)
- Silent data loss—user thinks memory is saved
- Valid user input discarded without feedback
- Cross-session continuity broken

**Prevention:**
1. **Validate extraction results:** Check if facts array is empty and surface to user
2. **Implement retry logic:** Retry up to 3 times on parsing failures (transient issues usually resolve)
3. **Clean invalid JSON:** Strip UTF-16 surrogate pairs before parsing
4. **More inclusive prompts:** "When in doubt, include the information rather than excluding it"
5. **Explicit error on empty extraction:** Return clear error instead of silent success

**Detection:**
- Log warnings when fact extraction returns empty arrays
- Monitor ratio of successful saves to extraction attempts
- Alert when extraction failure rate exceeds threshold

**Phase to address:** Phase 2 (Memory-write reliability)

---

### Pitfall 2: Async Fire-and-Forget Without Confirmation

**What goes wrong:**
Memory operations are async but treated as fire-and-forget. The user sees immediate acknowledgment, but the actual persistence happens later (or fails later) with no feedback.

**Why it happens:**
1. Plugin architecture uses async handlers for `chat.message` events
2. No await on critical path means failures don't block response
3. Users assume "no error = success" but async errors surface after user interaction ends
4. No synchronous confirmation before acknowledging to user

**Current codebase analysis:**
In `src/index.ts` lines 184-195:
```typescript
if (detectMemoryKeyword(userMessage)) {
  log("chat.message: memory keyword detected");
  const nudgePart: Part = {
    // ... creates synthetic message
  };
  output.parts.push(nudgePart);  // Nudge added immediately
}
```

The nudge is pushed immediately, but the actual `mem0` tool call happens asynchronously. If the tool call fails, the user has already been told to use the tool.

**Consequences:**
- User believes memory is captured ("I said remember this")
- Agent nudges user to use tool ("Memory trigger detected")
- Tool call fails silently
- User never learns the memory wasn't saved

**Prevention:**
1. **Synchronous validation:** Before nudging, verify API is reachable
2. **Tool-level confirmation:** Tool should return explicit confirmation/error, not just fire-and-forget
3. **Post-save feedback:** After tool execution, show "Memory saved: {content}" or "Failed to save: {reason}"
4. **Queue with retry:** Failed writes go to retry queue with exponential backoff

**Detection:**
- Log every addMemory call with result (success/failure)
- Monitor tool execution success rate
- Track gap between "memory trigger detected" and "memory saved" events

**Phase to address:** Phase 2 (Error visibility)

---

### Pitfall 3: Scope Routing Misconfiguration

**What goes wrong:**
Memories are saved to one scope (user vs project) but searched in another. User expects cross-project memory but it's project-scoped, or vice versa.

**Why it happens:**
1. Default scope behavior is implicit and varies by operation
2. User doesn't understand scope semantics ("user" = cross-project, "project" = current directory)
3. `org_id` and `project_id` workspace routing is optional but affects behavior when set
4. Scope ID construction uses hashed identifiers that are hard to debug

**Current codebase analysis:**
In `src/services/client.ts` lines 194-199:
```typescript
private getScopeUserId(scope: MemoryScopeContext): string {
  if (scope.projectId) {
    return `${CONFIG.scopePrefix}:${scope.userId}:${scope.projectId}`;
  }
  return `${CONFIG.scopePrefix}:${scope.userId}`;
}
```

If `projectId` is accidentally set or unset, memories route to completely different namespaces.

**Consequences:**
- User: "I said to remember this for all projects" → saved to project scope
- Next session in different project → memory not found
- User: "Why doesn't it remember?"
- Data appears lost but is just misrouted

**Prevention:**
1. **Explicit scope in feedback:** "Saved to PROJECT scope: {content}" vs "Saved to USER scope: {content}"
2. **Scope validation:** If user mentions "all projects" or "always," suggest user scope
3. **Debug mode:** Show scope ID in logs so misrouting is visible
4. **Search both scopes by default:** `mem0 search` without scope should query both

**Detection:**
- Log scope ID for every write and read
- Compare search query scopes to write scopes
- Alert on scope with no recent writes but active searches

**Phase to address:** Phase 2 (User expectation alignment)

---

### Pitfall 4: The "Nudge But No Follow-Through" Pattern

**What goes wrong:**
Plugin detects memory keyword and nudges the agent to call `mem0 add`, but the agent doesn't actually call the tool. User thinks memory is saved.

**Why it happens:**
1. Memory nudge is just a synthetic message—no enforcement
2. Agent may ignore the nudge or respond conversationally instead of using the tool
3. User sees nudge appear and assumes the system handled it
4. No verification that the tool was actually invoked

**Current codebase analysis:**
In `src/index.ts` lines 21-29:
```typescript
const MEMORY_NUDGE_MESSAGE = `[MEMORY TRIGGER DETECTED]
The user wants you to remember something. You MUST use the \`mem0\` tool with \`mode: "add"\` to save this information.
...
DO NOT skip this step. The user explicitly asked you to remember.`;
```

The nudge says "You MUST use" but there's no mechanism to ensure compliance.

**Consequences:**
- User: "I said 'remember this' and saw the [MEMORY TRIGGER DETECTED] message"
- Memory was never actually saved
- User assumes the plugin is broken when it's actually the agent not following instructions

**Prevention:**
1. **Auto-save option:** Bypass agent for simple "remember X" patterns—extract and save directly
2. **Tool call verification:** Track nudge → tool call correlation
3. **Fallback prompt:** If agent responds without tool call, inject follow-up: "You have not yet saved this memory. Use the mem0 tool."
4. **User-visible confirmation:** Show "Saving memory..." → "Memory saved" or "Agent did not save—retry?"

**Detection:**
- Log nudge injection timestamp
- Log tool call timestamps
- Alert on nudge without subsequent tool call within N messages

**Phase to address:** Phase 2 (Memory-write reliability)

---

### Pitfall 5: Timeout and Network Failures Without Retry

**What goes wrong:**
API calls timeout or fail due to transient network issues. No retry means permanent data loss for that operation.

**Why it happens:**
1. 30-second timeout may be insufficient for LLM-based extraction
2. Network hiccups, API rate limiting, or Mem0 service issues
3. Current implementation has single-attempt with catch-all error handler
4. No exponential backoff or circuit breaker

**Current codebase analysis:**
In `src/services/client.ts` lines 67-74:
```typescript
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_, reject) =>
      setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
    ),
  ]);
}
```

Single attempt, no retry. Lines 275-336 show `addMemory` with try/catch but no retry loop.

**Consequences:**
- Transient failures become permanent data loss
- "It worked yesterday but not today" syndrome
- User loses trust in reliability

**Prevention:**
1. **Retry with backoff:** 3 attempts with exponential backoff (1s, 2s, 4s)
2. **Circuit breaker:** After N failures, fast-fail with clear error
3. **Offline queue:** Queue failed writes for retry when connection restored
4. **Timeout tuning:** 30s may be too short for complex extractions

**Detection:**
- Log retry attempts and outcomes
- Monitor timeout rate vs failure rate
- Track queue depth for offline writes

**Phase to address:** Phase 2 (Memory-write reliability)

---

### Pitfall 6: Compaction Saves Fail Silently

**What goes wrong:**
Context compaction triggers summary generation, but saving that summary as a memory fails silently. Session context is lost.

**Why it happens:**
1. Compaction is async and runs in background
2. `saveSummaryAsMemory` (lines 301-321 in compaction.ts) logs failure but user never sees it
3. Summary generation succeeds but persistence fails
4. User expects session continuity but summary was never stored

**Current codebase analysis:**
```typescript
async function saveSummaryAsMemory(sessionID: string, summaryContent: string): Promise<void> {
  // ...
  if (result.success) {
    log("[compaction] summary saved as memory", { sessionID, memoryId: result.id });
  } else {
    log("[compaction] failed to save summary", { error: result.error });  // Silent!
  }
}
```

**Consequences:**
- User: "I had a long session and it compacted, but now it doesn't remember anything"
- Summary was generated but never persisted
- Session continuity broken despite compaction appearing to work

**Prevention:**
1. **Retry compaction saves:** Treat summary saves as critical—retry with backoff
2. **User notification:** If compaction save fails, show warning: "Session summary could not be saved"
3. **Fallback storage:** Save summary to local file if Mem0 API fails
4. **Compaction verification:** After compaction, verify summary exists in Mem0

**Detection:**
- Track compaction events vs successful summary saves
- Alert on compaction save failure rate

**Phase to address:** Phase 3 (Compaction reliability)

---

## Moderate Pitfalls

### Pitfall 7: Response Validation Gaps

**What goes wrong:**
API returns 200 but with unexpected payload structure. Code assumes success but data is malformed or missing.

**Why it happens:**
1. Mem0 API may return different structures for different versions/scenarios
2. Response parsing doesn't validate all required fields
3. Type assertions (`as Mem0AddResponse`) bypass runtime validation
4. Nested response structures change between API versions

**Current codebase analysis:**
In `src/services/client.ts` lines 312-336:
```typescript
const data = await response.json() as
  | Mem0AddResponse
  | Mem0AddResponseItem
  | Mem0AddResponseItem[];
// ... complex branching to extract items
const firstItem = items[0];
const id = firstItem?.id ?? firstItem?.memory_id;
```

Multiple possible shapes, no runtime schema validation.

**Prevention:**
1. **Schema validation:** Use Zod or similar to validate API responses
2. **Fail on missing fields:** If expected fields are missing, treat as error
3. **Version pinning:** Pin to specific Mem0 API version to avoid surprise changes
4. **Graceful degradation:** If response is malformed, retry or fail explicitly

**Phase to address:** Phase 2 (Error visibility)

---

### Pitfall 8: Sector/Type Mismatch Confusion

**What goes wrong:**
User specifies `sector: "procedural"` but code uses first tag as sector. Or type is set but not validated against allowed values.

**Why it happens:**
1. Sector extracted from `options?.tags?.[0]` (line 330 in client.ts)
2. Type and sector are separate concepts but conflated in some paths
3. No validation that sector is one of allowed HSG values
4. User may specify both sector and type, but behavior is unclear

**Prevention:**
1. **Explicit sector parameter:** Don't derive sector from tags
2. **Validation:** Reject invalid sector/type values with clear error
3. **Documentation:** Clear distinction between sector (HSG category) and type (memory classification)

**Phase to address:** Phase 1 (Input validation)

---

## Minor Pitfalls

### Pitfall 9: Logging Overload

**What goes wrong:**
Excessive debug logging in production makes it hard to spot actual issues.

**Prevention:**
1. Use log levels (debug/info/warn/error)
2. Reduce verbosity in production
3. Structured logging for easier filtering

**Phase to address:** Phase 4 (Observability)

---

### Pitfall 10: Privacy Filter False Positives

**What goes wrong:**
`stripPrivateContent` may over-filter, removing legitimate content that happens to contain `<private>`-like patterns.

**Prevention:**
1. Document exact filtering behavior
2. Allow users to review what was filtered
3. Option to bypass filtering for specific memories

**Phase to address:** Phase 1 (Privacy features)

---

## Phase-Specific Warnings

| Phase | Topic | Likely Pitfall | Mitigation |
|-------|-------|----------------|------------|
| Phase 2 | Memory-write reliability | Silent fact extraction failures | Validate extraction results, retry on empty |
| Phase 2 | Error visibility | Async failures not surfaced | Synchronous confirmation, post-save feedback |
| Phase 2 | User expectations | Scope misrouting | Explicit scope in feedback, search both scopes |
| Phase 2 | Agent compliance | Nudge but no tool call | Auto-save simple patterns, track nudge→call gap |
| Phase 3 | Compaction | Summary save failures | Retry critical saves, user notification on failure |
| Phase 4 | Observability | Can't diagnose failures | Structured logging, metrics, alerts |

---

## Sources

### Mem0 GitHub Issues (Primary)
- **Issue #3009:** "3 out of 5 memory creations lost - Fact extraction inconsistently returns empty results" — Confirms ~80% failure rate due to conservative extraction
- **Issue #4540:** "Fact extraction silently fails on malformed JSON from non-OpenAI LLMs" — Documents JSON parsing failures with Qwen3, no retry logic
- **Issue #4549:** "autoRecall silent failure — `before_agent_start` hook does not fire" — Scope/hook timing issues
- **Issue #4473:** "Qdrant Local Storage Loses Data on Restart" — Persistence layer failures
- **Issue #4279:** "Vector store defaults to /tmp/{provider}" — Data loss due to misconfiguration

### Production Patterns
- Medium: "Persistent Memory for AI Coding Agents: An Engineering Blueprint for Cross-Session Continuity" (Feb 2026) — Industry best practices for memory systems
- Medium: "Fire-and-Forget in ASP.NET Core: Why async/await Is Not Enough" — Async pattern failures
- Various API resilience articles (2024-2026) — Retry, circuit breaker, timeout patterns

### Codebase Analysis
- `/src/services/client.ts` — Mem0 API client implementation
- `/src/index.ts` — Memory trigger detection and tool registration
- `/src/services/compaction.ts` — Context compaction and summary persistence

---

## Recommendations Summary

**Immediate (Phase 2):**
1. Add retry logic to all write operations (3 attempts, exponential backoff)
2. Validate fact extraction results—error on empty arrays
3. Show explicit confirmation: "Memory saved to {scope} scope" or "Failed: {reason}"
4. Search both user and project scopes by default
5. Track nudge→tool call correlation and warn on gaps

**Short-term (Phase 3):**
1. Implement auto-save for simple "remember X" patterns
2. Add circuit breaker for Mem0 API failures
3. Retry compaction saves with user notification on persistent failure
4. Add schema validation for API responses

**Long-term (Phase 4):**
1. Structured metrics and alerting
2. Offline write queue for resilience
3. User-accessible memory log ("Show me what you've remembered")
4. Confidence scoring for memories
