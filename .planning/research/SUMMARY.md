# Research Summary: opencode-mem0 Debugging & Pitfalls

**Domain:** OpenCode plugin for persistent memory via Mem0 Platform API  
**Research Type:** Stack + Pitfalls dimensions for memory persistence debugging and reliability  
**Researched:** 2025-04-01  
**Overall Confidence:** HIGH

---

## Executive Summary

The opencode-mem0 plugin suffers from two fundamental problems: **silent failures in async operations** and **silent fact extraction failures** from the Mem0 API. When users say "remember this," they assume success—but failures are widespread, invisible, and based on real Mem0 production issues.

### Critical Discovery: The "Silent Extraction" Problem

Research of Mem0 GitHub issues reveals a devastating failure mode: the API returns HTTP 200, but underlying LLM-based fact extraction returns empty results (`{"facts": []}`). This happens ~60-80% of the time with certain inputs (Issue #3009). No retry, no user feedback. Users think memories are saved; they're not.

### Root Causes Identified:

1. **Silent Fact Extraction Failures**: Mem0 API returns 200 with empty facts; no validation or retry
2. **Async Error Handling Gaps**: Errors caught and returned as JSON, but LLMs may not surface them
3. **No Retry Logic**: Single transient failures result in permanent data loss
4. **Scope Routing Confusion**: Memories saved to one scope, searched in another
5. **Nudge Without Enforcement**: Memory keyword triggers nudge, but agent may not call tool
6. **Insufficient Observability**: Current `log()` writes to internal stream with no user visibility

### Code Flow (Problematic):
```
User says "remember X" 
  → Keyword detected → nudge injected
  → Agent may/may not call mem0 tool
  → API call made
  → Extraction returns empty facts (silent!)
  → OR: API fails, error caught
  → User sees nothing
```

---

## Key Findings

**Stack:** TypeScript/Bun OpenCode plugin → Mem0 Platform API (REST)

**Architecture:** 
- Async plugin hooks with fire-and-forget patterns
- Tool-based memory capture with no call verification
- Dual scope (user/project) with implicit defaults
- Compaction only for Claude models

**Critical Pitfalls (from Mem0 Issues #3009, #4540):**
- ~80% failure rate due to conservative fact extraction
- JSON parsing failures with non-OpenAI LLMs (Qwen3, Ollama)
- No retry on transient failures
- UTF-16 surrogate pair errors

**Secondary Pitfalls:**
- Scope misrouting (user vs project)
- Compaction saves fail silently
- Timeout without retry (30s single attempt)
- Response validation gaps

---

## Implications for Roadmap

Based on research (stack + pitfalls), suggested phase structure:

### Phase 1: "Critical Reliability Fixes" (Week 1-2)
**Addresses:** Silent extraction failures, retry logic, explicit feedback  
**Components:**
- Add retry logic (3x exponential backoff) for all write operations
- Validate fact extraction results—error on empty arrays
- Show explicit confirmation: "Saved to {scope} scope" or "Failed: {reason}"
- Search both user and project scopes by default

**Why First:** These fix the root cause of "remember commands don't persist" reports.

### Phase 2: "Observability Foundation" (Week 2-3)
**Addresses:** Silent async failures, debugging gap  
**Components:**
- Replace `log()` with Pino structured logging
- Add in-memory operation log (last 50 operations)
- Add `/mem0-debug` slash command to view recent operations
- Track nudge→tool call correlation

**Why Second:** Visibility helps diagnose remaining issues and verify Phase 1 fixes.

### Phase 3: "Resilience Layer" (Week 4-5)
**Addresses:** Transient errors, circuit breaker, compaction reliability  
**Components:**
- Add circuit breaker (5 failures → 30s cooldown)
- Retry compaction saves with user notification on failure
- Add health check to mem0 tool (`mode=health`)
- Pre-flight validation on plugin startup

**Why Third:** Advanced resilience builds on foundational reliability.

### Phase 4: "Advanced Observability" (Week 6+)
**Addresses:** Production monitoring, async tracing  
**Components:**
- OpenTelemetry tracing for operation spans
- Metrics collection (success rate, latency percentiles)
- Optional: Dashboard for memory operation health

**Why Fourth:** Nice-to-have for production; core functionality works without it.

---

## Phase Ordering Rationale

1. **Reliability → Observability → Resilience → Monitoring**
   - Fix the silent extraction failures first (root cause)
   - Add visibility to verify fixes
   - Add advanced resilience for edge cases
   - Monitoring is last (needs reliability to be meaningful)

2. **Critical → Important → Nice-to-Have**
   - Phase 1 fixes user-reported "remember doesn't work" issue
   - Phase 2 enables debugging
   - Phase 3+ are optimizations

3. **Synchronous → Asynchronous**
   - Retry logic is transparent
   - Operation log is immediately available
   - Tracing is background/async

---

## Research Flags for Phases

| Phase | Likely Needs Deeper Research | Reason |
|-------|------------------------------|--------|
| Phase 1 | MAYBE | Retry logic patterns for Mem0 API specifically |
| Phase 2 | NO | Pino integration is straightforward; Bun-compatible |
| Phase 3 | MAYBE | Circuit breaker state in Bun plugin context; Claude-only compaction behavior |
| Phase 4 | YES | OpenTelemetry overhead in Bun; verify compatibility |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Pitfalls | HIGH | Based on real Mem0 GitHub issues (#3009, #4540, #4549) with reproduction steps |
| Stack Recommendations | HIGH | Bun docs, Pino docs, and GitHub sources verified |
| Mem0 API Behavior | HIGH | Derived from production issues and codebase analysis |
| Phase Structure | MEDIUM-HIGH | Based on verified issues plus standard reliability engineering |
| Bun-Specific Features | HIGH | Official docs and merged PRs verified |
| User Impact | HIGH | Silent failures are well-documented; 80% failure rate in some scenarios |

---

## Gaps to Address

1. **Mem0 API Error Codes**: Limited official docs on rate limits and specific error responses
2. **Non-Claude Model Behavior**: Current compaction is Claude-only; how do other models behave?
3. **Bun Plugin Context**: Unclear how Bun plugins handle persistent state (circuit breaker needs state)
4. **OpenCode Tool Result Handling**: Need to verify how LLMs consume tool result JSON—do they surface errors?
5. **Real User Scenarios**: What specific phrases trigger extraction failures most often?

---

## Immediate Action Items

1. **Today**: Review PITFALLS.md for top 3 critical issues to fix
2. **This Week**: Implement retry logic for addMemory with extraction validation
3. **Next Week**: Add explicit scope feedback in tool responses
4. **Verify**: Test fact extraction with various input types to calibrate failure rate

---

## Files Created

| File | Purpose |
|------|---------|
| `PITFALLS.md` | Domain-specific pitfalls for memory persistence (10 pitfalls with prevention) |
| `STACK.md` | Technology recommendations with installation and patterns |
| `FEATURES.md` | Feature landscape for debugging/observability |
| `ARCHITECTURE.md` | System patterns for resilient async operations |
