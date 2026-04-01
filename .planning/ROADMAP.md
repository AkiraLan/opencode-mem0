# ROADMAP.md: opencode-mem0

**Version:** v1 Reliability Milestone  
**Granularity:** Coarse (2 phases)  
**Mode:** YOLO (auto-approve)  
**Defined:** 2025-04-01

---

## Overview

**Goal:** Fix the root cause of "remember xxx doesn't persist" reports by addressing silent failures and adding observability.

**Core Value:** Memories are reliably captured and retrieved to make every conversation feel continuous and contextually aware.

---

## Phases

- [ ] **Phase 1: Critical Reliability Fixes** - Address silent extraction failures, add retry logic, ensure explicit user feedback
- [ ] **Phase 2: Observability Foundation** - Structured logging, operation tracking, and debug visibility

---

## Phase Details

### Phase 1: Critical Reliability Fixes

**Goal:** Users can trust that "remember this" commands actually persist memories, with clear confirmation or failure feedback.

**Depends on:** Nothing (first phase)

**Requirements:** REL-01, REL-02, REL-03, REL-04, REL-05, SCOPE-01, SCOPE-02, SCOPE-03

**Success Criteria** (what must be TRUE):

1. When user says "remember X", they see either "✓ Saved to {scope} scope" or "✗ Failed: {reason}" within 5 seconds
2. Failed writes automatically retry up to 3 times with exponential backoff before showing failure
3. Empty fact extraction results (`{"facts": []}`) trigger a retry with reformatted content
4. Memory search queries both user and project scopes when scope is unspecified
5. Memory keyword detection verifies the mem0 tool was actually called and warns if not
6. All successful saves show: memory ID, scope, and extraction summary
7. All failures show: error type, actionable guidance, and retry status

**Plans**: TBD

---

### Phase 2: Observability Foundation

**Goal:** Developers and users can diagnose memory issues through structured logs and a debug command.

**Depends on:** Phase 1

**Requirements:** OBS-01, OBS-02, OBS-03, OBS-04, OBS-05

**Success Criteria** (what must be TRUE):

1. All memory operations emit structured JSON logs (Pino) with correlation IDs
2. Running `/mem0-debug` shows the last 50 memory operations with timing and status
3. Each operation has a unique correlation ID for end-to-end tracing
4. Error responses include actionable guidance (e.g., "Check API key", "Retry in 30s")
5. Logs include: operation type, scope, duration, success/failure, error details

**Plans**: TBD

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Critical Reliability Fixes | 0/8 | Not started | - |
| 2. Observability Foundation | 0/5 | Not started | - |

---

## Coverage

**v1 Requirements: 13 total**

| Requirement | Phase | Status |
|-------------|-------|--------|
| REL-01 | Phase 1 | Pending |
| REL-02 | Phase 1 | Pending |
| REL-03 | Phase 1 | Pending |
| REL-04 | Phase 1 | Pending |
| REL-05 | Phase 1 | Pending |
| OBS-01 | Phase 2 | Pending |
| OBS-02 | Phase 2 | Pending |
| OBS-03 | Phase 2 | Pending |
| OBS-04 | Phase 2 | Pending |
| OBS-05 | Phase 2 | Pending |
| SCOPE-01 | Phase 1 | Pending |
| SCOPE-02 | Phase 1 | Pending |
| SCOPE-03 | Phase 1 | Pending |

✓ **Coverage: 13/13 requirements mapped (100%)**
✓ **No orphaned requirements**

---

## v2 Backlog

Requirements deferred to future milestone (resilience and advanced observability):

- RES-01: Circuit breaker (5 failures → 30s cooldown)
- RES-02: Health check endpoint
- RES-03: Compaction save retries with user notification
- RES-04: Pre-flight configuration validation
- ADV-01: OpenTelemetry tracing
- ADV-02: Metrics collection
- ADV-03: Background verification queue
- ADV-04: Memory preview before save

---

## Notes

### Why Only 2 Phases?

With **coarse granularity** and YOLO mode, we compress the research-suggested 4 phases into 2 focused deliverables:

1. **Phase 1 combines:** Critical reliability + scope/feedback requirements
   - All "must have" fixes for the reported issue
   - Everything needed to make "remember this" trustworthy

2. **Phase 2 combines:** Basic observability
   - Logging and debugging capabilities
   - Enables verification that Phase 1 works

**v2 scope** (circuit breaker, tracing, metrics) is explicitly out of v1 to maintain focus on fixing the core reliability issue.

### Phase Dependencies

```
Phase 1 (Reliability)
    ↓
Phase 2 (Observability)
    ↓
v2: Resilience + Advanced Observability
```

Phase 2 depends on Phase 1 because:
- Observability is more useful when operations are reliable
- Debug output needs meaningful status to display
- Correlation IDs matter more when retries are working

---

*Last updated: 2025-04-01*
